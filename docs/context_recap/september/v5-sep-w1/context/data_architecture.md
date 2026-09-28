# Database Map

Tables, join keys, and query scope.

Scope: **Registrasi Pangan only**. Pengawasan (pemeriksaan/pengujian/sampling/balai) has **no connected source** — say so honestly; never invent tables.

## Tables

| Table | Coverage | Types | Notes |
|---|---|---|---|
| `t_produk_3_rilis_erla` | 2012 → now | native TIMESTAMP/BIGINT | risk `jenis_dokumen` · no commitment · **final states only** |
| `t_btp_3_erla` | Des 2017 → now | native TIMESTAMP/BIGINT | still receiving rows; did not stop in 2024 |
| `m_trader_rla` | company master | mixed | scale column `skala_industri` |
| `data_dictionary` | — | — | code→label, one set of rows per category (`kategori` + `sumber`) |

**Types are native.** The ERLA tables store their dates as `timestamp` and their ids (`trader_id`) as `bigint` — **no casting is required**. Carrying a `NULLIF(...)::timestamp` / `::bigint` cast in from a differently-typed source **breaks the query** and burns the single retry (`00-menghitung.md` §4).

## Joins — no foreign keys, so this must be known, not guessed

This database has **no foreign keys and no indexes** on the two big tables — every query is a full scan, which makes the **number of queries** the main cost, not their complexity.

- product/BTP `.trader_id` → **LEFT JOIN** `m_trader_rla` (orphans exist; an INNER JOIN discards data).
- Count companies from `t.trader_id`, **never** `m.trader_id` (a LEFT JOIN produces NULL).
- Codes → `data_dictionary` via exact `kategori` + `kode` (plus `sumber`).
- Identity: `nomor` = NIE · `produk_id` = permohonan · `trader_id` = company.
- No pre-computed `mv_*` views exist — coverage is always read live from the four tables above.

## Identity value shapes — recognise them, do not invent them

`produk_id` and `nomor` are **not** free-floating strings; each has a pattern. Recognising them separates rows that **already have an NIE** from rows that are **still applications**.

| Column | ERLA | BTP |
|---|---|---|
| `produk_id` (application number) | `EREG…` | `EBTP…` |
| `nomor` with an issued NIE | `MD …` (domestic) / `ML …` (import) | same |
| `nomor` **without** an NIE | **same as `produk_id`** (`EREG…`) | same |

**Binding rules:**
- **`nomor` = `produk_id` → the row has NO NIE yet** (still an application). Never present an `EREG`/`EBTP`-prefixed value as an "NIE" — call it what it is: an application number that has not issued.
- **NIE prefixes follow origin, not the reverse.** `MD` = local, `ML` = import — consistent with `negara_pabrik`, but the origin filter stays `negara_pabrik` (`60-asal-produksi.md`), never the prefix. **Both prefixes occur in this repo** (`MD` and `ML`): the prefix marks the product's origin, never which system the row belongs to.
- **Every identifier (application number, NIE, brand, name) appearing in the answer must come from `execute_sql` this turn.** Formats outside the patterns above (e.g. `DMY-…`) do not exist in the database — never fabricate one.

## Scope — single system

This repository answers from **ERLA only**. The only tables in scope are the four listed above:

`t_produk_3_rilis_erla` · `t_btp_3_erla` · `m_trader_rla` · `data_dictionary`.

**A query against any table outside that list is forbidden** — say so instead of reaching for a table that is out of scope, and do not combine tables from outside the list into a result. All ERLA columns are **native** (`timestamp` / `bigint`): write the query with no casts.

**Multi-dimension shapes:** crossing dimensions ("per year AND region") = ONE query,
`GROUP BY date_trunc('year', tanggal), daerah_pabrik`. Independent aspects = one query each, synthesised in the answer. Do not mimic a 2D grouping with repeated 1D queries.

## Routing

- A column no page covers → `95-dimensi-lain.md` (the dimension discovery procedure).
- Touching a table not queried this turn → `describe_table` before writing the query.
