# Risk & Commitment

kategori risiko, menengah rendah/tinggi, MR, MT, komitmen, pemenuhan, dibatalkan, disetujui.

**Komitmen is structurally ERBA-only** (`status_komitmen` and `jenis_penolakan_komitmen` exist only in `t_produk_3_erba`) — state that limit. **Risk is not**; read the section below to the end.

## Risk — two columns, two schemas, and the codes do not correspond

This is the most expensive trap in the database. Risk lives in a different column per system, and the same code number points to a different class in each.

| | Column | `sumber` in dictionary | Codes |
|---|---|---|---|
| **ERBA** | `kategori_dokumen` | **ERBA only** | `301` Tinggi · `302` Menengah Tinggi · `303` Menengah Rendah · `304` Tinggi Notifikasi |
| **ERLA** | `jenis_dokumen` | **ERLA and ERBA** | `000` Belum Dikategorikan · `301` Pangan Low Risk · `302` Pangan High Risk · `303` Pangan Medium Risk |

⚠️ **`kategori_dokumen` exists and is filled in all four tables — including ERLA.** That is not permission to use it. The dictionary records the category's `sumber` as **ERBA only**, so the `301`–`304` values on the ERLA side are **not** the BPOM risk schema. UNIONing them adds two different schemas together and produces a number that means nothing — with no error, and a result that looks perfectly reasonable.

Never put `kategori_dokumen` and `jenis_dokumen` in the same query.

### Cross-schema correspondence — 3 classes, not 4

Both columns live side by side in `t_produk_3_erba`, so the correspondence is a structural fact you can derive yourself (`GROUP BY kategori_dokumen, jenis_dokumen` in ERBA) rather than a guess:

| ERBA `kategori_dokumen` | ERLA `jenis_dokumen` |
|---|---|
| `301` Tinggi | `302` Pangan High Risk |
| `302` Menengah Tinggi | `303` Pangan Medium Risk |
| `303` Menengah Rendah | `301` Pangan Low Risk |
| `304` Tinggi Notifikasi | **no clean counterpart** — report it as an ERBA-specific class |

Note that `301` and `303` **swap meanings** between the schemas. Using an ERBA code on ERLA produces no error — it produces the wrong class. The ERLA schema has only three levels, so "Tinggi Notifikasi" cannot be separated there.

### Scope — default to ERBA and say so; do not ask, do not merge silently

The BPOM risk schema is the ERBA schema. Therefore:

| Question | Do this |
|---|---|
| Risk, system not named | **Answer ERBA** and **say** that this risk schema belongs to ERBA. Do not `request_clarification` — the scope is decided by the schema itself |
| ERLA / "gabungan" / "nasional" explicitly named | Use `jenis_dokumen` on the ERLA side via the correspondence table, present **per side, labelled**, and state that the two systems use different schemas |

What is forbidden is not answering ERBA — that is the correct default. What is forbidden is **merging without correspondence** (`kategori_dokumen` on both sides) and **presenting ERBA figures as national figures without the caveat**.

- Questions touching the risk FAMILY report each class as its own labelled figure. Do not widen one requested class into its neighbour — "produk MR" means Menengah Rendah only.
- "Risiko Tinggi" in the ERBA schema = `IN ('301','304')` — Tinggi includes Tinggi Notifikasi.
- MR/MT are official BPOM shorthand for Menengah Rendah / Menengah Tinggi. Write them out in full at least once; "Medium Risk" alone loses the Rendah/Tinggi distinction, and in the ERLA schema "Medium" actually corresponds to Menengah **Tinggi**.

## Commitment — the `status_komitmen` column

**Mixed format.** Stored as TEXT in two shapes: some rows `'5'`, some `'5.0'`, for the same logical value.

```sql
WHERE status_komitmen = '5'                              -- WRONG, loses the '5.0' rows
WHERE ROUND(status_komitmen::numeric)::int::text = '5'   -- CORRECT
WHERE status_komitmen LIKE '5%'                          -- CORRECT, more concise
```

Affected codes: `0, 1, 4, 5, 7, 8, 9`. Normalisation applies to **every** `status_komitmen` filter.

**Canonical "Disetujui" = `4` + `7` combined**, but the answer always shows the labelled parts: `4` Komitmen Disetujui (murni) · `7` Komitmen Disetujui Dengan Catatan · the combined total as a labelled sum. Code `8` (Validasi Pembatalan) is transient, moving towards `5` — include dates if you use it.

**Commitment rejection reasons** — `jenis_penolakan_komitmen` (ERBA-only, codes 1–10) is **multi-valued**, pipe-separated (`'1|3'`). Match with `string_to_array(col,'|') @> ARRAY['<code>']`, never with plain equality — plain equality loses every combined row. To rank reasons: `unnest(string_to_array(jenis_penolakan_komitmen,'|'))` then `GROUP BY`.

## Two readings of commitment — decide from the SUBJECT, before writing SQL

| Reading | Trigger | Filter |
|---|---|---|
| **A — NIEs whose commitment status is X** | the question counts "NIE"/"izin edar" as the subject | keep ALL NIE filters (status, jenis_permohonan) **and** add the `status_komitmen` filter |
| **B — permohonan whose commitment [ended up X]** | the question asks how many were "dibatalkan/ditolak/disetujui" as a lifecycle outcome | **drop** the valid-NIE `status` filter and `jenis_permohonan` |

The reason is structural, not numeric: **most commitment events happen before the NIE is issued**, so reading B's population mostly has no NIE. Requiring an active NIE status there filters out exactly the population being asked about.

Decide from the subject of the question — never from the size of the result. Picking a reading because its number "looks more reasonable" is tuning the filter towards the answer you hope for.

**"Produk MR yang dibatalkan" (no NIE/izin-edar word) = komitmen dibatalkan, code `5`** — reading B, `produk_id`, no NIE filter. The NIE-side word for termination is *dicabut/dihapus* (`0000`/`0009`, `20-status-pipeline.md`); "dibatalkan" standing alone belongs to the commitment lifecycle. Live check 4 Sep 2026: MR kode 5 = 6.041 — the NIE-termination reading returns a different, larger number.

**`status_komitmen` is a per-application attribute that MOVES.** One NIE's revision rows can carry
different komitmen codes: live example `MD 011171000100051` = 301/komitmen `7` (2022) → 302/`4`
(2023) → 302/`1` (2024). Consequences:
- Counting rows with kode 4/7 counts **applications at that revision's state** — an NIE whose
  komitmen later moved can appear in several codes. That is correct for reading B ("permohonan
  yang komitmennya disetujui"), and the labelled 4/7 split stays mandatory.
- For **"kondisi komitmen NIE sekarang"** (reading A on the current state), resolve to the
  **latest application row per `nomor`** first (`DISTINCT ON (nomor) … ORDER BY nomor,
  tanggal_bayar DESC`), then count — otherwise a finished-commitment NIE is still counted under
  its old code.

## Routing

- **"Berdasarkan Kontrak" / makloon is NOT this page** — that is the `status_produk` column
  (`304`), a production-method filter. → **see** `60-asal-produksi.md` (panel binding).
- Meaning of an unlisted `status_komitmen` code → dictionary category `STATUS_KOMITMEN`, `sumber` `ERBA` (P2 path, Gate 2).
- Process stages also mentioned → **see** `20-status-pipeline.md`
- jenis permohonan also mentioned → **see** `15-permohonan.md`
- "belum ditetapkan kategori risikonya" → **see** `90-kualitas-data.md` (that is `jenis_dokumen='000'`, and the status filter is dropped there).
- Period mentioned → **see** `80-waktu-periode.md`
