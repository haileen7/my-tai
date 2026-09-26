---
name: laravel-feature-testing
description: "Add or fix Laravel tests and keep the quality gates green."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [laravel, livewire, pest, playwright, testing, phpstan, baseline, quality-gates]
    category: software-development
    related_skills: ["test-driven-development", "systematic-debugging", "requesting-code-review", "laravel-hermes-init"]
---

# Laravel Feature Testing

Writing tests for a new or changed Laravel + Livewire feature, repairing tests that
change broke, and running the gates that must be green before the work ships.

**This skill vs test-driven-development:** that one enforces RED-GREEN discipline;
this one carries the Laravel mechanics — fixtures, Livewire interaction, browser
specs, and the static-analysis gates that run beside the suite.

**The project's AGENTS.md / CLAUDE.md overrides this skill** on commands, seeders,
and which tiers exist. Read it first; this skill carries only what it does not
document.

## When to Use

- "Add tests" / "fix the tests" for a feature
- A change touching routes, permissions, seeders, Blade components, cache namespaces,
  or factories — i.e. anything other tests may have pinned
- Before commit, when a gate must be shown green with real numbers

## Always-On Rules

1. **See it red first.** Write the test, run only that file, and watch it fail for
   the reason you intended. A test never observed red proves nothing — a passing
   test you just wrote may be passing because it asserts nothing about your change.
2. **Two tiers, both green.** The framework suite (fast, full) and the browser spec
   suite (slower, environment-swapping) exercise different layers; a feature is not
   done on one tier. Report both with their pass counts.
3. **Fixtures come from seeders/factories in the test**, never from whatever is
   manually seeded in a dev database — CI has none of that.
4. **Repair dependents before declaring done.** Any identifier you changed — a
   hardcoded array, a permission name, a route, a cache namespace, a seeded table —
   gets grepped across the *whole* test tree *and* the browser spec tree, not just
   the file that failed.
5. **Do not run the browser tier concurrently with the framework tier.** Browser
   suites commonly swap an env file or rebuild a DB; two suites sharing those
   resources corrupt each other's run.
6. **One test process at a time per test database.** Every framework-tier run
   shares the one `DB_DATABASE` from `phpunit.xml` and mutates it, so a background
   full suite plus an isolation loop (or a worktree run) will corrupt each other.
   Before believing any mass-failure count, run `pgrep -af 'pest|artisan test'`,
   stop the strays, and re-run once alone. See
   `references/red-test-attribution.md`.
7. **A test is evidence only if it fails for the stated reason.** Two ways a test
   lies: the stored data it needs is rewritten before it is ever persisted, so it
   exercises a state the application cannot reach; or a multi-column match is
   already satisfied by a sibling field, so the assertion never reaches the code
   under test. Prove the fixture survives a write round-trip before writing the
   test, and make every non-target field in it non-matching. See the pitfalls.

## Procedure

1. **Find the seam.** Locate the existing test class or spec covering the same
   component or page and mirror its setup — base `TestCase`, `RefreshDatabase`,
   shared seed helpers, existing fixtures. A test that re-invents setup diverges
   from its neighbours and rots first.
2. **Seed the authorization layer deliberately.** With Spatie-style permissions,
   granting a permission to a *role* happens in the role seeder (usually by syncing
   every permission). Seeding only the permission seeder leaves roles without it, so
   an authorization assertion fails while the permission visibly exists — seed the
   seeder that wires roles, and assert with `hasPermissionTo()`, not `can()`.
3. **RED.** Run only the new file and confirm the failure:
   `php artisan test tests/Feature/ThingTest.php` — or for a spec,
   `npx playwright test tests/e2e/thing.spec.ts`.
4. **GREEN.** Make the smallest change that satisfies it.
5. **Repair dependents** (rule 4 above), then re-run the touched files.
6. **Gates in order:** formatter on changed files → static analysis → full suite →
   browser suite. Each gate's output is evidence; keep its tail.
7. **Isolate before theorizing.** A failure seen only in a full/randomised run must
   be reproduced alone before any code is read: if it passes alone it is order- or
   seed-dependent, and the next probe is re-running the suite with the seed the
   runner printed — not a source-code hypothesis.
8. **Report numbers, not adjectives**: counts of passed/failed per tier, plus the
   formatter and static-analysis verdicts.

## Pitfalls

- **A model write hook can make the "bad data" unreachable.** Before writing a
  regression test that persists malformed stored data — mixed script, an
  unnormalised character, a legacy spelling — check whether the model already
  rewrites it on write: `static::saving` / `creating` hooks, `Attribute` mutators,
  casts, observers. A `saving` hook that normalises a name makes that name
  impossible to persist, so the test can only fail through a path the app never
  takes and any "fix" for it is dead code. Reachability is then limited to rows
  written before the hook, imports, or fixtures that bypass the ORM — decide which
  of those you mean before writing the test, and say so in the report. Prove it
  with a round-trip (`Model::create([...])`, then read the column back and inspect
  its code points); do not reason from the attribute setter alone.
- **Never insert probe rows with `DB::table()->insert()` when the question is
  about what the application stores.** The query builder skips model events and
  accessors, so it will happily write states no code path can produce and you will
  "discover" a bug that does not exist. Use the model, or `Model::insert()` if you
  need a single insert, and re-read through the model to confirm what landed.
- **A new test that passes against the unfixed code is not a passing test.** It is
  passing through a branch you did not mean to exercise: an `OR` group whose
  sibling column also matches, a component that renders the value in a second
  place, or a default that already satisfies the assertion. Make every non-target
  field in the fixture non-matching so only the code under test can satisfy it, and
  pin values the factory draws at random — localized names collide with localized
  search terms often enough to matter. If a test still cannot be made to fail
  without the fix, it documents existing behaviour: label it as such or drop it
  rather than filing it as a regression guard.
- **`syncPermissions` replaces the role's whole set; it does not add to it.**
  Calling it with a partial list inside a permission seeder strips the role of
  every permission left out, so adding one new permission to a seeder can lock
  admins out of unrelated sections. Use `givePermissionTo` for a targeted grant
  and reserve `syncPermissions` for the seeder that deliberately owns the role's
  entire set.
- **Pest `--filter` needs the method name.** A humanized name containing spaces
  (`--filter "filter unit applies"`) prints `INFO No tests found.` and exits 0 —
  a silent no-op that reads as a green run. Use `--filter=test_filter_unit_applies`.
  When looping one test to measure a flake rate, match on a stable substring, not a
  column-aligned pattern: Pest pads its summary differently for passing and failing
  rows, so a fixed-width `grep` silently drops half the results.
- **PHPStan baseline churn is not an app error.** A baseline entry carries an
  `ignore.count`; your code starting to hit an ignored error shifts the count
  (`expected N, found N+1`), and code that stops hitting it becomes an *unmatched*
  entry. Fix the real typing errors your change introduced first, then regenerate
  the baseline once with the repo's baseline command (`composer phpstan-baseline`
  when present) and re-run to confirm zero errors. Regenerating to silence an error
  your own change added buries a regression instead of baselining legacy debt.
- **Static-analysis blind spots are style, not permission.** Without a package
  bridge, analyser config often hides model static calls, so `Model::find()` or
  `Model::firstWhere()` lints clean while `Model::query()->find()` is the typed,
  analyser-visible form — use the query-builder form when a baseline entry or a
  `Model::query()` return-type error points at it. Likewise annotate `$casts` with
  an explicit `array<string, string>` var tag when the analyser insists on `mixed`.
- **Component state inside Blade scope/slot closures must be read via `$this->`.**
  Inside `@scope(...)` / `<x-slot>` callbacks the closure captures view scope, not
  the Livewire component's public props, so a bare `$editingId` raises
  "Undefined variable" at render time. Always `$this->editingId`.
- **Attribute selectors must survive component whitespace.** Blade UI components pad
  rendered attributes (a placeholder can come out with a trailing space), so an
  exact `input[placeholder="…"]` match silently fails while the field is on screen.
  Match by substring — `input[placeholder*="…"]` — and use role+name queries with
  `exact: false` for labels.
- **Factory builders don't proxy `unique()`.** `$this->unique()` on a factory builder
  throws `BadMethodCallException`; call `fake()->unique()->…` in factory attributes
  instead.
- **A red browser run often skips its own teardown.** Wrapper scripts written with
  `set -e` exit at the first failing command, leaving a swapped `.env`, a stale dev
  server, or a borrowed port behind. Wrap the run in your own restore/cleanup
  (copy the original env back, kill the test server) so the developer's environment
  is intact whether the suite is green or red.
- **View/config caches hide fixes.** After touching routes, config, or Blade, a
  stale cached view or route can make a passing change fail with a bogus error —
  `php artisan view:clear` / `route:clear` / `config:clear` before re-running.

## Integration

- `test-driven-development` — the RED-GREEN rhythm this procedure sits inside.
- `systematic-debugging` — Phase 1 when a gate stays red after isolation.
- `requesting-code-review` — the pre-commit verification pipeline these gates feed.
- `read-the-damn-docs` — before asserting a package's own API (Livewire, MaryUI,
  Spatie, Playwright) rather than assuming it from memory.
- `references/red-test-attribution.md` — the ordered probes for deciding whether a
  red run is yours, self-inflicted contention, or a pre-existing infrastructure bug,
  plus how to read a failing CI job.