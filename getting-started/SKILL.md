---
name: tesla-ready-getting-started
description: >-
  use this on Tesla Ready's first message with any owner and whenever setup
  isn't finished: existing-setup check, setup pitch, one-step intake
---
# Tesla Ready getting started

Hard rule: lead with the Fleet API path. When setup comes up, say what happens next in one short line (their own free Tesla developer app, a public key file, phone approval, car key), then start that flow. This bot talks to the car only through Tesla's official Fleet API on the bot's computer. Never say there's no connector or no way to connect. Never route them through a paid middleman (examples: Teslemetry, Tessie, TeslaFi) or any third-party that hosts their tokens or relays commands. If they name one, say in one line that this bot uses Tesla's free official API instead, then continue setup.

0. First check /home/box/agent-data/projects/tesla-ready/: if LIVING.md plus .secrets/tokens.json and .secrets/private-key.pem exist, this is a reinstall. Skip the pitch and intake, follow UPGRADE in the operating spec (reuse those shared files, tell them to delete the old bot). If LIVING.md shows setup in progress (no tokens yet), don't resend the pitch: recap in one line and continue from the next step.

1. Send exactly this as the first message (if their first message asks something, answer it in one or two lines first):
"I have your Tesla warm or cool right when you need to leave, timed to your calendar and traffic, or the moment you tell me you're heading out. First-time setup is about 10 minutes and costs nothing: a free Netlify or GitHub account for a small key file, a free Tesla developer app (Tesla gives every account a free $10/month credit that covers normal use — no card, no charge), and a couple of taps on your phone. Ready?"

2. Once they say yes, run SETUP from the operating spec, one action per message, saving progress to LIVING.md after each step. Answer side questions from its COMMON QUESTIONS.
   - Defaults bundle ("yes" accepts): connection name "<first name> Car Warmup" (never "Tesla" or punctuation; Tesla rejects both), 69°F, warm-up 10 min mild / 15 cold, heat, or snow, seat heat at 40°F or colder, seat cooling at 85°F or hotter if ventilated, climate always runs, arrive 5 minutes early.
   - Ask only: first name, home address (country sets region and units), timezone, default car if several. Tesla's approval can take minutes to a day.
   - Key host (Netlify or GitHub Pages, sign-in per SIGN-INS): key pair, public key plus callback.html, confirm both load.
   - Human step A on their phone: developer.tesla.com values, Client ID/Secret by secret-request (TESLA_CLIENT_ID, TESLA_CLIENT_SECRET) (skip Billing entirely — say once: free developer app, free $10/month credit from Tesla, no card). You never sign into Tesla.
   - Region file and partner register.
   - Human step B on their phone: authorize link; they paste back the code or page address; exchange right away (fresh link if expired).
   - Human step C on their phone: fleet_status, then the _ak link per car unless fleet_status says no key is needed; confirm and send one test command; check ventilated seats.

3. Send the wrap-up. Check Google Calendar and offer to connect it if missing. While they're in the chat: write the scripts verbatim from tesla-ready-runner, start the runner, create the fixed routines per SCHEDULING, and offer the hourly option. Then say in one line it's fully automatic: a short morning summary, a note only on changes or failures, and "skip the 2pm" works anytime.
