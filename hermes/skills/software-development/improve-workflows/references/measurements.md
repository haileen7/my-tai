# Measuring a Repo Before Arguing About Its Cost

Depth for the speed / stability rows of the claim checklist in `plan-review.md`.
A number quoted in a review is a claim; take it from a run, not from a habit.

## Time the command the pipeline runs

The scoped variant is not a cheaper form of the full one, it is a different
command, and its timing says nothing about the job in CI:

| What you time | What it actually covers |
|---|---|
| `vendor/bin/lint --dirty` | only files modified in the worktree |
| `vendor/bin/lint --test` | the whole repo — this is the form a CI lint job runs |
| `vendor/bin/analyse` with a warm result cache | one file changed, rest read from cache |
| same, after clearing the result cache | the cost a fresh clone or CI runner pays |

So: run the full command for the headline number, and if you want to argue that
a narrower variant is worth it, give the cold number too. A multi-second
analyzer with a warm cache and a two-minute one cold are both real; publishing
only the warm one to make a gate look expensive is the defect the reviewer is
supposed to catch in the other direction.

```
/usr/bin/time -f "cold: %e s" vendor/bin/phpstan clear-result-cache
/usr/bin/time -f "cold: %e s" vendor/bin/phpstan analyse --no-progress
/usr/bin/time -f "warm: %e s" vendor/bin/phpstan analyse --no-progress
```

## Read what CI really executes

Grep the workflow for the command, not for the tool name — the tool name appears
in many steps with different flags:

```
grep -nE 'run:|pest|phpstan|pint|coverage|min=|view:clear|config:clear' .github/workflows/*.yml
```

A CI job often runs lint, tests and static analysis as SEPARATE jobs with
separate commands and separate flags. "Reproduces CI" then means picking one
job, and the honest acceptance criterion is "reproduces the tests job" — the
coverage gate and the env-rewrite step (copying an env file and rewriting DB
credentials to match the service containers) are usually not reproducible on a
developer box.

## Reproduce a failing suite before believing either verdict

A single green run is not a stability verdict, and a single red run is not a
flake verdict. When a full suite fails, capture the printed random-order seed
and replay that exact order — an order-dependent failure reproduces, a genuine
flake usually does not:

```
grep -E 'Random Order Seed|Tests:' /tmp/suite.log
php artisan test --order-by=random --random-order-seed=<seed>
```

Then run the whole suite again unchanged. Report what both runs did. If one
order failed and another passed, that is itself the finding: the plan cannot
claim the suite is deterministic, and the honest recommendation is to run it
once in the order CI uses rather than to hand-write an exclusion list.

Full-suite runs take minutes: start them with a background process and
`notify_on_complete`, keep working, and read the log file afterwards. Never sit
in a poll loop waiting for them.
