# Time & Periods

tahun, bulan, tren, terbit, kedaluwarsa, masa berlaku, sejak, sampai, per tahun.

## Four date columns — the panel's event bases, choose from what is being asked

| Panel basis / column | Answers |
|---|---|
| `tanggal` | **NIE issuance** ("Tanggal Terbit NIE") — "izin edarnya terbit", "NIE tahun X", **pendaftaran baru** |
| `tanggal_bayar` | **approval / payment** ("Tanggal Bayar SPB") — "disetujui", "persetujuan", "bayar SPB" |
| `tanggal_aju` | **submission** ("Tanggal Permohonan") — "diajukan", "pengajuan masuk", "permohonan registrasi" |
| `tanggal_exp` | **kedaluwarsa** — a condition about validity ending |

`Tanggal Terbit SPB` exists as a panel label but has **no warehouse column** — say so and offer
`tanggal_bayar` or `tanggal`, labelled (`05-filter-katalog.md`). Entity, date, and status come from
ONE row of the decision table (`00-menghitung.md` §1): `tanggal_aju` pairs with `produk_id` and no
status filter; `tanggal_bayar` and `tanggal` pair with `nomor` and the valid tier.

`tanggal_berkas` and `tanggal_diambil` are processing dates — **never** for counting. Using
`tanggal_aju`/`tanggal_bayar` for an "terbit" question shifts the result and mixes in applications that never issued. **There is no "market release date" column** — the closest is `tanggal`; state that limitation honestly rather than inventing a column.

**Fill rates are not equal.** The issuance column is filled only for rows that actually issued, so filtering on it silently drops unfinished applications. When the scope matters, check first:
`COUNT(*) FILTER (WHERE NULLIF(<col>,'') IS NOT NULL)` against `COUNT(*)`.

## Period query shape

- **Bounded ranges**, not `EXTRACT(YEAR …)` — `EXTRACT` forces a full-table transfer and belongs only in labelling grouped results.
- Cast on the ERBA side only: `NULLIF(tanggal,'')::timestamp` (`00-menghitung.md` §4).
- Trends: `date_trunc('year'|'month', …)` with ONE `GROUP BY` shaped like the answer — do not assemble the table from separate queries.
- **Periods with no rows vanish from the result**, and on a chart the line draws straight across the gap. The gap-filling rules (conclude the grain first — yearly is always Jan 1; zeros only for flow metrics) live in `skills/visualize-chart` §"A time series must be dense in its own calendar".
- **The headline still comes from its own global query.** `GROUP BY period` then summing partitions double-counts `nomor` values appearing in more than one period.
- **A month named without a year means ALL years.** "permohonan Mei" aggregates every May in the
  data (`EXTRACT(MONTH …)=5`, labelled) — it does not mean May of the running year. Adding a
  year the question never named silently drops most of the population; if the user might mean
  the current year, that is a Gate 1 clarification.

### Three recurring shapes — semester, rolling window, month-without-year

- **Semester 1 vs semester 2**: split on `EXTRACT(MONTH FROM <date>) <= 6` (S1) vs `> 6` (S2) —
  one query, two `FILTER` counts on the same entity. When the year is still running, the later
  semester is partial: label it "YTD per {date}" and never compare it to a complete semester as
  if both were whole.
- **Last 12 months** (rolling): bound by
  `>= date_trunc('month', CURRENT_DATE) - INTERVAL '11 months' AND < date_trunc('month', CURRENT_DATE)`
  for 12 *complete* months, or extend the upper bound to `CURRENT_DATE` when the running month
  should appear (then mark it in-progress). Anchor the window on the panel's chosen date column —
  never on a hardcoded list of months.
- **"Per bulan untuk 12 bulan terakhir"** = the rolling window + `date_trunc('month', …)`; the
  dense-series fill rules in `skills/visualize-chart` still apply to gaps.

## Time-flavoured answers — break them down, never one bare number

Questions mentioning **tahun / periode / tren / "rentang 2026"** get a layered answer, not a single figure:

1. **Total** (from one global `COUNT(DISTINCT …)` query).
2. **Breakdown per period** — one row per **year**; per **month** when the range is ≤ 2 years **or** the user asked for it ("per bulan", "bulanan", a specific year). Build with ONE `GROUP BY date_trunc('year'|'month', …)` shaped like the answer table.
3. **Breakdown per category** when the question crosses another dimension (per jenis permohonan, per risk, per system) — columns = code parts, rows = periods.
4. **Scope notes** — system, entity, the date column used; plus honest context: "2026 masih berjalan (YTD per {date})", "2022 is small because ERBA only started Sep 2022".

This breakdown is the **default shape** for time-flavoured questions — not an optional extra. But it is **not a substitute** for a segment that failed to resolve: if a segment is missing, resolve it or ask — do not answer with a trend table as an escape hatch.

Whether the per-period parts sum to the total depends on the column: versioned columns (period, `status`, system, code family) repeat one `nomor` across periods, so **do not sum the partitions**; lead with the global total and say the parts need not sum to it.

⚠️ **Do not add date ranges the question did not ask for.** "Sanity" guards like `>= '2000-01-01'` feel safe but they filter — and on data-quality questions they discard exactly the rows being sought. Ranges exist only when the question names them.

⚠️ **Date columns can hold sentinel values** far outside the operational range (e.g. year 1900 or 1970 as "unknown" markers). An unbounded trend will surface them as their own bucket. Check once before building a trend:
`SELECT MIN(<col>), MAX(<col>) FROM <table>` — if the minimum predates the system's age, add a lower bound or note it.

## "Masih berlaku" — two valid readings

| Reading | Filter | Rule source |
|---|---|---|
| **Active status** | `status='0999'` | `00-menghitung.md` §2 |
| **Active AND not expired** | `status='0999'` AND (`tanggal_exp` > today OR `tanggal_exp` empty) | this page |

When the question asks for both at once ("masih berlaku semua atau ada yang sudah lewat"), the answer **must** give **two labelled figures** — not pick one silently.

The two readings can diverge widely, and the gap **concentrates in the legacy system**: the system holding years of history has many NIEs whose status was never updated even though their dates passed. So **split per system** — a combined answer hides that the issue is one system's quirk. To gauge the size in one shot:
`COUNT(DISTINCT nomor) FILTER (WHERE <not expired>)` beside the active count, per system.

**Status and validity are separate dimensions.** `0099` "Tidak Berlaku" is a status marker, NOT the result of a date calculation — do not fold it into a `tanggal_exp`-based expiry count, and do not conclude an NIE was revoked just because its `tanggal_exp` passed.

## Routing

- "Kapan paling banyak berakhir" → group `tanggal_exp` with ONE `GROUP BY`; never hand-pick a year.
- Status/stages mentioned → **see** `20-status-pipeline.md`
- jenis permohonan mentioned → **see** `15-permohonan.md` (entity · date · status per the decision table, `00-menghitung.md` §1)
- Segments mentioned → **see** `10-segmen-produk.md`
