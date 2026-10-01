# Verifying an incoming change in isolation

Use when you must decide whether someone else's change is sound — reviewing a PR,
validating a patch someone handed you, or reproducing a claim in a changelog — and
you need to run **their** test suite, not your own working tree.

A green CI badge is not verification. The runner has the full checkout, the built
assets, and a seeded environment. Reproducing locally is what finds defects no check
can see. A change that cites a test count is claiming a number you can reproduce, so
reproduce it and compare.

## Worktree, not checkout

Never check the change out over the current branch — the user's work must be
untouched when you finish.

```bash
WT="$(mktemp -d)/incoming"
git worktree add --detach "$WT" <sha>      # fetch the ref first if it lives in a fork
cp .env "$WT/.env"                          # gitignored; copy it, never echo secrets
mkdir -p "$WT"/storage/framework/{views,cache,sessions} "$WT"/storage/logs "$WT"/bootstrap/cache
```

Read the change itself without leaving the current branch:

```bash
git diff <base>..<sha> --stat
git show <sha>:path/to/file.php             # any file at that ref, no checkout needed
```

## Order of operations

1. Formatter check on changed files
2. Static analysis
3. Framework suite (Pest/PHPUnit)
4. Browser suite (Playwright/Cypress)

The tiers exercise different layers; a change is not verified on one of them. Keep each
gate's tail as the evidence, and report counts per tier.

## Pitfalls that silently invalidate the entire run

- **Never symlink `vendor/` into the worktree.** A symlink resolves back to the main
  repo, so the autoloader's base path points at the *base* checkout — your tests
  silently run the old code and report a clean pass. This is the most dangerous
  failure in this whole workflow because it produces a *false green*, not an error.
  Copy it (`cp -a vendor "$WT/vendor"`) and confirm:

  ```bash
  php -r 'echo realpath("vendor"), PHP_EOL;'   # must point inside the worktree
  ```

- **Page-rendering tests need the built assets — but rebuild them if the change
  touches frontend source.** A missing `public/build/manifest.json` fails every page
  test with an unrelated Vite error, which reads like a real regression. Copying the
  main repo's `public/build/` in is only safe when the change is backend-only; the
  moment the diff includes JS/CSS/Vue/Blade-facing assets, the copied bundle is the
  *base* build and the browser tier silently exercises the old frontend while
  reporting a clean pass. Rebuild inside the worktree whenever frontend source is in
  the diff:

  ```bash
  ln -sfn "$PWD/node_modules" "$WT/node_modules"   # deps may be symlinked; assets may not
  (cd "$WT" && npm run build)
  git -C "$WT" status --short public/build         # must show the new bundle
  ```

  Symlinking `node_modules` is fine (the bundler resolves it per file at build time);
  the fatal symlink is `vendor/`, whose autoloader base path is baked in at install.
- **Frontend interactions prove the UI change, not just the store behind it.** A
  browser spec that reads state through the framework's public store (`Alpine.store`,
  a component's data bag) verifies the data layer and can pass while the widget the
  user actually touches is broken. Drive the rendered control — the checkbox in the
  filter panel, the button in the toolbar — and assert the observable result. Also
  assert the invariant the fix was about on the *live* object, not on a copy the test
  made before the interaction.
- **An assertion can pass for a reason unrelated to the fix, and the tally still looks
  clean.** Listener counts, timer registrations, and cache sizes have legitimate
  baseline values contributed by layers the change never touched. Before flagging a
  threshold as "data-dependent and flaky", count the real contributors: for Leaflet,
  only `TileLayer`, `Tooltip`, and the shared `Renderer` emit a `zoomanim` handler —
  `Path` (and so `Polyline`) does not implement `getEvents` at all, so N polylines add
  zero listeners. An inflated-headroom claim based on a plausible-sounding class
  hierarchy is a common and embarrassing false positive.
- **One test database, one suite at a time.** Whatever the suite config names is the
  database it mutates; a background full run plus anything else against it corrupts both.
- **A local edit to a file the incoming change also touches blocks the checkout.** If
  your diff is identical to theirs, discard yours and proceed; otherwise stash it.
- **Foreign keys block "reset this row to zero" probes** in relational schemas. Build a
  dedicated fixture subtree with the exact value instead of mutating existing rows.
- **Finish clean:** `git status --short` in the worktree, delete every probe file, and
  kill any server the run started. A stray probe that asserts `true` is worse than
  no probe.

## Probing behaviour the suite misses

A fully green suite can still hide a defect. Write a **throwaway probe test** in the
worktree that forces the specific interaction and prints before/after state, so the
measurement itself is the evidence:

```php
$c = Livewire::test('comp', [...])->assertStatus(200);
fwrite(STDERR, "[BEFORE] expected present = " . (str_contains($c->html(), $expected) ? 'YES' : 'NO') . "\n");
$c->call('someAction');                       // cross the round-trip boundary
fwrite(STDERR, "[AFTER ] expected present = " . (str_contains($c->html(), $expected) ? 'YES' : 'NO') . "\n");
```

**Derive the expected value from the database, never hardcode it.** A seeder or factory
may already have created a row, so a hardcoded number will certify broken code as
correct. Print the DB count in the same run; if `BEFORE` does not match, the probe is
wrong, not the code.

For stateful components, first-mount assertions prove nothing — see
`references/livewire-hydration.md` for what breaks only after a round-trip.

## Report only what you reproduced

A finding is reportable when you have measured it yourself. Treat a subagent's or a
review tool's claim as a hypothesis: run the probe, and discard the claim if it does
not reproduce. A wrong "confirmed" finding costs more credibility than a missed one,
and it damages the author's work.

**Verify a subagent's correction before you publish it — and verify your own claims
with the same standard.** A reviewer agent is not more reliable than you at reading a
minified bundle; this session one got a real finding right and another wrong, and the
wrong one was wrong *because* it was plausible-sounding. When an agent contradicts
something you wrote, treat it as a hypothesis too: re-grep the source and decide on
the evidence, not on who said it.

**When a published finding turns out to be wrong, retract it on the same thread.**
Silence leaves the author acting on a non-issue, and a private "actually never mind"
leaves the wrong claim as the last word. Post the correction inline at the original
line, name the specific claim that was wrong, state what the source actually shows,
and say whether the review verdict changes. A retraction is cheap; a bad finding that
the author already fixed around is not.

## Regression or pre-existing — always distinguish

Before blaming the change, read the same code at the base ref:

```bash
git show <base>:path/to/file.php
```

- Present in both → **pre-existing** defect: a separate issue, not this change's fault
- Absent at base, introduced by the change → **regression**: blocks approval
- The change **added** a guard, test, or normalisation the base lacked → an
  improvement; mislabelling it a regression is wrong in the other direction

State which one you mean, with the base-ref evidence. A refactor that also fixes a
pre-existing bug is still a refactor, and the fix belongs in the summary, not the
blocker list.
