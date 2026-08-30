# Metaculus AI Benchmark bot — build log

Fresh, independent build. Nothing reused from the `oracle` project.

## 2026-08-30 — Contender upgrade (Opus session). READ THIS FIRST.

The LIVE deployment is **GitHub Actions**, not the Azure cron. Workflow
`.github/workflows/run_bot_on_tournament.yaml` runs `python main.py` (default
tournament mode = seasonal 33022 + minibench) every 20 min at :07/:27/:47, using
repo secrets (AZURE_API_KEY/BASE/VERSION, METACULUS_TOKEN). The Azure-cron notes
below are historical/secondary — that box runs at :00/:20/:40 and, if still live,
first-forecasts questions with stale code before Actions fires (see "cron race").

**What shipped this session (main.py):** the prior run's improvements had been
left UNCOMMITTED — the live bot was still the plain template. Committed + pushed
them, plus two additions:
- Binary aggregation: trimmed geometric-mean-of-odds + mild extremize (`_EXTREMIZE
  = 1.15`, set 1.0 to disable). Rationale in RESEARCH.md #4.
- Multiple-choice aggregation: per-option geometric-mean-of-odds, clamped off 0/1
  (the MC parse prompt deliberately emits 0% options → would blow up log-odds) and
  renormalized. Falls back to base-class mean on any inconsistency/error.
- Already present from prior run: 5-forecast ensemble, decorrelating framings,
  multi-query web research + synthesis brief, structured superforecaster prompts,
  calibration nudge. See RESEARCH.md for the full technique list + sources.

**Test — PASSED.** `python main.py --mode test_questions` on bot-testing-area:
all 9 questions forecasted, 0 errors / 0 minor errors, binary+MC(new GMO)+numeric
all posted to Metaculus. NOTE: must run under **Python 3.11** (CI version). The
local `.venv` is Python 3.14, which hits a litellm/anyio bug ("unknown async
library, or not in async context") on every LLM call — created `.venv311/`
(gitignored) for testing. Also: this shell exports STALE `AZURE_API_KEY`/`_BASE`
that override `.env` (dotenv doesn't override existing env) — run tests with
`env -u AZURE_API_KEY -u AZURE_API_BASE -u AZURE_API_VERSION` so `.env`'s good key
(cognitiveservices host) wins. Direct curl to the deployment returns 200; the
`.env` key is valid.

**cron race — needs USER.** If the honcho-azure cron is still active it forecasts
first with old code every window (skip-already-forecasted makes first-poster win).
Fix: `ssh honcho-azure` then `crontab -r` (or `git pull` in `~/metaculus-bot`).
SSH may fail on NSG allowlist if your IP rotated.

## Status: LIVE on Azure (honcho-azure), auto-forecasting via cron.

Pipeline validated end-to-end (dry run posted real forecasts, grounded in live keyless web search). The live
tournament simply has no open questions at this moment; the recurring loop
picks up new ones automatically.

## Model — Azure Foundry gpt-5.6-sol (working string: `azure/gpt-5.6-sol`)

- litellm `azure/gpt-5.6-sol`, env AZURE_API_KEY / AZURE_API_BASE / AZURE_API_VERSION.
- **Correction 1 — the AZURE_API_KEY given to me was invalid** (401 on every
  host + path shape). Replaced with `key1` from
  `az cognitiveservices account keys list -n aztea-foundry -g rg-aztea-fleet`.
  (The bad key may have belonged to the other account `aztea-aoai-sponsor`.)
- **Correction 2 — real endpoint host** is `aztea-foundry.cognitiveservices.azure.com`,
  not `services.ai.azure.com` (both actually work with the right key; standardized
  on cognitiveservices). Deployment `gpt-5.6-sol` (model gpt-5.6-sol 2026-07-09) confirmed present.
- gpt-5.6-sol is a **reasoning model**: it rejects `temperature != default`.
  Fixed by dropping the explicit temperature and setting `litellm.drop_params = True`
  in main.py (auto-strips temperature/top_p on every internal path too).
- OpenRouter removed entirely. Same Azure deployment used for default/summarizer/parser.
- The startup banner "No LLM key set (OPENROUTER/OPENAI/ANTHROPIC)" is **cosmetic** —
  check_environment doesn't know about AZURE; the bot uses gpt-5.6 fine (dry run proved it).

## Tournaments — LIVE right now

- **Correction 3 — the live seasonal tournament is Summer 2026 FutureEval**
  (id **33022**, slug summer-futureeval-2026, closes **2026-11-05**), NOT "Fall".
  API + SDK both confirm; `CURRENT_AI_COMPETITION_ID` resolves to 33022.
- MiniBench: `CURRENT_MINIBENCH_ID = "minibench"` (rolling ~2-week rounds).
- Also open: Market Pulse 26Q3 (33066, closes 2026-10-01) and Metaculus Cup
  Summer 2026 (33021) — not currently targeted; add if wanted.
- **No open questions in 33022 or minibench at this moment** — its latest batch
  is all `closed`/`resolved`. This is the normal between-batch gap; the cron loop
  will forecast the next batch the instant it opens. (bot-testing-area has open
  questions, which is how the dry run validated a real submit.)

## Dry run — PASSED (posted real forecasts)

`python main.py --mode test_questions` on bot-testing-area posted predictions to
questions **43329, 43330, 43331, 43332** (+ reasoning comments). gpt-5.6 produced
calibrated multi-option/binary/numeric forecasts. Then stopped (smoke test done).

## Live deployment

- Box: **honcho-azure** (172.177.132.171, eastus2), `~/metaculus-bot`, venv `.venv`.
- Recurring loop (installed, cron active):
  `*/20 * * * * cd ~/metaculus-bot && flock -n /tmp/metac-bot.lock .venv/bin/python -u main.py --mode tournament >> bot.log 2>&1`
  - flock prevents overlap (a run can exceed 20 min); `cd` so dotenv finds `.env`;
    `skip_previously_forecasted_questions=True` makes re-runs idempotent.
- Verified one cron-equivalent run: exit 0, "No new questions to forecast on this run."
- Logs on box: `bot.log` (cron), `dryrun.log`, `live.log`.

## What the bot does

Metaculus's official template (`forecasting-tools`, the harness Metaculus
benchmarks its own bots on). Per open question: research → reason with gpt-5.6 →
5 predictions aggregated → submit, skipping already-forecast questions.

## Open items — need the USER (I can't)

1. **Bot-maker survey — REQUIRED for any payout** (winner or not). Linked from the
   participate/resources pages: https://www.metaculus.com/futureeval/participate/
   and the resources notebook https://www.metaculus.com/notebooks/38928/ .
   User must fill it and allow Metaculus to inspect the bot code.
2. ~~AskNews creds~~ — RESOLVED. Bot now grounds forecasts in **keyless live web
   search** (DuckDuckGo via `ddgs`, no API key): `keyless_web_search()` +
   `_web_grounded_research()` in main.py. Per question it searches title+resolution
   criteria, pulls ~10 results, fetches top-3 page texts (requests + tag-strip),
   feeds them to gpt-5.6-sol, and appends a "Retrieved sources" URL list. Fails soft
   to LLM-only per question (try/except). 6h per-question cache (`.research_cache.json`)
   so cron re-runs don't re-search. Verified: dry run posted 9 forecasts, all citing
   retrieved sources (e.g. q43327 cited EU portal + Wikipedia member-list, "27 member
   states", dated Aug 29 2026).

## Nice-to-have

- Add Market Pulse 26Q3 (33066) as a second target if you want that ~$7k pool.
- The `.env` on the box holds the working Azure key + Metaculus token (gitignored, not committed).
