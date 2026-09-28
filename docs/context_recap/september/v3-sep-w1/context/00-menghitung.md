# How to Count — Entity, Status Tiers, Exclusions, Casting, UNION

These rules apply to every data question.

The product tables are **versioned**: a single NIE can occupy multiple rows (`status='9999'` means the record has been superseded by a newer revision).

## 1. Entity, date, and status — decided from ONE row of the decision table

Domain facts that decide the branch (confirmed against the data, `15-permohonan.md` §versioning):

- **Revisions never change the NIE — new registrations can.** Perubahan and daftar ulang keep
  the same `nomor`; a new registration (`301`/`305`) can issue an ADDITIONAL NIE for the same
  product, so a product may hold several NIEs at once. For per-product questions pick one NIE
  deterministically — prefer the `0999` row, then the latest `tanggal` — and say which one you
  used.
- **One application per apply.** Every revision, perubahan, or daftar ulang creates a new
  `produk_id` (nomor pengajuan). 30.354 NIEs carry more than one application (max 71).
- **"Surat keputusan" is counted as applications** — the question asks how many times a decision
  was made, so count `produk_id`.

| Wording in the question | Entity | Date column | Status filter |
|---|---|---|---|
| "diajukan" · "Tanggal Permohonan" · "pengajuan masuk" · "permohonan registrasi" (volume) · "berapa kali pengajuan" | `COUNT(DISTINCT produk_id)` | **`tanggal_aju`** | **none** |
| "surat keputusan" / "keputusan yang diterbitkan" | `COUNT(DISTINCT produk_id)` | `tanggal_bayar` | valid tier |
| "NIE" · "izin edar" · "produk terdaftar" · "terbit" · **"pendaftaran baru"** | `COUNT(DISTINCT nomor)` | **`tanggal`** | valid tier |
| **"disetujui"** · **"persetujuan"** · "bayar SPB" | `COUNT(DISTINCT nomor)` | **`tanggal_bayar`** | valid tier |
| "perusahaan" | `COUNT(DISTINCT t.trader_id)` from the **product table**, not `m_trader_*` | — | — |

The panel's four date bases map to columns (`05-filter-katalog.md`): Tanggal Permohonan →
`tanggal_aju` · Tanggal Bayar SPB → `tanggal_bayar` · Tanggal Terbit NIE → `tanggal` ·
Tanggal Terbit SPB → **does not exist** — say so and offer the other two, labelled.

- Entity, date column, and status filter come from the **same row** of this table. Choosing
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

**Choose the table based on the subject of the question, not on which columns happen to be available.** Questions about **processed food products** (including jenis permohonan, kemasan, segmen, product risk) are counted from `t_produk_3_erba` / `t_produk_3_rilis_erla`. Questions about **BTP / bahan tambahan pangan** (pewarna, pengawet, perisa, antioksidan, bentuk sediaan) are counted from `t_btp_3_erba` / `t_btp_3_erla`. The two share many column names (`jenis_permohonan`, `status`, …), so a query against the wrong table still runs and still returns a plausible number — pick the table from what is being asked, not from what is available. Combine all four tables only when the user explicitly asks to include BTP (`data_architecture.md`).

## 2. Two status tiers — determined by the wording of the question

| Tier | Trigger | ERBA | ERLA |
|---|---|---|---|
| **Terdaftar** (has been issued) | "terdaftar", "total", "berapa NIE", "pernah terbit" | `IN ('0999','0906','9999')` | `IN ('0099','0999','0906','9999')` |
| **Aktif** (narrower) | "aktif", "masih berlaku" | `= '0999'` | `= '0999'` |

- "saat ini" on its own does not imply the Aktif tier — "terdaftar … saat ini" still uses Terdaftar.
- When both readings are live, lead with Terdaftar and attach Aktif as a labelled second figure rather than swapping silently.
- Do not add a `tanggal_exp` narrowing unless the question asks for "masih berlaku".
- These tiers apply **only to issued-NIE populations**. Populations defined by some other workflow state (komitmen, pipeline stage, data quality) have their own status conditions; layering a valid-NIE set on top of them erases the population being asked about, because most of it has never had an NIE issued. Ask yourself: is this population defined by NIE issuance, or by something else? If something else — do not stack.
- **Only the raw-`pengajuan` branch (row 1 of the table in §1) drops the status filter; every other branch keeps its tier.**

### Conflicting active versions — resolve once

This applies only when the answer reads a per-entity derived attribute (date, status, class) and the entity's active rows disagree on its value. A plain `COUNT(DISTINCT nomor)` is unaffected.

When it does apply, pick the determining version once — the active row with the latest date — and use it consistently for every derived attribute. ⚠️ Do not split the population with two separate `FILTER` conditions over the raw rows: an entity with rows on both sides gets counted twice. Test whether the parts sum to the total; if they exceed it, resolve the versions first rather than presenting the split.

## 3. Mandatory exclusions — there are exactly three

| What | ERBA | ERLA |
|---|---|---|
| Test accounts | `trader_id::bigint NOT IN (5,17,50,85)` | `trader_id <> 3384` |
| Empty `nomor` | `nomor <> ''` — only when the entity is `nomor` | same |
| Empty `status` in ERBA | stored as **four spaces** — `TRIM(status)=''` catches it, `status <> ''` does not | — |

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

## 4. Casting — only `t_produk_3_erba` (every column is TEXT)

| Column | Cast |
|---|---|
| `tanggal`, `tanggal_bayar`, `tanggal_exp` | `NULLIF(col,'')::timestamp` |
| `trader_id` | `::bigint` |
| `status_komitmen` | `ROUND(...::numeric)::int::text` |

⚠️ **Types are per TABLE, not per system.** In `t_btp_3_erba` the dates are already `timestamp` and `trader_id` is already `bigint` — carrying the product casts over breaks the query and burns the single retry. `t_produk_3_rilis_erla` and `t_btp_3_erla` are native, no casts. PostgreSQL only: there is no `TRY_CAST`/`SAFE_CAST`.

## 5. UNION of ERBA + ERLA

```sql
SELECT nomor, tanggal::timestamp AS tanggal, trader_id::bigint AS trader_id
FROM t_produk_3_erba
WHERE tanggal IS NOT NULL AND tanggal <> ''
  AND status IN ('0999','0906','9999')
  AND trader_id::bigint NOT IN (5,17,50,85)
  AND tanggal::timestamp >= '{Y}-01-01' AND tanggal::timestamp < '{Y+1}-01-01'
UNION ALL
SELECT nomor, tanggal, trader_id
FROM t_produk_3_rilis_erla
WHERE status IN ('0099','0999','0906','9999')
  AND trader_id <> 3384
  AND tanggal >= '{Y}-01-01' AND tanggal < '{Y+1}-01-01'
```

Write the WHERE separately per side — the status sets, test-account filters, and casts genuinely differ. `nomor` values do not overlap across systems, so `UNION ALL` is safe.

**ERBA runs from Sep 2022 to now (zero rows before that); ERLA runs from 2012 to now — older, but still receiving rows.** Take system age from this section only; do not guess it from column types or table names (`t_produk_3_erba` being all-TEXT is a migration artifact, not a sign of age). Guessing produces answers with reversed sentences while the numbers stay correct.

⚠️ **When grouping a UNION result, check each group's system composition.** One value can come almost entirely from one system and the next from the other. When the composition differs, a column that exists in only one system starts reading as a property of the group — when it is a property of the system. Decide and state: split per system, or show only columns both systems share. A single `GROUP BY system, <dimensi>` makes this visible.

### Before any UNION: four ways the same concept can differ across systems

A shared column name does not guarantee a shared meaning. Four situations, each producing a wrong number without an error:

| Situation | Tell | Correct handling |
|---|---|---|
| Column does not exist on one side | `information_schema` | The question is structurally single-system — answer for the side that has it and state the limit. Do not present it as a national figure |
| Exists but nearly empty | lopsided fill rates | That side probably uses a **different column** for the same concept — look for the candidate before concluding |
| Code ranges differ | values from one side return 0 on the other | Separate namespaces. 0 means "wrong system's code", not "the concept does not exist here". Resolve each side against its own values |
| Filled, similar codes — but a different SCHEMA | no visible sign at all | **The most dangerous one.** The query runs, the number looks plausible, and the result is a sum of two different schemas. See `30-risiko-komitmen.md` |

The fourth situation cannot be detected from fill rates or row counts — only from the dictionary's `sumber` description. Before UNIONing a coded column, confirm its `sumber` in `data_dictionary` covers **both** systems. If `sumber` names only one system, that column is not the same field on the other side, no matter how identical the name and how full the data.

The closing rule: before reporting 0 or "none" for one system, list that system's own values first — and before adding two sides together, confirm they share a schema.

## 6. Numbers and execution

- **The headline comes from one global `COUNT(DISTINCT …)` with no `GROUP BY`.** Do not sum partitions when one entity can appear in more than one — which happens for versioned columns (period, `status`, system, code family), because a single `nomor` repeats across revisions. The test is simple: can one `nomor` hold more than one value in the grouping column? If yes, take the global count and say the parts need not sum to it. Columns where an entity holds exactly one value at a time (`status_komitmen`) may be summed.
- One statement per call, no `;`.
- Do not filter with `EXTRACT(YEAR …)` — it forces a full-table transfer. Use bounded ranges; keep `EXTRACT` for labelling grouped results.
- `ILIKE '%…%'` scans the whole column — use it once to discover a value, then count with `=` on that value.

## Routing

- Unresolved coded concept → **back** to the page map in `SEEKNAL_ASK.md`.
- Period / trend / validity mentioned → **see** `80-waktu-periode.md`.
- "belum / tanpa / kosong / belum ditetapkan" → **see** `90-kualitas-data.md` (there, dropping the status filter is the correct move).
- Touching a table not queried this turn → check types in `data_architecture.md` or run `describe_table` before writing casts.
