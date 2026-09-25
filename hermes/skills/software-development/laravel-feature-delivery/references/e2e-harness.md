# Running the Playwright suite in an isolated Laravel app

Pattern for any Laravel repo whose e2e suite runs against a throwaway database
and a locally-served app.

## First-time setup

1. Build the e2e env file from the checked-in example plus the values your app
   already needs, without echoing secrets:

   ```bash
   cp .env.e2e.example .env.e2e
   get() { grep -E "^$1=" .env | head -1 | cut -d= -f2-; }
   setv() { key="$1"; val="$2"; [ -n "$val" ] && sed -i "s|^$key=.*|$key=$val|" .env.e2e; }
   setv APP_KEY "$(get APP_KEY)"
   setv DB_USERNAME "$(get DB_USERNAME)"
   setv DB_PASSWORD "$(get DB_PASSWORD)"
   setv REDIS_PASSWORD "$(get REDIS_PASSWORD)"
   ```

2. Confirm the locale line the specs depend on is present (assertions on Persian
   pagination and validation text fail without it), then verify only that the
   value is set, never print it.

3. Create the throwaway database from the PostGIS template if the app uses it:

   ```bash
   DBU=$(grep -E '^DB_USERNAME=' .env | cut -d= -f2-)
   export PGPASSWORD="$(grep -E '^DB_PASSWORD=' .env | cut -d= -f2-)"
   psql -h 127.0.0.1 -U "$DBU" -d postgres \
     -c "CREATE DATABASE <e2e_db> WITH OWNER=$DBU TEMPLATE=template_postgis;"
   ```

4. Install the browser once: `npx playwright install chromium`.

Never point the e2e database at a development or test database — the harness
runs `migrate:fresh --seed`.

## The restore trap

A harness that swaps `.env` with `set -e` and no `trap` exits before restoring
when a test fails, leaving the repo on the e2e config with a stray server
running. Chain the cleanup onto your own command instead of trusting the script:

```bash
bash scripts/e2e-test.sh tests/e2e/<area> > /tmp/e2e.log 2>&1; RC=$?
[ -f .env.dev.bak ] && cp .env.dev.bak .env && rm -f .env.dev.bak
pgrep -f 'artisan serve --port=800' | xargs -r kill
echo "E2E_EXIT=$RC"; grep -E 'passed|failed' /tmp/e2e.log | tail -4
```

Also verify the backup file is actually gitignored — a leftover `.env.dev.bak` is
an untracked secret sitting in the working tree.

## Read failures from the artifacts

Playwright writes `test-results/<slug>/error-context.md` containing a YAML DOM
snapshot at the point of failure, plus a screenshot and a video. Read the YAML
before re-running anything: it usually reveals the selector problem immediately.

## Reporting during long runs

A full Playwright run and a full Pest run each take minutes. Start them in the
background, keep working on anything not dependent on the result, and report
progress on a short interval rather than going silent — the user should never
have to ask what is happening. Report the real numbers, including counts of
pre-existing risky/skipped tests, and distinguish `N passed` from `0 failed`.
