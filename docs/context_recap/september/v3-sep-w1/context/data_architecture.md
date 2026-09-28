# Database Map

Tables, join keys, UNION topology, and how ERBA differs from ERLA.

Scope: **Registrasi Pangan only**. Pengawasan (pemeriksaan/pengujian/sampling/balai) has **no connected source** — say so honestly; never invent tables.

## Tables

| Table | Coverage | Types | Notes |
|---|---|---|---|
| `t_produk_3_erba` | Sep 2022 → now | **ALL TEXT — casting required** | risk `kategori_dokumen` · commitment `status_komitmen` |
| `t_produk_3_rilis_erla` | 2012 → now | TIMESTAMP/BIGINT | risk `jenis_dokumen` (different codes) · no commitment · **final states only** |
| `t_btp_3_erba` | Jun 2022 → now | **MIXED — not all TEXT** | dates & `trader_id` already native |
| `t_btp_3_erla` | Des 2017 → now | native | still receiving rows; did not stop in 2024 |
| `m_trader_rba` / `m_trader_rla` | company master | mixed | scale column named differently: `skala_industri_id` vs `skala_industri` |
| `data_dictionary` | — | — | code→label, exactly 21 categories (`kategori` + `sumber`) |

**Types are per TABLE, not per system.** "ERBA is all-TEXT" holds for `t_produk_3_erba` and for nothing else — `t_btp_3_erba` shares the system but not the types. Carrying a cast across **breaks the query** and burns the single retry (`00-menghitung.md` §4).

## Column structure & system asymmetry

**`t_produk_3_rilis_erla` is a pure subset of `t_produk_3_erba`**: 94 shared columns, **0** ERLA-only columns, and only **6** ERBA-only columns —
`ecolabel` · `jenis_penolakan_komitmen` · `kode_kbli` · `sni_sukarela` · `status_komitmen` ·
`sub_kemasan_id`.

The consequence: any question resting on one of those six columns is **structurally single-system**, not a scope choice. Name the system; do not present the result as a national figure.

## Joins — no foreign keys, so this must be known, not guessed

This database has **no foreign keys and no indexes** on the two big tables — every query is a full scan, which makes the **number of queries** the main cost, not their complexity.

- product/BTP `.trader_id` → **LEFT JOIN** `m_trader_*` (orphans exist; an INNER JOIN discards data).
- Count companies from `t.trader_id`, **never** `m.trader_id` (a LEFT JOIN produces NULL).
- Codes → `data_dictionary` via exact `kategori` + `kode` (plus `sumber`).
- Identity: `nomor` = NIE · `produk_id` = permohonan · `trader_id` = company.
- No combined `mv_*` views exist — combined coverage is always a manual UNION.

## Identity value shapes — recognise them, do not invent them

`produk_id` and `nomor` are **not** free-floating strings; each has a per-system pattern. Recognising them separates rows that **already have an NIE** from rows that are **still applications**.

| Column | ERBA | ERLA | BTP |
|---|---|---|---|
| `produk_id` (application number) | `ERBA…` | `EREG…` | `EBTP…` |
| `nomor` with an issued NIE | `MD …` (domestic) / `ML …` (import) | same | same |
| `nomor` **without** an NIE | **same as `produk_id`** (`ERBA…`/`EREG…`) | same | same |

**Binding rules:**
- **`nomor` = `produk_id` → the row has NO NIE yet** (still an application). Never present an `ERBA`/`EREG`/`EBTP`-prefixed value as an "NIE" — call it what it is: an application number that has not issued.
- **NIE prefixes follow origin, not the reverse.** `MD` = local, `ML` = import — consistent with `negara_pabrik`, but the origin filter stays `negara_pabrik` (`60-asal-produksi.md`), never the prefix.
- **Every identifier (application number, NIE, brand, name) appearing in the answer must come from `execute_sql` this turn.** Formats outside the patterns above (e.g. `DMY-…`) do not exist in the database — never fabricate one.

## UNION topology

| Intent | Tables |
|---|---|
| NIE / processed food products (combined) | `t_produk_3_erba` ∪ `t_produk_3_rilis_erla` |
| BTP (combined) | `t_btp_3_erba` ∪ `t_btp_3_erla` |
| Total including BTP (**only when the user explicitly asks**) | all four |

"Pangan olahan" = the product tables only. The WHERE is written separately per side — status sets, jenis_permohonan sets, casts, and test-account filters all differ (`00-menghitung.md` §5).

> **Forecast exception:** this topology is for general analyst queries only. **Forecast is ERBA-only** — never UNION ERLA into a `run_forecast` series (`forecast_guide.md`). And do not query the stale `forecast_permohonan` table; compute through `run_forecast`.

**Multi-dimension shapes:** crossing dimensions ("per year AND region") = ONE query,
`GROUP BY date_trunc('year', tanggal), daerah_pabrik`. Independent aspects = one query each, synthesised in the answer. Do not mimic a 2D grouping with repeated 1D queries.

## Routing

- A column existing in only one system, or a column no page covers →
  `95-dimensi-lain.md` (the dimension discovery procedure).
- Touching a table not queried this turn → `describe_table` before writing casts.
