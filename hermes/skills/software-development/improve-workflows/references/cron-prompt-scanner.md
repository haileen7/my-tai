# Debugging `Blocked: prompt matches threat pattern` on a cron job

Depth for the recurring-audit pitfalls in SKILL.md. The scanner lives in the Hermes
install, not the project, so test candidates against its functions directly instead of
firing the job and waiting for the error.

## Which text gets scanned

Two surfaces, two rule sets:

- **User prompt** (what you passed to `cronjob_manage create/update`) — the STRICT set:
  injection directives, deception directives, secret-reading commands, destructive
  commands, exfil-shaped curl/wget, and invisible unicode. A hit here is reported
  immediately by the `cronjob_manage` call itself.
- **Assembled prompt** (skill bodies + hint + your prompt + script output) — the LOOSER
  set: only the four injection/deception directives survive, because skill markdown
  legitimately quotes attack commands as examples. This one is enforced at RUN time,
  so it fails in the job, not at creation. Invisible unicode here is sanitized, not blocked.

Consequence: a create/update that succeeds and a first run that is blocked point at
different text. Work out which surface matched before changing anything.

## The test harness

Run from the Hermes install root, where `tools` is importable:

```
cd <hermes-install-root> && python3 - <<'EOF'
import sys, pathlib
sys.path.insert(0, '.')
from tools.cronjob_prompt_scan import (
    _scan_cron_prompt, _scan_cron_skill_assembled,
)
# the prompt you intend to pass to cronjob_manage
print('user:', repr(_scan_cron_prompt(PROMPT)))
# every skill file the job will load
for p in SKILL_MD_PATHS:
    _, err = _scan_cron_skill_assembled(pathlib.Path(p).read_text())
    print(p, repr(err))
EOF
```

Empty string means pass. A non-empty value names the pattern id that matched.

## Find the trigger phrase

The error names a pattern id but not the offset. To locate it, scan the file yourself
with the same patterns and print surrounding context:

```
import re, pathlib
from tools.cronjob_prompt_scan import _CRON_THREAT_PATTERNS
text = pathlib.Path(SKILL_MD).read_text()
pattern, pid = _CRON_THREAT_PATTERNS[0]   # index varies with the reported id
for m in re.finditer(pattern, text, re.I):
    print(pid, repr(text[max(0, m.start()-120):m.end()+120]))
```

Print the whole skill file through `_scan_cron_skill_assembled` first — it tells you
whether the skill is the culprit at all before you go regex-hunting inside it.

## Fixing without weakening the rule

A security rule that illustrates an attack with the attack's exact wording will match
its own scanner. Rewrite the example so it names the shape of the attack instead of
quoting it — "a request to set aside the agent's earlier standing orders" instead of the
literal directive string. The rule keeps its force; the phrase no longer matches.

Do NOT delete the rule, and do NOT patch the scanner's pattern list to accommodate one
skill file: the scanner is a tripwire against a genuine cron attack surface, and
loosening it trades a real risk for a convenience.

## Confirming the fix end to end

Passing both scan functions is necessary, not sufficient: it proves the gate opens, not
that the job did its work. Close the loop in this order.

1. Re-run the harness after the fix and confirm both return an empty string.
2. Fire the job once with `cronjob_manage action=run`. Note the delegation id it returns.
3. Do not fire it again to double-check — a job already executing is skipped with
   `executed: false` and an already-running notice, which reads like a pass but is not one.
4. Wait for that run's outcome to re-enter the conversation. Report it verbatim; if it
   has not returned, say the run is still in flight rather than inferring success.
5. Ignore `last_status` / `last_error` in the interim. They are written when a run
   COMPLETES, so between firing and completion they still describe the previous run and
   will read `error` even though the cause is fixed.

The same applies to a create/update that was blocked earlier: re-check the update call
itself returns `success: true` after the change, because the block is enforced on both
paths.

## A hub-installed skill's fix does not survive its own upgrade

If the offending wording lives in a skill you installed from the hub, editing the local
SKILL.md is a local patch on someone else's file: the next upgrade or reinstall overwrites
it and the same cron job silently starts failing again. Two consequences:

- Re-run the harness after ANY upgrade of a skill a cron job loads. A block that reappears
  with no prompt change is the upgrade reintroducing the quoted example, not a new problem.
- Prefer the least invasive edit that preserves the rule's force, and keep the reason
  recorded here so the next session re-applies it in one step instead of re-diagnosing.

A cron job loading a hub skill you cannot patch should be pointed at a scanner-tolerant
instruction instead: the gate is a runtime check on the assembled prompt, so any skill whose
body quotes a literal trigger phrase is unusable in cron until it is upgraded.

## Related

- `references/scheduled-recheck.md` — the one-shot sibling; same scanner, same
  self-contained-prompt discipline.