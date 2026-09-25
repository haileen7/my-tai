# Attributing a red test run

Use this when a gate is red and the question is *whose fault is it*. Three causes
look identical from the summary line: your diff, the environment, or a pre-existing
infrastructure bug. Work the probes in order — each one is cheap and rules something out.

## Probe 0: did you start more than one test process?

**Run this before reading any code.**

```bash
pgrep -af 'pest|artisan test'
```

Every framework-tier run in the project shares the single `DB_DATABASE` from
`phpunit.xml` and mutates it — no transaction rollback, no isolation. A second
concurrent run (a full suite in the background, an isolation loop, a worktree run)
fights the first one over schema and produces a **mass** failure count, often the
entire suite, with errors like `Deadlock detected` or `relation does not exist`
landed on tests that never touch your change.

Stop every process, then re-run once, alone, and only judge the result that comes back.

| What you did | What it causes |
|---|---|
| Full suite in background + an isolation loop | The isolation loop's `migrate:fresh` races the suite's per-test transaction setup |
| Full suite + a `git worktree` run | Two bootstraps, one database |
| Framework tier + browser tier | The browser harness swaps the env file and rebuilds the DB |

This is a self-inflicted failure that looks exactly like a catastrophic regression.
Recognising it by the failure *count* alone wastes an hour of bisecting your own diff.

## Probe 1: does the diff touch the failing file?

```bash
git diff --name-only <base-sha>..HEAD | grep -F '<FailingTest>'
```

No match means the test was already dependent on shared state. Then check whether
the file's fixture changed indirectly (a seeder, a factory, a cache namespace, a
permission list that other tests assert against).

## Probe 2: isolate, then measure the rate

```bash
php artisan test tests/Feature/FooTest.php --filter=test_specific_method
```

Run it several times. A failure that appears on some runs and not others is
order- or seed-dependent, not logic — the next probe is re-running with the seed
the runner printed, not forming a hypothesis about the source.

## Probe 3: prove it against the baseline

The decisive test for "is this pre-existing?". Build a throwaway checkout at the
upstream base commit and reproduce there — the working tree stays untouched.

```bash
git worktree add -q /tmp/base <upstream-base-sha>
cd /tmp/base
ln -s /path/to/repo/vendor vendor      # reuse the installed dependencies
cp /path/to/repo/.env .env             # same DB credentials the failing run used
php /path/to/repo/vendor/bin/pest tests/Feature/FooTest.php --filter=test_specific_method
git worktree remove --force /tmp/base   # always clean this up
```

If the base reproduces the failure, the bug predates the work. Say so explicitly in
the report, name the failure, and stop bisecting your own change — offer the
infrastructure fix as separate work rather than folding it into the current diff.

Symlinking `vendor` and copying `.env` is what makes this cheap: no second install,
no second database, and the base run is otherwise identical to the failing one.

## Reading the failure out of CI

`gh run view --log-failed` prints ANSI colour codes inline, so grepping it returns
mangled lines. Strip them first:

```bash
gh run view <run-id> --repo <owner>/<repo> --job <job-id> --log-failed 2>/dev/null \
  | sed 's/\x1b\[[0-9;]*m//g' > /tmp/ci-fail.log
grep -nE 'FAILED|Tests: ' /tmp/ci-fail.log
```

Prefer the specific `--job` over the whole run: the run log bundles the matrix,
static analysis and mutation stages together, and the failing test is often buried
behind hundreds of lines of unrelated output.

Note also that a job's logs are only complete once the whole run finishes — if a
later stage (mutation testing) is still going, `--log-failed` returns nothing useful.
Poll the run status rather than assuming no checks were registered.

## The shared-database parallel deadlock

A recurring infrastructure failure in Laravel + Pest projects that run `--parallel`:

```
SQLSTATE[40P01]: Deadlock detected
  ... drop table "..." cascade
```

`--parallel` forks N processes that all point at the **same** test database. Each
one's `migrate:fresh` / drop-table takes conflicting locks, and Postgres resolves
the deadlock by killing whichever transaction lost.

Fixes, in order of preference:

1. **Per-process database name.** Have each parallel worker use a suffixed
   database so the workers never share schema. This keeps parallelism, which is
   usually the point of running it in CI.
2. **Drop `--parallel`** for the affected job. Slower, but stable.
3. **Retry once on deadlock.** Treats the symptom; acceptable only as a
   short-term bridge, and it hides real contention if it becomes routine.

Adding a heavy seeder call to many `beforeEach` blocks widens the window in which
this can happen, so a fix that works today can start failing tomorrow as fixtures
grow. Prefer fixing the database isolation over trimming the fixture.
