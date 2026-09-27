---
name: hermes-messaging-platforms
description: Use when picking or auditing a Hermes gateway platform.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [hermes, gateway, messaging, platforms, adapters, config, setup]
    related_skills: [hermes-agent]
---

# Hermes Messaging Platforms

## When to Use

- A user asks which gateway/messaging platform works, which ones need extra tools, or which is a cheaper substitute for one that needs a binary/daemon (buzz, raft, whatsapp, signal, photon).
- A user asks to set up, enable, or audit a specific Hermes messaging channel, or asks what a platform's prerequisites and required env vars are.
- A user challenges a claim about a platform's setup cost or availability.
- Any "does Hermes support X / what does X need?" question about the gateway surface.

Do NOT use for TUI/desktop/dashboard surfaces, or for model/provider configuration.

## Where the truth lives

The source tree is at `$HERMES_HOME/hermes-agent` (default `~/.hermes/hermes-agent`; profiles move it — never hardcode `~/.hermes` when handing a path to a user).

- Builtin adapters: `gateway/platforms/<name>.py`
- Plugin adapters: `plugins/platforms/<name>/adapter.py`
- Every adapter ends with a `register_platform(...)` / `ctx.register_platform(...)` call carrying the declarative contract: `check_fn` (PASSIVE probe), `ensure_deps_fn` (ACTIVE pip installer), `required_env`, `install_hint`, `setup_fn`.
- Registry: `gateway/platform_registry.py`. Per-platform setup UX: `hermes_cli/gateway_setup_wizard.py`.
- Docs index: `https://hermes-agent.nousresearch.com/docs/llms.txt`; per-platform page at `/docs/user-guide/messaging/<name>`. The docs page is good for user-facing env vars; the CODE is canonical for gating.

## The dependency test (run this before answering anything about setup cost)

Three tiers, because they differ in what the user has to do:

1. **Nothing extra** — stdlib or already a core dep (`httpx`, `websockets`, `Markdown`, …). The user just configures credentials.
2. **A pip package** — `hermes setup` / `ensure_deps_fn` installs it. One command for the user; not a burden worth flagging as "extra tooling".
3. **An external binary or daemon** — the ONLY tier users mean by "needs extra tools". Always call this out.

Mechanical probes, in order:

```bash
cd "$HERMES_HOME/hermes-agent"
grep -rn "install_hint=\|ensure_deps_fn=" plugins/platforms/*/adapter.py gateway/platforms/*.py
grep -rn "shutil.which(\|subprocess.Popen" plugins/platforms/*/*.py gateway/platforms/*.py
head -20 gateway/platforms/signal.py gateway/platforms/bluebubbles.py   # daemon adapters name the external service in the docstring
```

Read the `install_hint` string first — it is the curated one-line answer and is exactly what the setup wizard shows the user. `ensure_deps_fn` present ⇒ pip tier. `shutil.which(<binary>)` or a `Popen` of a bridge ⇒ external-tooling tier. An adapter that only speaks HTTP to `127.0.0.1:<port>` (signal-cli, BlueBubbles) is the daemon tier even with no `which()` call — check the module docstring.

Confirm a pip tier against `pyproject.toml` before claiming a dep exists: check the `[messaging]` / `[matrix]` / per-platform extras AND the core `dependencies` list. A package in an extra is still "installed for you"; a package in neither is a real prerequisite.

## Service cost and quotas (when the user asks "is it free?")

Never call a backing service free or unlimited from the marketing page. Fetch the machine-readable tiers endpoint — typically an unauthenticated JSON array at `/v1/tiers` or `/api/pricing` — and quote the enforced numbers. The shape is usually a list of `{code, prices, limits}` objects where the entry with no `code` is the anonymous free tier.

Read `limits.basis` before recommending: a per-IP daily cap is invisible in normal use and then bites hard on a shared-egress server. Also check `reservations` (can anyone claim the topic/name you chose?) and any attachment/message expiry, because those decide whether the free tier is viable for a long-lived channel.

## Reporting results

- Answer with a tiered list of platform names, not a flat list of 25 platforms. The user asked which few work like Telegram, not for the catalogue.
- One line per platform: name, what it needs, the file that proves it.
- End with a concrete recommendation and the next action ("بگو تا ntfy را راه‌اندازی کنم؟"), not a survey.
- Match the user's language.

## Pitfalls

- **Don't classify by "needs a pip install".** Nearly every platform needs pip, and `hermes setup` does it. Conflating tiers makes a working platform look blocked.
- **Don't trust the docs index alone.** `/docs/user-guide/messaging` lists every platform but says nothing about which need external binaries. Fetch the per-platform page or grep the code.
- **Similar names, opposite cost.** `whatsapp` (Baileys Node bridge) vs `whatsapp_cloud` (pure HTTP); `matrix` (linux-gated pip extra) vs `irc` (stdlib). Resolve to the adapter file before asserting.
- **Don't truncate the registration call.** `check_fn` and `ensure_deps_fn` sit on the lines AFTER `name=`/`adapter_factory=`; a one-line grep gives the wrong tier.
- **Don't probe the interpreter on PATH.** A bare `python3 -c "import aiohttp"` failing says nothing about Hermes' managed runtime. Read `install_hint` and the extras instead.
- **The generic wizard is not the whole setup.** `hermes gateway setup` only invokes the platform's `setup_fn`. Read that function's body: if it just writes env vars, the real onboarding (QR pairing, device flow, port binding) lives in a dedicated subcommand — grep `hermes_cli/subcommands/<name>.py` and the `cmd_<name>` handlers in `hermes_cli/main_platform_setup.py`, which `hermes_cli/main.py` registers separately. Telling a user the wizard completes pairing when it cannot wastes their turn.
- **Enabling the inbound platform does not enable its outbound toolsets.** Toolset-gated platforms (a2a being the clear case) ship their client tools off by default and resolved per platform surface: `hermes tools enable <toolset> --platform <name>`. Enabling the platform in the wizard changes nothing about what the agent can call.
- **Never answer "Hermes can't do X" from memory.** If a platform isn't in the tree you found, check `llms.txt` before concluding it doesn't exist.

## Depth

`references/platform-dependency-matrix.md` — full tier table with adapter paths, external tools, and required env vars.
