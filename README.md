# Tesla Ready (Grok Bot) — public instructions

These are the skills behind the [Tesla Ready](https://x.ai/bot/L5Sczwabgsf59rZ6RTChF) Grok Bot: what it does, how setup works, and how it talks to your car.

**Install:** https://x.ai/bot/L5Sczwabgsf59rZ6RTChF

## What it does
**Free.** Warms or cools a Tesla cabin so it is at temperature when you leave (calendar + traffic, or on demand). One-time phone setup; Tesla's free $10/month developer credit covers normal use — no card required.

## How it talks to the car
Only through **Tesla's official Fleet API**. You create your own free Tesla developer app; the bot hosts a public key file (Netlify or GitHub), you approve access on your phone, and you add the bot as a key on the car. Commands run from the bot's computer using Tesla's open-source vehicle-command tools.

It does **not** use Teslemetry, Tessie, TeslaFi, or any paid middleman that holds your tokens or relays commands.

## What's in this repo
| Folder | Skill |
|--------|--------|
| `getting-started/` | First-message pitch, existing-setup / reinstall check, intake |
| `operating-spec/` | Full behavior: setup steps, timing rules, failures, teardown |
| `runner/` | Exact `runner.sh`, `tesla.sh`, proxy helpers written on the bot's computer |

No secrets live here. Client ID/secret, tokens, and private keys stay only on the owner's Grok Bot computer under `.secrets/`.

## Cost
Normal use fits inside Tesla's free $10/month developer credit. No card is required for typical warm-ups.

## License
Instructions and scripts in this repo are published so you can audit before you install. Use at your own risk; Tesla's terms apply to Fleet API use.
