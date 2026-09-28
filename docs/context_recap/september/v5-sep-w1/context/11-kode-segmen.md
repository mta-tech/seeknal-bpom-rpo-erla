# Segment Codes — jenis_pangan & kategori_pangan

AMDK, garam, formula bayi, and how to derive the rest.

`jenis_pangan` is the INDUK level; `kategori_pangan` is its ANAK level in the same hierarchy. Start from INDUK. Neither field is registered in `data_dictionary` — the mapping is empirical, and the answer should say so. The user filter panel's label↔code catalog (multi-select, thematic families like "mengandung susu", cair/padat formula pairs, BTP entries that only look like jenis pangan) lives in `05-filter-katalog.md`.

## Segment anchors

| Segment | ERLA |
|---|---|
| AMDK | `jenis_pangan IN ('651','652','655')` |
| Garam beryodium | use `kategori_pangan='12010103'` or `nama_kategori ILIKE '%garam%'` (broader, also catches bumbu) |
| Formula bayi (strict) | `nama_kategori ILIKE 'Formula Bayi%'` |
| **Pangan bayi & anak** (family) | `klasifikasi_id='311'` |

"Formula bayi" is not "produk bayi & anak" (much wider, many more codes). Choose from the wording: "**formula** bayi" → the strict row · "**pangan** bayi", "produk bayi" → the family row. Only when the sentence genuinely settles neither, ask.

⚠️ **In ERLA, the strict formula-bayi reading cannot go through `jenis_pangan`.** The codes are too coarse: `622` mixes Formula Bayi with Formula Lanjutan, Formula Pertumbuhan, and Formula Khusus Untuk Anak, while `624` is entirely Formula untuk Keperluan Medis Khusus. Using that trio for a "formula bayi" question inflates the result substantially. Use `nama_kategori` for the strict reading and `klasifikasi_id='311'` for the family; the three-code route is wrong for both.

## Rules that are easy to violate without noticing

**Codes come from the data, not from memory.** The anchors above are examples, not a complete list. A code that returns no rows is absent from the data — list the column's own values before concluding a segment is missing.

**Parent before child.** An INDUK code covers its whole family; an ANAK code is one variant inside it and silently drops the siblings. Step down to a child only when the question names that variant. To see how many children an INDUK shelters:
`SELECT kategori_pangan, COUNT(*) … WHERE jenis_pangan='<induk>' GROUP BY 1`.

**Panel labels are a legacy catalog.** A label quoting a code ("Sosis Daging (0809)") is a legacy panel entry; ERLA uses its own `jenis_pangan` / `kategori_pangan` values, so resolve ERLA segments through `kategori_pangan` / `nama_kategori`. Never synthesize an equivalent from a similar name — crosswalk-by-name invents a population the question never asked about and multiplies the headline.

## Deriving a segment that is not in the table above

1. Probe `nama_kategori` to find the exact value (`12-nama-kategori.md`).
2. From the matching rows, read the accompanying `jenis_pangan` / `kategori_pangan` —
   `SELECT jenis_pangan, kategori_pangan, COUNT(*) … WHERE nama_kategori='<exact>' GROUP BY 1,2`.
3. If the mapping is a clean 1:1, count by code (cheaper and reusable). If it spreads across several codes, count by `nama_kategori` and state the spread. One `nama_kategori` value can map cleanly to a single code or spread across several — read the actual spread before deciding how to count.

## Breakdown & ranking by pangan group — which column carries it

**Preferred: `jenis_pangan`.** The panel's food dimension is a list of 4-digit `jenis_pangan`
codes ("UsusAyamGoreng (0803)", "Tempe (0620)" — `05-filter-katalog.md`), so a ranking question
("10 kategori/jenis pangan terbanyak") groups by **`jenis_pangan`** and labels the codes through
the panel catalog or `nama_kategori` (labels are often empty — say so).

**Fallback: `kategori_pangan`** when no `jenis_pangan` code/label resolves for the concept
(unknown code, free-text family): group on the **2-digit prefix** `LEFT(kategori_pangan, 2)`
(e.g. `07` bakeri · `08` daging · `13` PKGK) for the group level, and step down to the full ANAK
code only when the question names a variant. The prefix is a reliable group proxy — it equals
`LEFT(jenis_pangan, 2)` on nearly all rows that carry both, and
`kategori_pangan` is filled on **every** row that has a `jenis_pangan`.

Anything deeper than two digits is not comparable — the code depths vary, and a prefix that looks the same is not the same category.

## Routing

- **Back to** `10-segmen-produk.md` if the segment turns out to be free text.
- Segment fits no code → **continue to** `12-nama-kategori.md`; do not answer with a different metric.
