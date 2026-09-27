# Platform dependency matrix

Derived from the source tree (`$HERMES_HOME/hermes-agent`, `plugins/platforms/*/adapter.py` + `gateway/platforms/*`).
Re-run the greps in SKILL.md before quoting — adapters move; this is a map, not the contract.

## Tier 1 — no extra tooling (stdlib or core dep)

| Platform | Adapter | Notes |
|---|---|---|
| ntfy | `plugins/platforms/ntfy/adapter.py` | `httpx` only; phone push. Topic name IS the identity — `NTFY_ALLOWED_USERS` = the topic itself. 4096-char body cap. |
| irc | `plugins/platforms/irc/adapter.py` | stdlib sockets. `IRC_SERVER` + `IRC_CHANNEL` + `IRC_NICKNAME`. |
| email | `plugins/platforms/email/adapter.py` | stdlib `smtplib`/`imaplib`. `EMAIL_ADDRESS`, `EMAIL_PASSWORD`, `EMAIL_SMTP_HOST`, `EMAIL_IMAP_HOST` — blank keys count as missing. |
| a2a | `plugins/platforms/a2a/__init__.py` | stdlib `http.server` on a daemon thread; `check_requirements()` is always True. Binds `127.0.0.1` unless a bearer token is configured, then accepts remote. Default port 9900; Agent Card at `GET /.well-known/agent-card.json`, JSON-RPC 2.0 at `POST /`. Outbound client tools are a separate `a2a` toolset, off by default per platform (`hermes tools enable a2a --platform <name>`). `A2A_ALLOWED_USERS` is deliberately bypassed — peers authenticate by token/IP, not user allowlist. |
| whatsapp_cloud | `gateway/platforms/whatsapp_cloud.py` | Cloud API over HTTP. Distinct from `whatsapp` below. |

## Tier 2 — pip package, auto-installed by `hermes setup` / `ensure_deps_fn`

slack, discord, google_chat, teams, line, mattermost, sms, homeassistant, feishu, dingtalk, weixin, wecom,
webhook + whatsapp_cloud (via the `messaging` extra, which carries `aiohttp`),
simplex (`websockets`), matrix (`mautrix[encryption]` — extra is linux-gated; no Windows/macOS wheels).

## Tier 3 — requires an external binary or daemon

| Platform | External requirement | Signal in code |
|---|---|---|
| buzz | `buzz` CLI on PATH or `BUZZ_CLI_PATH` (Rust, `github.com/block/buzz`) | `shutil.which("buzz")`; outbound always shells out to the CLI. Inbound is native Nostr over the bundled `websockets` package, so the CLI is only strictly needed for OUTBOUND send. |
| raft | `raft` CLI (`raft.build`) | `shutil.which("raft")`; without it degrades to wake-only polling |
| whatsapp | Node.js + Baileys bridge (`@whiskeysockets/baileys`) | `find_node_executable("node")` + `subprocess.run([npm, "install", ...])` |
| photon | `spectrum-ts` sidecar via `hermes photon setup` | node binary resolution / `PHOTON_NODE_BIN` |
| signal | `signal-cli daemon` running separately (HTTP on `127.0.0.1:8080`) | module docstring; adapter only GETs `/api/v1/check`. Wizard prints "Signal requires signal-cli running as an HTTP daemon." |
| bluebubbles | BlueBubbles macOS server | adapter REST-sends to the local server |
| msgraph_webhook | Microsoft Graph subscription on the tenant | webhook registration flow |

## The wizard is not always the full setup

`hermes gateway setup` dispatches to each platform's `setup_fn` (`hermes_cli/gateway_setup_wizard.py::_configure_platform`). Several of those only write env vars. Onboarding that needs a terminal, a QR scan, or a device flow lives in a dedicated subcommand instead — the `hermes whatsapp` vs `hermes gateway setup` split is the canonical example. Check `hermes_cli/subcommands/` and the `cmd_*` handlers before telling a user which command to run.

```bash
hermes whatsapp              # Baileys: mode choice, npm install, QR pair → creds.json
hermes whatsapp-cloud        # Meta Cloud API wizard
hermes photon setup          # device flow + spectrum-ts sidecar
hermes tools enable a2a --platform telegram    # toolset is separate from the platform
```

## Service cost and quotas

Backing services change pricing; the adapter file says nothing about it. Fetch the provider's unauthenticated JSON tiers endpoint (ntfy: `https://ntfy.sh/v1/tiers`, shape `[{limits}, {code, prices, limits}, ...]` where the bare object is the anonymous tier) and read `limits.basis` (`ip` vs `tier`) before advising. A per-IP cap looks free in single-user use and breaks a shared-egress server.

## Configuring one

```bash
hermes gateway setup     # interactive wizard; every registered platform appears here
hermes gateway status    # connection state + last reason, per platform
hermes pairing list      # DM allowlist alternative to env allowlists
```

Secrets in `~/.hermes/.env`; behavioural settings in `config.yaml` under `gateway.platforms.<name>.extra`.
Never put a non-credential setting in `.env`, and never hand-edit `config.yaml` for a user — use
`hermes config set KEY VAL`.
