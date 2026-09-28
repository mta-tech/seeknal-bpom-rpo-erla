# Status Codes — mapping codes to labels, closing sets, handling codes missing from the dictionary

Use this page when the stage being asked about is not in the bucket table of `20-status-pipeline.md`, or when a code needs to be translated into a label for the answer.

## How to look it up

```sql
SELECT kode, deskripsi, sumber FROM data_dictionary
WHERE kategori = 'STATUS' ORDER BY sumber, kode;
```

The category is small — read all of it; do not ILIKE across categories.

## Three traps that make lookups fail silently

**1. ERLA codes are stored without zero-padding.** The dictionary stores `999`, `99`, `9`, `0`; the data stores four characters `0999`, `0099`, `0009`, `0000`. Filtering directly with the dictionary's values returns **zero rows**. Pad first: `LPAD(kode,4,'0')`.

**2. The `status` column mixes two namespaces.** Some values carried by ERBA data — `0500`, `0504`, `0417`, `0900`, `0909`, `0916`, plus `0299` in `t_btp_3_erla` — are actually registered in the **ERLA** block with official descriptions. "Not in the ERBA block" means "check the ERLA block", **not** "unknown code". Labelling these as anomalies throws away information that is actually available.

**3. One description attaches to several codes.** "Pendaftar - Perlu Data Tambahan" = **5 codes** (`0901`,`0914`,`0915`,`0917`,`0951`) · "Pendaftar - Draft" = 3 (`0900`,`0910`,`0912`) · "Pendaftar - Proses Verifikasi Ditolak" = 3 (`0905`,`0909`,`0916`). A label search looks like it returned duplicates — it did not; each row carries its own population. When a description repeats, take **all** codes sharing it.

## The closure boundary — widening is as wrong as narrowing

"Ditolak Sistem" (`0908`,`0911`,`0918`) and "Ditolak petugas" (`0902`,`0905`,`0913`) share the word *ditolak* but have different populations. Merging them because the strings look similar produces a number nobody asked for. **The bucket list in `20-status-pipeline.md` defines the edges of the set, not keyword matches** — and that list wins over dictionary lookups.

## A value that will never appear in the dictionary

Empty `status` in ERBA is stored as **four spaces** — not NULL, not `''`. `TRIM(status)=''` catches it; `status <> ''` does not. This is the largest group absorbed by the `NOT IN` bucket; mention it when presenting a `NOT IN` total.

## Routing

- **Back to** `20-status-pipeline.md` once the codes are resolved — the buckets and answer-shape rules live there.
- The code turns out to belong to a different column (not `status`) → `95-dimensi-lain.md`; the values `301`/`302` appear in **9 different categories**.
