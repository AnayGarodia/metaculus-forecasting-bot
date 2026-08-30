# What actually wins AI forecasting tournaments — and what this bot implements

Sources: Metaculus AI Benchmark (AIB) quarterly result writeups; "Diversity is the
Strength of the AI Crowd" (arXiv 2606.29661); Halawi et al., "Approaching
Human-Level Forecasting with Language Models" (NeurIPS 2024, arXiv 2402.18563);
ForecastBench (arXiv 2409.19839); Samotsvety / normative-aggregation practice.

## The findings that matter

1. **The base model dominates scaffolding.** Across AIB, the single biggest lever
   is the underlying LLM, not the pipeline. Top bots that shared one prompt ranked
   by which model they used. → We run the strongest model available to us
   (Azure Foundry `gpt-5.6-sol`, a reasoning model) for every role.

2. **Ensemble of N independent forecasts, then aggregate.** The winning recipe is
   deliberately simple: one research pass, one good prompt, forecast ~5 times,
   submit an aggregate. → `predictions_per_research_report=5`.

3. **Aggregate with geometric-mean-of-odds, not the mean or median.** AIB analysis
   and the normative-aggregation literature find geometric mean of odds (equiv.
   mean of log-odds) beats median and trimmed mean for Brier score. → Binary uses
   trimmed geometric-mean-of-odds; multiple-choice uses per-option
   geometric-mean-of-odds, renormalized. Numeric/date keep the framework's robust
   median-of-CDFs.

4. **Extremizing helps only when members are independent.** Averaging regresses
   toward 50%; extremizing (scaling pooled log-odds by a factor >1) undoes that —
   BUT only when forecasters are decorrelated. Our members share one model and one
   research brief, so they are correlated; aggressive extremizing would just
   manufacture false confidence. → We apply a *mild* fixed factor (`_EXTREMIZE =
   1.15`) to the pooled binary log-odds, clamped. Set to 1.0 to disable.

5. **Decorrelate the ensemble members.** "Diversity is the strength of the crowd":
   low-correlation members improve the aggregate more than better individual
   members. We can't cheaply add other models, so we decorrelate via prompt: each
   of the 5 samples gets a random framing (outside-view-first / inside-view /
   status-quo-skeptic / neutral).

6. **Structured superforecaster reasoning beats a bare ask.** Halawi et al. and
   Metaculus's own template use an explicit scratchpad: question → resolution
   criteria → retrieved evidence → reference-class base rate → status quo →
   evidence for/against → adjustment → calibrated final. → All four question types
   (binary/MC/numeric/date) use this ordered structure.

7. **Retrieval matters; a single good search is enough.** Top bots do one internet
   search feeding the prompt. We go a little further: an LLM decomposes each
   question into 3 targeted queries (latest news / base rate / resolution source),
   we fetch and strip the top pages, synthesize an evidence brief with dates,
   numbers, and source URLs, and cache it for 6h. Keyless (DuckDuckGo via `ddgs`),
   so no API dependency. Degrades to LLM-only if search fails.

8. **LLMs are systematically overconfident; nudge toward the base rate.** Binary
   AIB questions resolve Yes only ~35% of the time. → The prompt has an explicit
   calibration step pulling estimates toward that prior and toward 50%, and never
   states 0%/100%. This operates per-forecast (fixing model overconfidence),
   independently of the pool-level extremize in (4).

## What `forecasting-tools` already provides (so we didn't rebuild it)

- The `ForecastBot` loop: load questions, run research ×R, forecast ×(R·P),
  aggregate, publish, skip-already-forecasted. We override only research, the four
  per-type prompts, and aggregation.
- `structure_output` — LLM-parses freeform reasoning into typed predictions
  (BinaryPrediction / PredictedOptionList / list[Percentile]) with validation
  samples. We keep it; it removes brittle regex parsing.
- Numeric percentile handling: `NumericDistribution.from_question` enforces bounds
  and ordering; base-class aggregation takes the median CDF. We keep both.
- Robustness: `return_exceptions=True` at the tournament level means one bad
  question can't crash the run; our aggregation overrides fall back to the base
  class on any error.

## Gaps / possible future levers (not done — cost/complexity vs. payoff)

- True multi-model ensemble (add one cheap decorrelated model) — the single
  highest-value remaining lever per the diversity result, gated on a 2nd API key.
- Tuning `_EXTREMIZE` and the trim against a resolved-question backtest set.
- More research passes (`research_reports_per_question>1`) — more cost, marginal.
