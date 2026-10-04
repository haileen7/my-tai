---
name: hermes-plugins
description: Use when adding Hermes plugins like superpowers.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, plugins, superpowers]
    related_skills: [hermes-agent]
---

# Hermes Agent Plugins

## When to Use

Adding or troubleshooting a Hermes Agent plugin such as obra/superpowers.

## Install procedure

1. Check current state: `grep -n -A8 'plugins:' ~/.hermes/config.yaml` and `ls ~/.hermes/plugins/`.
2. Clone the plugin repo into the plugins dir:
   `git clone --depth 1 https://github.com/<org>/<repo> ~/.hermes/plugins/<name>`
   Prefer the official CLI when it supports the source (`hermes plugins install <src>`); git clone is the verified fallback.
3. Confirm the plugin ships a Hermes manifest: `~/.hermes/plugins/<name>/.hermes-plugin/plugin.yaml` must exist.
4. Add the plugin name to `plugins.enabled` in `~/.hermes/config.yaml` if absent.
5. Restart the Hermes gateway. Plugins load at gateway start; an already-running session keeps its old plugin set, so changes are invisible until restart.

## Pitfalls

- A repo may ship manifests for OTHER harnesses (`.claude-plugin/`, `.codex-plugin/`, ...) that Hermes cannot parse; the real manifest lives in `.hermes-plugin/plugin.yaml`. Check `.hermes-plugin/` explicitly rather than relying on top-level manifests. Why: multi-harness repos (e.g. obra/superpowers) need one manifest per agent harness.
- `plugins.enabled` listing a name is not proof of installation — the directory may be missing. Verify the on-disk clone before reporting success.
- After restart, verify loading via skill registration or gateway logs rather than assuming; silent plugin disable happens when the manifest/module import fails.
