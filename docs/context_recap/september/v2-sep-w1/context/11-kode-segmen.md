# Segment Codes — jenis_pangan & kategori_pangan

AMDK, garam, formula bayi, and how to derive the rest.

`jenis_pangan` is the INDUK level; `kategori_pangan` is its ANAK level in the same hierarchy. Start from INDUK. Neither field is registered in `data_dictionary` — the mapping is empirical, and the answer should say so. The user filter panel's label↔code catalog (multi-select, thematic families like "mengandung susu", cair/padat formula pairs, BTP entries that only look like jenis pangan) lives in `05-filter-katalog.md`.

## Verified segment anchors

| Segment | ERBA | ERLA |
|---|---|---|
| AMDK | `jenis_pangan IN ('1401','1402')` | `jenis_pangan IN ('651','652','655')` |
| Garam beryodium | `jenis_pangan = '1204'` — an INDUK code covering every salt variant | no `1204` namespace; use `kategori_pangan='12010103'` or `nama_kategori ILIKE '%garam%'` (broader, also catches bumbu) |
| Formula bayi (strict) | `jenis_pangan IN ('1301','1302')` | `nama_kategori ILIKE 'Formula Bayi%'` |
| **Pangan bayi & anak** (family) | `klasifikasi_id='311'` | `klasifikasi_id='311'` — same column and code in both systems |

"Formula bayi" is not "produk bayi & anak" (much wider, many more codes). Choose from the wording: "**formula** bayi" → the strict row · "**pangan** bayi", "produk bayi" → the family row. Only when the sentence genuinely settles neither, ask.

⚠️ **In ERLA, the strict formula-bayi reading cannot go through `jenis_pangan`.** The codes are too coarse: `622` mixes Formula Bayi with Formula Lanjutan, Formula Pertumbuhan, and Formula Khusus Untuk Anak, while `624` is entirely Formula untuk Keperluan Medis Khusus. Using that trio for a "formula bayi" question inflates the result roughly **nine times over**. Use `nama_kategori` for the strict reading and `klasifikasi_id='311'` for the family; the three-code route is wrong for both.

## Two rules that are easy to violate without noticing

**No namespace overlap.** `jenis_pangan` shares not a single value between ERBA and ERLA — the code lengths and ranges differ. The three anchors above are just examples of a property that holds for **every** segment, including the hundreds not listed here. A code carried across systems always returns 0, and that 0 means wrong namespace.

**Parent before child.** An INDUK code covers its whole family; an ANAK code is one variant inside it and silently drops the siblings. Step down to a child only when the question names that variant. To see how many children an INDUK shelters:
`SELECT kategori_pangan, COUNT(*) … WHERE jenis_pangan='<induk>' GROUP BY 1`.

## Deriving a segment that is not in the table above

1. Probe `nama_kategori` to find the exact value (`12-nama-kategori.md`).
2. From the matching rows, read the accompanying `jenis_pangan` / `kategori_pangan` —
   `SELECT jenis_pangan, kategori_pangan, COUNT(*) … WHERE nama_kategori='<exact>' GROUP BY 1,2`.
3. If the mapping is a clean 1:1, count by code (cheaper and reusable). If it spreads across several codes, count by `nama_kategori` and state the spread. One `nama_kategori` value can map cleanly to a single code in one system while spreading across several in the other — check each side; do not conclude from either one alone.

## Breakdown & ranking by pangan group — which column carries it

**Preferred: `jenis_pangan`.** The panel's food dimension is a list of 4-digit `jenis_pangan`
codes ("UsusAyamGoreng (0803)", "Tempe (0620)" — `05-filter-katalog.md`), so a ranking question
("10 kategori/jenis pangan terbanyak") groups by **`jenis_pangan`** and labels the codes through
the panel catalog or `nama_kategori` (labels are often empty — say so).

**Fallback: `kategori_pangan`** when no `jenis_pangan` code/label resolves for the concept
(unknown code, free-text family): group on the **2-digit prefix** `LEFT(kategori_pangan, 2)`
(e.g. `07` bakeri · `08` daging · `13` PKGK) for the group level, and step down to the full ANAK
code only when the question names a variant. The prefix is a reliable group proxy — it equals
`LEFT(jenis_pangan, 2)` on 99.87% of ERBA rows that carry both (269.067 of 269.419, live
14 Sep 2026), and `kategori_pangan` is filled on **every** row that has a `jenis_pangan`
(212 distinct `jenis_pangan` codes, 1.185 `kategori_pangan` codes).

Anything deeper than two digits is not comparable across systems — ERBA and ERLA use different depths, and a prefix that looks the same is not the same category.

## Routing

- **Back to** `10-segmen-produk.md` if the segment turns out to be free text.
- Segment fits no code → **continue to** `12-nama-kategori.md`; do not answer with a different metric.
