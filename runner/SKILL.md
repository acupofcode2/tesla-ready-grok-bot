---
name: tesla-ready-runner
description: 'use this when Tesla Ready sets up or repairs its timer: exact runner.sh and helper scripts to write verbatim'
---
# Tesla Ready runner

Write these four files verbatim (tested in real use; don't edit or author your own). Only the folder /home/box/agent-data/projects/tesla-ready/ is used.

Needs: python3, curl, jq, flock, git, go, openssl, procps (pgrep), and the proxy at ~/go/bin/tesla-http-proxy (if missing: `mkdir -p ~/go/bin && git clone --depth 1 -b v0.4.1 https://github.com/teslamotors/vehicle-command /tmp/vc && cd /tmp/vc && go build -o ~/go/bin/ ./cmd/tesla-http-proxy ./cmd/tesla-control`; `go install @latest` fails). In .secrets/ (chmod 600): client_id, client_secret (saved from TESLA_CLIENT_SECRET; only for the partner-token and code-exchange calls you make by hand; send it as `--data-urlencode client_secret@.secrets/client_secret`, never on a command line), tokens.json (the full token response), tokens_obtained_at (epoch seconds, written with tokens.json), private-key.pem, and a localhost TLS pair: `openssl req -x509 -nodes -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -keyout proxy-tls-key.pem -out proxy-tls-cert.pem -days 3650 -subj /CN=localhost -addext subjectAltName=DNS:localhost,IP:127.0.0.1`.

Optional plain files in the folder: fleet_api_url (one line, the region's Fleet API base URL; NA if missing) and direct_cmd_vins (one VIN per line for cars whose fleet_status (`tesla.sh fleet VIN`) says vehicle_command_protocol_required is false, e.g. pre-2021 Model S/X; their commands skip the proxy).

schedule.json is a JSON list of entries: {"id" (calendar event instance id, or od-<time> for on-demand), "event", "climate_start" (ISO with the UTC offset valid on that date), "leave", "temp" (°F), "seat" (auto|heat|cool|none), "seats" (driver|both), "heat_f" (default 40), "cool_f" (default 85), "vin", "status" (pending|done|partial|failed|skipped|cancelled; partial = climate on but a seat command failed), "result", "reported"}. Edit it only under the lock (`flock schedule.json.lock`) with an atomic replace. Learned seat rules go in seat/heat_f/cool_f. The runner sets status, result, and reported=false; you set reported=true after telling them.

/home/box/agent-data/projects/tesla-ready/runner.sh (chmod 755)
```python
#!/usr/bin/env python3
# Tesla Ready timer: every 30s read schedule.json; ~60s before a pending entry's climate_start run the warm-up; record status/result; log to runner.log. Never logs secrets.
import argparse, datetime as dt, fcntl, json, os, re, socket, subprocess, sys, tempfile, time
DIR = "/home/box/agent-data/projects/tesla-ready"
TESLA = os.path.join(DIR, "tesla.sh")
START_PROXY = os.path.join(DIR, "start-proxy.sh")
TOKENS = os.path.join(DIR, ".secrets", "tokens.json")
TOKENS_AT = os.path.join(DIR, ".secrets", "tokens_obtained_at")
TICK, LEAD, STALE, WAKE_GAP, REFRESH_MARGIN = 30, 60, 600, 1200, 1800
DRIVING = {"D", "R", "N"}
ARGS = None
P = {}

def now():
  return dt.datetime.now().astimezone()

def scrub(s, n=220):
  s = re.sub(r"eyJ[\w-]+\.[\w-]+\.[\w-]*", "<jwt>", str(s))
  s = re.sub(r"(?i)bearer\s+\S+", "Bearer <redacted>", s)
  s = re.sub(r"(?i)(access_token|refresh_token|client_secret|id_token)\"?\s*[:=]\s*\"?[^\s\",}]+", r"\1=<redacted>", s)
  return " ".join(s.split())[:n]

def log(msg):
  with open(P["log"], "a") as f:
    f.write(f"{now().isoformat(timespec='seconds')} {scrub(msg, 600)}\n")

class Locked:
  def __init__(self, path):
    self.path = path
  def __enter__(self):
    self.fd = open(self.path, "a+")
    fcntl.flock(self.fd, fcntl.LOCK_EX)
    return self
  def __exit__(self, *a):
    fcntl.flock(self.fd, fcntl.LOCK_UN)
    self.fd.close()

def read_json(path, default):
  try:
    with open(path) as f:
      return json.load(f)
  except FileNotFoundError:
    return default

def write_json_atomic(path, data):
  fd, tmp = tempfile.mkstemp(prefix=".tmp-", dir=os.path.dirname(path) or ".")
  try:
    with os.fdopen(fd, "w") as f:
      json.dump(data, f, indent=2)
      f.write("\n")
      f.flush()
      os.fsync(f.fileno())
    try:
      os.chmod(tmp, os.stat(path).st_mode & 0o777)
    except FileNotFoundError:
      os.chmod(tmp, 0o644)
    os.replace(tmp, path)
  except BaseException:
    try:
      os.unlink(tmp)
    except OSError:
      pass
    raise

def entries_of(data):
  return data.get("entries", []) if isinstance(data, dict) else data

def load_schedule():
  with Locked(P["schedule_lock"]):
    return read_json(P["schedule"], [])

def update_entries(updates):
  with Locked(P["schedule_lock"]):
    data = read_json(P["schedule"], [])
    for e in entries_of(data):
      if e.get("id") in updates and e.get("status") != "cancelled":
        e.update(updates[e["id"]])
    write_json_atomic(P["schedule"], data)

def load_state():
  return read_json(P["state"], {})

def save_state(st):
  write_json_atomic(P["state"], st)

def is_runner(pid):
  try:
    with open(f"/proc/{pid}/cmdline", "rb") as f:
      return b"runner.sh" in f.read()
  except OSError:
    return False

def claim_pid():
  try:
    old = int(open(P["pid"]).read().strip())
  except (OSError, ValueError):
    old = None
  if old and old != os.getpid() and is_runner(old):
    print(f"runner already running (pid {old}); exiting", file=sys.stderr)
    sys.exit(0)
  with open(P["pid"], "w") as f:
    f.write(f"{os.getpid()}\n")

class RateLimited(Exception):
  pass

def sh(args, timeout=90):
  try:
    r = subprocess.run(args, capture_output=True, text=True, timeout=timeout, stdin=subprocess.DEVNULL, cwd=DIR)
  except subprocess.TimeoutExpired:
    return 124, "", "timeout"
  out, err = r.stdout, r.stderr
  blob = err
  try:
    j = json.loads(out)
    if not (isinstance(j, dict) and j.get("response") is not None):
      blob += " " + out
  except ValueError:
    blob += " " + out
  if re.search(r"\b429\b|too many requests|rate.?limit", blob, re.I) and "wake rate limit" not in blob:
    raise RateLimited(scrub(blob, 120))
  return r.returncode, out, err

def tjson(args, timeout=90):
  rc, out, err = sh([TESLA] + args, timeout)
  try:
    return rc, json.loads(out), err
  except ValueError:
    return rc, None, (out + " " + err)

def token_expires_in():
  try:
    obtained = int(open(TOKENS_AT).read().strip())
    with open(TOKENS) as f:
      return obtained + int(json.load(f).get("expires_in", 0)) - time.time()
  except Exception:
    return -1

def proxy_up():
  try:
    with socket.create_connection(("127.0.0.1", 4443), timeout=2):
      return True
  except OSError:
    return False

def cmd(vin, name, body=None):
  rc, j, err = tjson(["cmd", vin, name] + ([json.dumps(body)] if body is not None else []), timeout=60)
  if isinstance(j, dict):
    resp = j.get("response")
    if isinstance(resp, dict) and resp.get("result") is True:
      return True, "ok"
    reason = resp.get("reason") if isinstance(resp, dict) else None
    return False, scrub(reason or j.get("error") or j.get("error_description") or json.dumps(j), 120)
  return False, scrub(err or f"rc={rc}", 120)

def key_error(msg):
  return bool(re.search(r"public key|not paired|key.*(missing|unknown|not.*(found|recognized))|unknown_key", msg, re.I))

def f_to_c(f):
  return round((float(f) - 32) * 5 / 9, 1)

def seat_plan(entry, outside_f, has_cool):
  seat = (entry.get("seat") or "auto").lower()
  heat_f, cool_f = float(entry.get("heat_f", 40)), float(entry.get("cool_f", 85))
  known = outside_f is not None
  want = None
  if seat == "heat" or (seat == "auto" and known and outside_f <= heat_f):
    want = "heat"
  elif seat == "cool" or (seat == "auto" and known and outside_f >= cool_f):
    want = "cool" if has_cool else "cool-unsupported"
  if want == "heat":
    return want, 3 if (known and outside_f < 20) else 2
  if want == "cool":
    return want, 3 if (known and outside_f > 95) else 2
  return want, 0

def asleep(j, err):
  blob = (json.dumps(j) if j is not None else "") + " " + (err or "")
  return bool(re.search(r"vehicle unavailable|asleep|offline|408", blob, re.I))

def execute(entry, st):
  vin = entry.get("vin")
  if not vin:
    return "failed", "no vin on entry"
  temp_f = float(entry.get("temp") or 69)
  if not 50 <= temp_f <= 90:
    return "failed", f"temp {temp_f} is outside 50-90 (schedule.json temps are °F)"
  steps = []
  if token_expires_in() < REFRESH_MARGIN:
    rc, out, err = sh([TESLA, "refresh"], 90)
    if rc != 0:
      return "failed", "token refresh failed (needs re-approval?): " + scrub(err, 100)
    steps.append("token refreshed")
  try:
    direct = vin in open(os.path.join(DIR, "direct_cmd_vins")).read().split()
  except OSError:
    direct = False
  if not direct and not proxy_up():
    rc, out, err = sh([START_PROXY], 30)
    time.sleep(1)
    if not proxy_up():
      return "failed", "proxy not running: " + scrub(err or out, 100)
    steps.append("proxy started")
  # one read gives state, shift, outside temp, and seat config; a 408 means asleep
  rc, d, err = tjson(["data", vin, "drive_state;climate_state;vehicle_config"], 45)
  resp = d.get("response") if isinstance(d, dict) else None
  if not isinstance(resp, dict):
    if not asleep(d, err):
      return "failed", "vehicle_data read failed: " + scrub(d if d else err, 120)
    lw = st.get("last_wake")
    if not isinstance(lw, dict):
      lw = st["last_wake"] = {}
    last = lw.get(vin, 0)
    if time.time() - last < WAKE_GAP:
      return "skipped", f"car asleep; already woken {int((time.time()-last)/60)} min ago for {st.get('last_wake_entry', '?')}, no second wake within 20 min"
    ok = False
    for attempt in (1, 2):
      lw[vin], st["last_wake_entry"] = time.time(), entry["id"]
      save_state(st)
      rc, out, err = sh([TESLA, "wake", vin], 150)
      if rc == 0 and "online" in out:
        ok = True
        steps.append("woke" + (" (retry)" if attempt == 2 else ""))
        break
      steps.append(f"wake {attempt} failed: " + scrub(err, 60))
      if "wake rate limit" in err:
        time.sleep(20)
    if not ok:
      return "failed", "couldn't reach the car (offline or no signal; 408 after wake + retry)"
    rc, d, err = tjson(["data", vin, "drive_state;climate_state;vehicle_config"], 45)
    resp = d.get("response") if isinstance(d, dict) else None
    if not isinstance(resp, dict):
      return "failed", "vehicle_data read failed after wake: " + scrub(d if d else err, 100)
  else:
    steps.append("online")
  shift = (resp.get("drive_state") or {}).get("shift_state")
  if shift in DRIVING:
    return "skipped", f"car is being driven (shift {shift}); not touched"
  cs = resp.get("climate_state") or {}
  oc = cs.get("outside_temp")
  outside_f = round(oc * 9 / 5 + 32, 1) if isinstance(oc, (int, float)) else None
  has_cool = bool((resp.get("vehicle_config") or {}).get("has_seat_cooling")) and vin not in st.get("cooling_unsupported", [])
  steps.append(f"outside {outside_f}F" if outside_f is not None else "outside n/a")
  c = f_to_c(temp_f)
  ok, r = cmd(vin, "set_temps", {"driver_temp": c, "passenger_temp": c})
  if not ok:
    return "failed", ("missing key (signed command rejected): " if key_error(r) else "set_temps failed: ") + r
  steps.append(f"set_temps {c}C")
  ok, r = cmd(vin, "auto_conditioning_start")
  if not ok:
    return "failed", ("missing key (signed command rejected): " if key_error(r) else "auto_conditioning_start failed: ") + r
  steps.append("climate on (command ok)")
  want, lvl = seat_plan(entry, outside_f, has_cool)
  both = (entry.get("seats") or "driver").lower() == "both"
  if want == "heat":
    for pos in ([0, 1] if both else [0]):
      ok, r = cmd(vin, "remote_seat_heater_request", {"heater": pos, "seat_position": pos, "level": lvl})
      steps.append(f"seat heat {pos} L{lvl} {'ok' if ok else 'failed: ' + r}")
  elif want == "cool":
    # proxy v0.4.1 sends seat_cooler_level as the car enum (1 off..4 high), so level L goes as L+1; positions 1 FL, 2 FR
    for pos in ([1, 2] if both else [1]):
      ok, r = cmd(vin, "remote_seat_cooler_request", {"seat_position": pos, "seat_cooler_level": lvl + 1})
      steps.append(f"seat cool {pos} L{lvl} {'ok' if ok else 'failed: ' + r}")
      if not ok and not re.search(r"429|timeout|asleep|offline", r, re.I):
        if vin not in st.setdefault("cooling_unsupported", []):
          st["cooling_unsupported"].append(vin)
        break
  elif want == "cool-unsupported":
    steps.append("seat cool skipped (no ventilated seats)")
  return ("partial" if any("failed" in x for x in steps) else "done"), "; ".join(steps)


def parse_t(s):
  t = dt.datetime.fromisoformat(s)
  return t.astimezone() if t.tzinfo is None else t

def tick():
  entries = entries_of(load_schedule())
  t = now()
  st = load_state()
  backoff = st.setdefault("backoff", {})
  stale, due = {}, []
  for e in entries:
    if e.get("status") != "pending" or not e.get("id"):
      continue
    try:
      cs = parse_t(e["climate_start"])
    except Exception:
      stale[e.get("id")] = {"status": "failed", "result": "bad climate_start", "reported": False}
      continue
    delta = (t - cs).total_seconds()
    if delta > STALE:
      why = "rate limited (429) until window closed" if e.get("id") in backoff else "missed: more than 10 min past climate_start; car not touched"
      stale[e["id"]] = {"status": "skipped", "result": why, "reported": False}
      backoff.pop(e.get("id"), None)
    elif delta >= -LEAD and time.time() >= backoff.get(e["id"], {}).get("next", 0):
      due.append(e)
  if stale:
    update_entries(stale)
    save_state(st)
    for i, u in stale.items():
      log(f"{i} {u['status']}: {u['result']}")
  groups = {}
  for e in sorted(due, key=lambda e: parse_t(e["climate_start"])):
    groups.setdefault(e.get("vin"), []).append(e)
  for vin, grp in groups.items():
    lead = grp[0]
    try:
      status, result = execute(lead, st)
    except RateLimited as ex:
      wait = 60
      for e in grp:
        b = backoff.get(e["id"], {"n": 0})
        b["n"] += 1
        wait = min(60 * 2 ** (b["n"] - 1), 240)
        b["next"] = time.time() + wait
        backoff[e["id"]] = b
      save_state(st)
      log(f"{lead['id']} 429 rate limited; retry in {wait}s ({scrub(ex, 80)})")
      continue
    except Exception as ex:
      status, result = "failed", f"runner error: {type(ex).__name__}: {scrub(ex, 100)}"
    st["last_run_ok"], st["last_run_at"] = status == "done", time.time()
    upd = {}
    for k, e in enumerate(grp):
      upd[e["id"]] = {"status": status, "result": scrub(result if k == 0 else f"shared run with {lead['id']}: {result}", 400), "reported": False}
      backoff.pop(e["id"], None)
    save_state(st)
    update_entries(upd)
    log(f"{','.join(e['id'] for e in grp)} {status}: {result}")

def main():
  global ARGS
  ap = argparse.ArgumentParser()
  ap.add_argument("--once", action="store_true")
  ap.add_argument("--schedule", default=os.path.join(DIR, "schedule.json"))
  ARGS = ap.parse_args()
  sched = os.path.abspath(ARGS.schedule)
  base = os.path.dirname(sched)
  P.update(schedule=sched, schedule_lock=sched + ".lock", state=os.path.join(base, "runner_state.json"),
       log=os.path.join(base, "runner.log"), pid=os.path.join(base, "runner.pid"))
  os.umask(0o022)
  claim_pid()
  log(f"runner start pid {os.getpid()} schedule={sched}")
  last_err = None
  try:
    while True:
      try:
        tick()
        last_err = None
      except Exception as ex:
        m = f"tick error: {type(ex).__name__}: {scrub(ex, 150)}"
        if m != last_err:
          log(m)
        last_err = m
      if ARGS.once:
        break
      time.sleep(TICK)
  finally:
    try:
      if open(P["pid"]).read().strip() == str(os.getpid()):
        os.unlink(P["pid"])
    except OSError:
      pass

if __name__ == "__main__":
  main()
```

/home/box/agent-data/projects/tesla-ready/tesla.sh (chmod 755)
```bash
#!/usr/bin/env bash
# usage: tesla.sh refresh | list | fleet VIN... | state VIN | wake VIN | data VIN [endpoints] | cmd VIN command [json]   (never prints tokens)
set -euo pipefail
umask 077
DIR=/home/box/agent-data/projects/tesla-ready
S="$DIR/.secrets"
TOK="$S/tokens.json"
API=$(cat "$DIR/fleet_api_url" 2>/dev/null || echo https://fleet-api.prd.na.vn.cloud.tesla.com)
PROXY=https://127.0.0.1:4443
AUTH_URL=https://fleet-auth.prd.vn.cloud.tesla.com/oauth2/v3/token
WAKELOG="$S/wake_times"
die(){ echo "error: $*" >&2; exit 1; }
authhdr(){ printf 'Authorization: Bearer %s\n' "$(jq -r .access_token "$TOK")"; }
api(){ # method url [json] [curl args]; no curl --retry: it re-sends 408/429, and each try is billed
  local m=$1 u=$2 body=${3:-}
  if [[ -n "$body" ]]; then
    curl -sS --max-time 60 -X "$m" -H @<(authhdr) -H 'Content-Type: application/json' --data "$body" "${@:4}" "$u"
  else
    curl -sS --max-time 30 -X "$m" -H @<(authhdr) "${@:4}" "$u"
  fi
  echo
}
refresh(){
  exec 9>"$DIR/tokens.lock"; flock -w 60 9 || die "token lock busy"
  local cid rt tmp resp
  cid=$(tr -d '[:space:]' < "$S/client_id")
  rt=$(jq -r .refresh_token "$TOK")
  [[ -n "$rt" && "$rt" != null ]] || die "no refresh_token in tokens.json"
  resp=$(curl -sS -X POST "$AUTH_URL" -H 'Content-Type: application/x-www-form-urlencoded' \
    --data-urlencode grant_type=refresh_token --data-urlencode "client_id=$cid" \
    --data-urlencode "refresh_token@"<(printf '%s' "$rt"))
  if ! jq -e '.access_token and .refresh_token' >/dev/null 2>&1 <<<"$resp"; then
    echo "refresh failed:" >&2
    jq '{error, error_description}' <<<"$resp" >&2 2>/dev/null || echo "(non-JSON response)" >&2
    exit 1
  fi
  tmp=$(mktemp "$S/.tokens.XXXXXX")
  printf '%s\n' "$resp" > "$tmp"; chmod 600 "$tmp"; mv -f "$tmp" "$TOK"
  date +%s > "$S/tokens_obtained_at"; chmod 600 "$S/tokens_obtained_at"
  echo "token refreshed; expires_in=$(jq -r .expires_in "$TOK")s"
}
vstate(){ api GET "$API/api/1/vehicles/$1" | jq -r '.response.state // .'; }
wake(){
  local vin=$1 now cnt st i
  now=$(date +%s); touch "$WAKELOG"
  cnt=$(awk -v n="$now" '$1>n-60' "$WAKELOG" | wc -l)
  (( cnt < 3 )) || die "wake rate limit: $cnt wakes in last 60s (max 3/min)"
  echo "$now" >> "$WAKELOG"
  st=$(api POST "$API/api/1/vehicles/$vin/wake_up" | jq -r '.response.state // .error // "?"')
  [[ "$st" == online ]] && { echo online; return 0; }
  for ((i=0;i<4;i++)); do
    sleep 10
    st=$(vstate "$vin"); echo "  state: $st" >&2
    [[ "$st" == online ]] && { echo online; return 0; }
  done
  die "vehicle not online after ~40s (last state: $st)"
}
case "${1:-}" in
  refresh) refresh ;;
  list) api GET "$API/api/1/vehicles" | jq '[.response[]? | {vin, display_name, state}]' ;;
  fleet) [[ $# -ge 2 ]] || die "usage: fleet VIN..."; api POST "$API/api/1/vehicles/fleet_status" "$(jq -nc '{vins: $ARGS.positional}' --args "${@:2}")" | jq . ;;
  state) [[ $# -ge 2 ]] || die "usage: state VIN"; api GET "$API/api/1/vehicles/$2" | jq . ;;
  wake) [[ $# -ge 2 ]] || die "usage: wake VIN"; wake "$2" ;;
  data) [[ $# -ge 2 ]] || die "usage: data VIN [endpoints]"
        if [[ -n "${3:-}" ]]; then api GET "$API/api/1/vehicles/$2/vehicle_data" "" -G --data-urlencode "endpoints=$3" | jq .
        else api GET "$API/api/1/vehicles/$2/vehicle_data" | jq .; fi ;;
  cmd) [[ $# -ge 3 ]] || die "usage: cmd VIN command [json]"
       if grep -qxF "$2" "$DIR/direct_cmd_vins" 2>/dev/null; then api POST "$API/api/1/vehicles/$2/command/$3" "${4:-{\}}"
       else api POST "$PROXY/api/1/vehicles/$2/command/$3" "${4:-{\}}" --cacert "$S/proxy-tls-cert.pem"; fi ;;
  *) sed -n 2p "$0"; exit 1 ;;
esac
```

/home/box/agent-data/projects/tesla-ready/start-proxy.sh (chmod 755)
```bash
#!/usr/bin/env bash
# idempotent: Tesla vehicle-command proxy on 127.0.0.1:4443
set -euo pipefail
DIR=/home/box/agent-data/projects/tesla-ready
S="$DIR/.secrets"
BIN="${TESLA_PROXY_BIN:-$HOME/go/bin/tesla-http-proxy}"
LOG="$DIR/proxy.log"
PIDF="$DIR/proxy.pid"
if [[ -f "$PIDF" ]] && kill -0 "$(cat "$PIDF")" 2>/dev/null; then echo "proxy already running (pid $(cat "$PIDF"))"; exit 0; fi
pid=$(pgrep -f "^$BIN .*-port 4443" || true); pid=${pid%%$'\n'*}
if [[ -n "$pid" ]]; then echo "$pid" > "$PIDF"; echo "proxy already running (pid $pid)"; exit 0; fi
setsid nohup "$BIN" -host 127.0.0.1 -port 4443 -key-file "$S/private-key.pem" \
  -cert "$S/proxy-tls-cert.pem" -tls-key "$S/proxy-tls-key.pem" \
  -session-cache "$S/proxy-session-cache.json" -verbose >>"$LOG" 2>&1 < /dev/null &
echo $! > "$PIDF"
sleep 1
if kill -0 "$(cat "$PIDF")" 2>/dev/null; then echo "proxy started (pid $(cat "$PIDF"))"
else echo "proxy failed to start; see $LOG" >&2; tail -5 "$LOG" >&2; exit 1; fi
```

/home/box/agent-data/projects/tesla-ready/start-runner.sh (chmod 755)
```bash
#!/usr/bin/env bash
DIR=/home/box/agent-data/projects/tesla-ready
setsid nohup "$DIR/runner.sh" >> "$DIR/runner.log" 2>&1 < /dev/null &
sleep 1
echo "runner pid: $(cat "$DIR/runner.pid" 2>/dev/null || echo '?')"
```

Start: `/home/box/agent-data/projects/tesla-ready/start-proxy.sh && /home/box/agent-data/projects/tesla-ready/start-runner.sh` (the pidfile stops a second runner).
Health check: `kill -0 $(cat /home/box/agent-data/projects/tesla-ready/runner.pid) && grep -q runner.sh /proc/$(cat /home/box/agent-data/projects/tesla-ready/runner.pid)/cmdline`, else run start-runner.sh. Read the last lines of runner.log; a "tick error" line means schedule.json is unreadable: fix it right away.
Use tesla.sh for every manual call (refresh, list for VINs, fleet VIN... for fleet_status and key pairing, state, wake, data, cmd) so tokens never print; it is also what the token-refresh routine runs.
