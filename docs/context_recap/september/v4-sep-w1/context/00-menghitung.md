# How to Count — Entity, Status Tiers, Exclusions, Casting

These rules apply to every data question.

**Scope.** This repo answers for the ERLA domain only. The only queryable tables are `t_produk_3_rilis_erla`, `t_btp_3_erla`, `m_trader_rla`, and `data_dictionary`. Any query naming a table outside this list (`t_produk_3_erba`, `t_btp_3_erba`, `m_trader_rba`) must be refused and never attempted.

The product tables are **versioned**: a single NIE can occupy multiple rows (`status='9999'` means the record has been superseded by a newer revision).

## 1. Entity, date, and status — decided from ONE row of the decision table

Domain facts that decide the branch (confirmed against the data, `15-permohonan.md` §versioning):

- **Revisions never change the NIE — new registrations can.** Perubahan and daftar ulang keep
  the same `nomor`; a new registration (`301`/`305`) can issue an ADDITIONAL NIE for the same
  product, so a product may hold several NIEs at once. **Count every NIE** — when one product
  carries more than one NIE (same name, same factory), count all of them as they stand; never
  pick just one.
- **One application per apply.** Every revision, perubahan, or daftar ulang creates a new
  `produk_id` (nomor pengajuan). Many NIEs carry more than one application.
- **One NIE can hold several `produk_id`.** Example: an NIE carries applications
  whose `produk_id` starts with `EREG…`.
- **Every NIE is counted.** When one product holds more than one NIE, count them all — do not
  collapse to a single NIE.

| Subject | Count with | Canonical date |
|---|---|---|
| Izin edar / NIE / produk | `COUNT(DISTINCT nomor)`, drop `nomor = ''` | `tanggal` (terbit) |
| Permohonan / pengajuan / registrasi | `COUNT(DISTINCT produk_id)` | `tanggal_bayar` |
| **Persetujuan** (already issued) | **`COUNT(DISTINCT nomor)`** | `tanggal` (terbit) |
| Perusahaan | `COUNT(DISTINCT trader_id)` from the product table | — |
| Surat keputusan | count from `nomor_surat` | — |

Notes that fix the branch:

- **`persetujuan` is a `nomor` subject, not a `produk_id` one.** "Persetujuan" means a product
  that has already been issued, so its entity is the NIE (`nomor`) — it is taken **out of** the
  Permohonan / pengajuan row above.
- **"Surat keputusan" counts `nomor_surat`.** In ERLA, `nomor_surat` is the real letter number
  (prefix `PN.…`), **not** the `produk_id`: `nomor_surat` and `produk_id` rarely coincide,
  so a decision count comes from the genuine surat number.
- **ERLA identifier prefixes.** `produk_id` starts with `EREG…`; an NIE starts with `MD …`
  (domestic) or `ML …` (import).
- **ERLA status tiers.** Terdaftar = `status IN ('0099','0999','0906','9999')`; Aktif = `status = '0999'`.
- **ERLA test-account exclusion.** `trader_id <> 3384`.
- **ERLA has no komitmen column.** `status_komitmen` and `jenis_penolakan_komitmen` do not exist
  in the ERLA tables.
- **ERLA types are native.** See §4 — carrying the TEXT casts into an ERLA query fails it.

The panel's four date bases map to columns (`05-filter-katalog.md`): Tanggal Permohonan →
`tanggal_aju` · Tanggal Bayar SPB → `tanggal_bayar` · Tanggal Terbit NIE → `tanggal` ·
Tanggal Terbit SPB → `tanggal_hprspb`.

- Subject, entity, and date come from the **same row** of this table. Choosing
  `produk_id` and then filtering on `tanggal` (issuance) mixes two populations: applications
  counted but filtered by the issuance event — unissued ones drop out, issued-outside-period
  ones slip in.
- `jenis_permohonan` (301–305) is the **service context of each application** (layanan) — apply it
  only when the question names the service (`15-permohonan.md`).
- Do not use `COUNT(*)` for NIE questions — it counts revision rows rather than unique entities, and no filter reliably corrects for that. The size of the overcount varies by table and filter; if you need to understand it, compare `COUNT(*)` with `COUNT(DISTINCT nomor)` under the same filter.
- Using `produk_id` for a NIE question is the most common entity mistake. One `nomor` spans many `produk_id`, so swapping the entity changes the answer — the wider the population, the larger the gap.
- For **pengajuan** questions, `COUNT(*)` and `COUNT(DISTINCT produk_id)` return the same result (`produk_id` is unique). Do not "correct" an already-correct application count.
- `tanggal_berkas` and `tanggal_diambil` are processing dates and are never used for counting.
- The same pairing logic applies to any coded column family, including columns not covered by any page.

**Choose the table based on the subject of the question, not on which columns happen to be available.** Questions about **processed food products** (including jenis permohonan, kemasan, segmen, product risk) are counted from `t_produk_3_rilis_erla`. Questions about **BTP / bahan tambahan pangan** (pewarna, pengawet, perisa, antioksidan, bentuk sediaan) are counted from `t_btp_3_erla`. The two share many column names (`jenis_permohonan`, `status`, …), so a query against the wrong table still runs and still returns a plausible number — pick the table from what is being asked, not from what is available.

## 2. Two status tiers — determined by the wording of the question

| Tier | Trigger | ERLA |
|---|---|---|
| **Terdaftar** (has been issued) | "terdaftar", "total", "berapa NIE", "pernah terbit" | `IN ('0099','0999','0906','9999')` |
| **Aktif** (narrower) | "aktif", "masih berlaku" | `= '0999'` |

- "saat ini" on its own does not imply the Aktif tier — "terdaftar … saat ini" still uses Terdaftar.
- When both readings are live, lead with Terdaftar and attach Aktif as a labelled second figure rather than swapping silently.
- Do not add a `tanggal_exp` narrowing unless the question asks for "masih berlaku".
- These tiers apply **only to issued-NIE populations**. Populations defined by some other workflow state (komitmen, pipeline stage, data quality) have their own status conditions; layering a valid-NIE set on top of them erases the population being asked about, because most of it has never had an NIE issued. Ask yourself: is this population defined by NIE issuance, or by something else? If something else — do not stack.
- **Only the raw-`pengajuan` branch (the Permohonan / pengajuan / registrasi row of §1) drops the status filter; every other branch keeps its tier.**

### Conflicting active versions — resolve once

This applies only when the answer reads a per-entity derived attribute (date, status, class) and the entity's active rows disagree on its value. A plain `COUNT(DISTINCT nomor)` is unaffected.

When it does apply, pick the determining version once — the active row with the latest date — and use it consistently for every derived attribute. ⚠️ Do not split the population with two separate `FILTER` conditions over the raw rows: an entity with rows on both sides gets counted twice. Test whether the parts sum to the total; if they exceed it, resolve the versions first rather than presenting the split.

## 3. Mandatory exclusions — there are exactly three

| What | ERLA |
|---|---|
| Test accounts | `trader_id <> 3384` |
| Empty `nomor` | `nomor <> ''` — only when the entity is `nomor` |
| Empty `status` | stored as **four spaces** — `TRIM(status)=''` catches it, `status <> ''` does not |

Test accounts barely move NIE counts, but they pile up in **Draft** — on a pipeline count they can flip the conclusion on their own. Always apply them, and mention them when comparing a figure against another source.

### A one-sentence test before adding any `WHERE` clause

> Could a row removed by this clause be part of a correct answer?
> If yes, this is a **narrowing** — and narrowing is allowed only when the question asks for it.
> If no, this is an **exclusion** — and exclusions are always allowed.

Test accounts and empty `nomor` pass that test: neither is a real izin edar regardless of the question. An empty column does not pass. Rows with an empty column are still real izin edar and still belong in the count — unless the question is about that column itself.

### Emptiness that correlates with meaning is a hidden filter

A column whose emptiness is not random is not a data defect — it marks a group. Filtering to "filled" values on such a column silently drops that group, with no error and no trace in the answer sentence.

The example worth memorising: **`daerah_pabrik` is empty exactly when the factory is abroad.** `WHERE daerah_pabrik IS NOT NULL` is therefore identical to "local products only" — it is not data cleaning. The three region columns are not even consistent with each other (`50-pihak-wilayah.md` §Wilayah).

To catch this before using any column as a guard, ask what the emptiness *means*. If empty means "not applicable to group X" rather than "not yet filled in", filtering on it discards group X. If you genuinely need fill rates, cross them against the column that defines the group instead of counting them in isolation.

## 4. Casting — ERLA columns are native

`t_produk_3_rilis_erla` and `t_btp_3_erla` are native: `tanggal`, `tanggal_bayar`, and `tanggal_exp` are already `timestamp`, and `trader_id` is already `bigint`. **Do not add the legacy TEXT casts** (`NULLIF(col,'')::timestamp`, `::bigint`) — they assume TEXT storage, they will fail the query, and they burn the single retry. PostgreSQL only: there is no `TRY_CAST`/`SAFE_CAST`.

## 5. Single-system scope

Every query in this repo runs against the ERLA tables alone. There is nothing to combine across sources: pick the ERLA table for the subject (above), apply the ERLA status tier (§2) and the ERLA exclusions (§3), and write plain ERLA filters.

**ERLA runs from 2012 to now** — older than the excluded legacy system, but still receiving rows. Take system age from this section only; do not guess it from column types or table names.

Before reporting 0 or "none" for a value, list the column's own values first (`SELECT DISTINCT <col>, COUNT(*) … GROUP BY 1`) — a 0 means the value is absent, not that the concept does not exist.

## 6. Numbers and execution

- **The headline comes from one global `COUNT(DISTINCT …)` with no `GROUP BY`.** Do not sum partitions when one entity can appear in more than one — which happens for versioned columns (period, `status`, code family), because a single `nomor` repeats across revisions. The test is simple: can one `nomor` hold more than one value in the grouping column? If yes, take the global count and say the parts need not sum to it. Columns where an entity holds exactly one value at a time may be summed.
- One statement per call, no `;`.
- Do not filter with `EXTRACT(YEAR …)` — it forces a full-table transfer. Use bounded ranges; keep `EXTRACT` for labelling grouped results.
- `ILIKE '%…%'` scans the whole column — use it once to discover a value, then count with `=` on that value.

## Routing

- Unresolved coded concept → **back** to the page map in `SEEKNAL_ASK.md`.
- Period / trend / validity mentioned → **see** `80-waktu-periode.md`.
- "belum / tanpa / kosong / belum ditetapkan" → **see** `90-kualitas-data.md` (there, dropping the status filter is the correct move).
- Touching a table not queried this turn → check types in `data_architecture.md` or run `describe_table` before writing casts.
