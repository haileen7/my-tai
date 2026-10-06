Every new session: default cwd is /home/runner/h-dashboard, and always use CodeGraph (`codegraph sync` first; codegraph_explore for code Q&A) + superpowers skills + read-the-damn-docs (web_search official docs) before acting; shadcn/improve for h-dashboard audits only on request.
§
Boost MCP occasionally dies on first stdio call ("lost its stdio subprocess") — just call it again. CLI fallback always works: php scripts/boost_tool.php <tool> '<json>'.
§
h-dashboard: origin=haileen7 fork; baran tracks origin/beta. Canonical beta has no remote — sync via `git fetch https://github.com/asgarimehdi/h-dashboard.git beta:refs/remotes/upstream/beta`.
§
MaryUI x-select defaults to optionValue='id'/optionLabel='name'. Options keyed 'value'/'label' need explicit option-value="value" option-label="label" or every <option> renders empty (blank control). Pass :options="$this->myOptions()" from a component method — a bare $myOptions is undefined in the Blade view.
§
scripts/e2e-test.sh: not concurrency-safe, never run two; swaps .env and .env.dev.bak may hold ALREADY-SWAPPED content — verify `grep DB_DATABASE .env` == h_dashboard after every run. Kill orphans via `kill $(pgrep -f 'artisan serve --port=800[1]')`, never pkill -f 'artisan serve'.
§
.env is gitignored; rebuild from `.env-example-github` + secrets in `.env.e2e`, override APP_URL=http://127.0.0.1:8000 and DB_DATABASE=h_dashboard, drop `secrets.` lines, verify `php artisan about --only=environment`. parse_ini_file('.env') fails (unquoted parens) — regex scan or config() instead.
§
Map perf fixed (adc561f): bottleneck was main-thread rendering (/map pan 620→52ms). Fix: circleMarker+lazy popup, icon cache, memo depth, canvas lines, dead loadStats removed.
§
Cron blocked by scanner: a SKILL.md quoting a literal classic injection example phrase trips tools/cronjob_prompt_scan._CRON_SKILL_ASSEMBLED, so EVERY cron job using that skill fails regardless of prompt. Fix = reword the skill example, not the prompt. Persian cron prompts must avoid U+200C ZWNJ. Pre-test with _scan_cron_prompt / _scan_cron_skill_assembled in ~/.hermes/hermes-agent.