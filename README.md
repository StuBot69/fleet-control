# Fleet Control

A small local dashboard for running a fleet of [Hermes Agent](https://github.com/NousResearch/hermes-agent) profiles on free and cheap model endpoints. It shows every profile's main model and fallback chain, sweeps the endpoints for health, and lets you change models safely. You preview the change as a diff first, and only an explicit Apply writes anything.

It's a personal tool, shared as-is. It works for one Mac running Hermes with about 20 profiles; expect to adapt paths and lanes to your own setup.

## What it does

- **Model assignment:** one table of every profile's main model and fallbacks, with each dropdown listing every endpoint in the live catalogue, grouped by lane (provider).
- **Preview, then apply:** changes go amber, *Preview changes* shows the exact diff, and *Apply* backs up each `config.yaml` before writing. Only the model, fallback and providers sections change; comments and every other setting keep their text.
- **Wiring checks:** a profile can only use a lane if its config defines it and its `.env` holds the key. `wiring.py` finds and closes those gaps, copying stanzas from the rest of the fleet and keys from `~/.hermes/.env` (key *names* are reported, never values).
- **Health sweeps and watchdog:** `refresh.py` probes every catalogue endpoint into `fleet.db`, and `watchdog.py` reports only new or cleared failures, so a known outage doesn't alert every run.
- **Machines and Hermes TV:** the page shares a nav and a live 3D-city backdrop with Hermes TV (a separate fleet viewer), and shows which machines in the fleet are running.

## Requirements

- macOS or Linux, Python 3.11+.
- Hermes Agent installed, with profiles under `~/.hermes/profiles/`.
- PyYAML, plus `ruamel.yaml` for comment-preserving writes. Both come with Hermes's own Python environment, which is the easiest way to run this.
- Hermes's home is expected at `~/.hermes`. A local GPU lane, `local_gpu`, points at `http://192.168.1.50:8080`; change that address in `catalog.py`, `refresh.py` and `app.py` to your own model server, or ignore it.

## Run it

```bash
~/.hermes/hermes-agent/venv/bin/python3 app.py 8787
```

Then open http://127.0.0.1:8787/. It only listens on localhost.

Useful one-offs:

```bash
python3 wiring.py          # report lanes a profile routes to but can't use
python3 wiring.py --fix    # close the fixable gaps (backups first)
python3 refresh.py         # catalogue + health sweep into fleet.db
python3 watchdog.py        # one watchdog pass
```

## Safety model

| Endpoint | Effect |
|---|---|
| `GET /api/state`, `/api/catalog`, `/api/wiring`, `/api/gateway` | read-only |
| `POST /api/plan` | returns a diff, writes nothing |
| `POST /api/apply` | the only path that edits a profile config; refuses unless the payload has `apply=true`; backs up first |
| `POST /api/refresh` | writes `fleet.db` only |
| `POST /api/wiring/fix` | adds missing stanzas/keys, backups first |

Runtime files (`fleet.db`, logs, `backups/`, `watchdog-state.json`) are git-ignored and never committed.

## A note on NVIDIA (Hermes v0.23.3+)

NVIDIA is a built-in Hermes provider. Don't give it a `providers.nvidia` block with `api_key` or `base_url`: since v0.23.3 that makes Hermes route it as a *custom* provider, which skips a local proxy, can fail authentication silently behind the fallback chain, and ignores timeout settings. Keep the key and any proxy address in the profile's `.env` (`NVIDIA_API_KEY`, `NVIDIA_BASE_URL`); `providers.nvidia` may hold only timeout keys. `wiring.py` enforces this.

## Files

| File | Purpose |
|---|---|
| `app.py` | HTTP server, API and role guide |
| `ui/index.html` | the whole front end (single file) |
| `wiring.py` | lane/key gap finder and fixer; comment-preserving config writes |
| `catalog.py`, `refresh.py`, `probe_one.py` | endpoint catalogue and health sweep |
| `watchdog.py` | periodic checks with de-duplicated alerts |
| `fleetctl.py`, `inventory.py`, `verify_all.py`, `check_cost.py` | command-line helpers |

## Contributing

Issues and pull requests are welcome, but this is maintained in spare time for one fleet, so replies may be slow and changes that only suit other setups may not be merged. Forking to adapt it to your own fleet is the expected way to use it.

Please never paste API keys, `.env` contents or logs with keys into an issue.

## Licence

MIT, see [LICENSE](LICENSE).
