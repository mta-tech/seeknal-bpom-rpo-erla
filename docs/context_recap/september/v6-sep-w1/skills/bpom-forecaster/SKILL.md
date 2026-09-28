---
name: bpom-forecaster
description: "Forecasting skill for predicting future registration trends. Computes projections deterministically from historical data with quality labels."
tags: [bpom, forecast]
version: "6.2.1"
---

# BPOM Forecaster (Trigger)

Routes BPOM forecast requests to `run_forecast`. This skill owns only the BPOM CAPTURE
parameters — all computation is deterministic (an ETS seasonal fit inside the IBA forecast
engine). The LLM does **no** forecast arithmetic: the SQL shapes the series, and the engine
produces the numbers.

Load `context/forecast_guide.md` first — it holds the data source, the SQL template, the series
registry, the quality thresholds, and the output rules. This file covers only the tool workflow.

## CAPTURE
Lock `{sql, periods}`.

- This repository is ERLA-only, so every series is an ERLA series; when the series is not stated, it is Permohonan Total.
- **Horizon translation:** `periods` is a step count on the SQL's own grain —
  monthly: 6 bulan → 6 · 1 tahun → 12 · 3 tahun → 36 · 5 tahun → 60 → capped.
  The cap is 36; the tool clamps silently, so when the request exceeds it, say in the answer
  that 36 months is the maximum supported horizon and present those.
- **"hingga/sampai {X}"** = every step from the period after the last actual **through the end
  of X** — the intermediate periods are part of the request, not just the named year/month.
  Always pass `periods` explicitly; the tool defaults to 3 when it is omitted.
- **"bulan depan" / "next month"** = the next **COMPLETE** month — the month AFTER the current
  running month, not the first forecast step. The forecast's first step is the running month
  (its data is not complete yet); "bulan depan" is the step after that. For example, with data
  complete through June and July still running, the first step is Juli and **"bulan depan" =
  Agustus**. Use `periods ≥ 2` and name that complete month (Agustus) as the answer's target;
  the running month (Juli) is a bridge step shown for context, never the headline.
- This repository is **ERLA-only**, so every available series is an ERLA series — drawn from
  `t_produk_3_rilis_erla` (products) and `t_btp_3_erla` (BTP). There is no second system and no
  cross-system UNION; the UNION topology reference in `data_architecture.md` does not apply
  here.
- Build the SQL using `forecast_guide.md` §1's template exactly (table + filter from §3's
  registry). If the **series** is ambiguous — the system itself is always ERLA — call
  `request_clarification` first.

## RUN
Call `run_forecast(sql, periods)`. Do not compute forecast numbers yourself. `execute_python` is not a forecast instrument — never use it to compute, extend, or sanity-check projections; the engine output is the single source (v6).

- **After a clarification is answered** (the user picks a scope): call `run_forecast` fresh this
  turn for the resolved scope. The forecast numbers in the answer must come from a
  `run_forecast` result produced **this turn** — never restate or quote numbers from a previous
  turn's answer or the conversation history (LLM recall drifts, and the answer would no longer
  match the tool's deterministic output and the downloaded CSV). If you already ran the forecast
  before the scope was resolved, run it again for the chosen scope; the tool is the single
  source of the numbers.

- `## Kesalahan` with "policy check (STEP 0.5)" → the SQL contains a JOIN /
  `generate_series` / recursive CTE — rebuild it flat per the template; do not retry the same
  pattern.
- `## Kesalahan` with "STEP 1: EXECUTE" → a runtime error — recheck against
  `forecast_guide.md` §1/§3 and retry corrected.
- `## Ditolak` → present the engine's reason (usually insufficient history) and offer the
  historical trend instead. Do not retry unless the request changes — and do not compute a
  replacement estimate yourself; a refusal has no numbers.

## PRESENT
Read the tool's markdown and compose prose around it — do not invent a new structure. Lead with
the quality label. Follow `forecast_guide.md` §5 for vocabulary (never show raw `sigma`,
`sub_type`, field names, or the CV number).

**Present the full computed horizon.** Every predicted period the tool returned appears in the
answer — never truncated to the first months or to the named year only. A "hingga {X}" answer
runs from the first predicted period through the end of X (add per-year subtotals when it spans
years).

**The Answer Contract applies to forecasts too (transparency, general).**
Every projected period is its own row (point + Rentang Realistis); the tool's history block is
presented in full alongside the projection, never dropped; multiple series → per-series labelled
sections (code + dictionary description per series). The stored CSV (tool-owned, combined)
covers exactly the horizon presented: all historical periods (`kind=historis`) AND all projected
periods (`kind=proyeksi-*`) in one file. When the request exceeds the 36-step cap, both the
answer and the export presentation state "36 bulan (maksimum yang didukung)" — never silently
deliver less than asked.

**Anomaly:** if the tool's markdown has an `## Anomali` block, include it and state that the
points were **not removed**. If the user asks about anomalies directly, call
`detect_anomaly(sql)` yourself (it works whether or not a forecast ran this turn) — see
`forecast_guide.md` §6.

**CSV (Store Contract — one combined store per question):** on a successful forecast,
`run_forecast` self-uploads ONE combined CSV — historical + projection together (columns
`period, kind, value, point, lower_80, upper_80, lower_95, upper_95`; `kind` = `historis` |
`proyeksi-…`) — tool-owned and already matching the answer. **Do not call `upload_to_s3`
yourself on a successful forecast** — there is no separate historical export; it is already
inside the combined file. Only when the forecast was refused or failed (`## Ditolak`, so no
combined CSV exists) and you fall back to a historical-trend answer may you export the history
via `upload_to_s3(filename="<series>_historis.csv", sql=<the history SQL, no LIMIT>)`.
Multi-series (each its own `run_forecast`) → each call self-uploads its own combined CSV. Never
`data=`/`columns=` — numbers you type are not evidence. Never paste the raw URL; the Download
button renders automatically.

## Stock vs Flow

| Question pattern | Kind | Y expression |
|---|---|---|
| "NIE **baru**/**terbit**/permohonan per bulan" | flow | `COUNT(DISTINCT produk_id)` |
| "NIE **aktif**/**terdaftar**/total sekarang" | stock | `SUM(COUNT(DISTINCT produk_id)) OVER (ORDER BY date_trunc(...))` |

For stock queries, the fit may show some upward drift but does not reliably track a real
cumulative total's growth rate — say so. A genuinely reliable stock projection needs a separate,
deliberate arithmetic step (last known stock + flow × periods), not `run_forecast` alone.

## Hard rules

- **Never use `execute_python` for forecast arithmetic.** LLM-generated Python varies per turn
  and produces inconsistent numbers for the same question. `run_forecast` is the single
  deterministic source.
- **Follow-up consistency:** if a follow-up asks about the same series (different horizon,
  clarifying question), reuse the exact SQL from the prior turn — do not rebuild it from
  scratch. Only rebuild for an explicitly different series/grain/filter. Read the follow-up
  against the earlier turns: carry over the settled series and horizon, and change only
  what the user names.
- **If the projection ran but its chart did not render**, the numbers still stand: present the
  full projection in words and note the chart could not be shown — never re-run `run_forecast`
  just to force the visual.
- **Same question → same SQL:** resolve the series from `forecast_guide.md` §3 exactly (same
  wording → same registry row, no improvised filters) and always keep the template's
  current-month cutoff — a slightly different history SQL still shifts the projection, so build
  it verbatim (the engine's window is fixed at 36 months, so the SQL is the only remaining
  variance).
- **Consistency contract:** the same question (same series, grain, horizon) must produce the
  same numbers in any session and in follow-ups — the engine is deterministic, so any difference
  means you built a different SQL or `periods`. Resolve from the registry verbatim, pass
  `periods` explicitly, and for follow-ups reuse the prior turn's exact SQL. The only legitimate
  difference is data drift — stamp the as-of date.
