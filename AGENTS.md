# AGENTS.md

## What this is

A single-file web app (`index.html`, ~2900 lines, all HTML/CSS/JS inline) for the
Kindle Paperwhite's e-ink browser. No build system, no package.json, no tests, no
lint. Deployed via GitHub Pages (`main` branch, `CNAME` = kindlenoodle.com).

## Verify changes

```sh
python3 -m http.server 8000   # serve from repo root, then open http://localhost:8000
```

- Do NOT open `index.html` via `file://` — the app XHR-loads `./data/daily.json` and that fails on file://.
- To simulate a Paperwhite 2, size the browser window to 758×1024.

## Target runtime: old Kindle WebKit — follow existing code style

- JS is ES5-style on purpose: `var`, function declarations, `XMLHttpRequest`. No `fetch`, arrow fns, modules, or template literals.
- CSS needs `-webkit-` prefixes (flexbox etc.) — copy the pattern from adjacent rules.
- All external calls (ip-api.com timezone, v1.hitokoto.cn quotes) must fail silently with fallbacks; Kindles are often offline. Keep that.

## E-ink invariants (do not break)

- CSS animations/transitions are globally disabled with `!important` at the top of the `<style>` block — never re-enable.
- Every screen change must go through `forceEinkRefresh()` (index.html:1577, black/white flash full refresh) or e-ink ghosting appears.
- Borders/lines instead of box shadows (ghosting).
- Layout is fixed 758px wide, scaled to the viewport via a `transform: scale()` on `<body>` (index.html:2926) — do not make it fluid/responsive.
- `<meta http-equiv="refresh" content="43200">` (12h auto-reload) is intentional.
- The clock uses UTC getters + `tzOffsetMs` (default UTC+8, updated from ip-api.com). Don't "fix" it to local-time getters.

## App structure

Tabs are `#page-0`…`#page-5` in one file: 0 = clock (flip/module styles), 1 = pomodoro, 2 = noodle timer, 3 = reading (opens weread.qq.com), 4 = TODO list (bottom-nav tab 4, see below), 5 = AI daily report (fetches `data/daily.json`, paginates into e-ink pages). Tab index always equals the page id (`switchTab(n)` shows `#page-n`); localStorage keys: `kindle_ob_done`, `kindle_noodle_clock_style`, `pomo_daily`, `ai_daily_read_*`.

## Data pipeline (AI daily report)

- The app reads only `data/daily.json` (schema: `{date, attribution, lead, sections:[{label, items:[{title, summary, sourceUrl, sourceName, permalink, attribution}]}], flashes}`).
- `.github/workflows/fetch-aihot.yml` is `workflow_dispatch` only — it is triggered daily (08:15 Beijing) by an external cron-job.org, not by GitHub's own cron. Don't add a `schedule:` trigger.
- The workflow's curl flags (`--speed-time/--speed-limit/--retry…`) and the non-browser User-Agent are deliberate workarounds for AI HOT's tarpit edge protection — don't simplify or remove them.
- The workflow commits `data/` as `github-actions[bot]` ("chore: update AI HOT daily …").
- `data/archive/YYYY-MM-DD.json` and `data/dailies-index.json` are also fetched/committed but **not used by index.html yet** (staged for a future past-issues feature) — don't delete them.
- All data JSON is single-line (no trailing newline), so `wc -l` reporting 0 is normal, not a broken file.

## TODO list tab (bottom-nav tab 4, page-4)

- Sources live in `data/todo-sources.json` (single-line array): `[{repo, branch, file, label?}]` — one entry per md file; `label` is the group header (falls back to repo name); order = display order. Adding a source = adding one entry, no code changes.
- The app fetches sources directly at runtime: same-origin XHR for the config, then one XHR per source to `raw.githubusercontent.com/<repo>/<branch>/<file>` (path segments URL-encoded; raw sends `Access-Control-Allow-Origin: *`). Fetch fires on every tab entry; no caching. Per-source failure shows a tappable retry row instead of killing the whole list.
- The CI-pipeline pattern (like fetch-aihot) is the agreed FALLBACK if direct fetch proves unreliable on real Kindles — not implemented; don't add it preemptively.
- Parser (`todoExtract`) scans only the first level-1 `# TODO` heading up to the next level-1 `#` heading (`##` inside stays in scope). It emits every unchecked `- [ ]` task plus its nearest enclosing TASK line as parent context (checked parents included, rendered as plain text); plain note lines are skipped when resolving parents and never displayed.
- `NAV_LEFT` has one entry per bottom-nav icon (5 entries; the AI-daily icon lives in the separate right card). Adding another tab means recomputing those positions (bar centers under each 56px icon) and bumping the `page-0..5` loops in `switchTab`/`enterDailyMode`/`exitDailyMode`.
