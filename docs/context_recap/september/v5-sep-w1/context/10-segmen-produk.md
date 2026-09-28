# Product Segmentation

`jenis_pangan` / `kategori_pangan` for segments such as bayi, formula, kopi, instan, AMDK, garam, sirup, mi, susu, roti, anggur, serbuk, and BTP.

## Segment resolution order

1. **Start with the coded columns.** `jenis_pangan` (INDUK) and `kategori_pangan` (ANAK) are precise, inexpensive to query, and reusable. Prefer `jenis_pangan`; when no `jenis_pangan` code/label resolves, the **`kategori_pangan` aspect** takes over (2-digit prefix = group level — facts and rules in `11-kode-segmen.md`).
2. **Fall back to free text** — `nama_kategori`. Its coverage can be thin, so check it first (`00-menghitung.md` §5) and probe the values from the data before relying on it.
3. Use `nama` / `merk` only when the question is explicitly about a product name or brand. `nama_produk` does not exist.

**Use ILIKE to discover, `=` to count.** Run ILIKE once — scoped, in a single query, with a LIMIT — to see the exact values, then count with `=` on those values. Counting directly through a pattern also captures neighbouring values the question never asked about. The query still succeeds and the number still looks plausible, which is exactly why the overcount is hard to notice.

## Two structural rules

**Codes come from the data, not from memory.** Neither `jenis_pangan` nor `kategori_pangan` is registered in `data_dictionary` — the mapping is empirical, so resolve it by probing the data. A 0-row result means the value is absent, not that the segment does not exist. `kategori_pangan` is comparable on the 2-digit prefix.

**Resolve the parent before the child.** `kategori_pangan` is the ANAK level under `jenis_pangan`; they are not parallel fields. Start from the INDUK level and step down to ANAK only when the question names that specific variant. Choosing an ANAK value for a family-level question silently excludes its sibling categories. To see whether the chosen INDUK covers several children:
`SELECT kategori_pangan, COUNT(*) … WHERE jenis_pangan='<induk>' GROUP BY 1`.

## Close the set — this applies to free text too

One ILIKE usually matches several `nama_kategori` values, and those values are not equivalent. "Kopi" covers a dozen-plus: Kopi Bubuk, Kopi Instan, Minuman Kopi, Biji Kopi, Minuman Serbuk Kopi — while "kopi instan" is exactly one. Answering "how many coffee products" with the Kopi Instan row alone is the free-text version of picking one code out of a set.

**A sibling can be larger than the requested value.** Sibling categories often differ by a single word (`Sirup Berperisa` vs `Sirup Encer Berperisa`), and the unrequested one can be bigger. Widening the pattern is not just additive — it can flip the ranking and hand the answer to a segment nobody asked about. Read the probe results before deciding the width.

**Spelling varies within the same column** (for example *i/y* variants of loanwords) — two values, one concept. A pattern anchored to one spelling loses the other without a sound. Check the probe list instead of assuming uniform spelling.

State the matched values in the answer so the reader can see the scope used. The width is decided by the question, not by string similarity — when it is genuinely ambiguous, ask (Gate 1).

## Common mistakes to avoid

- Answering a segment question with an annual trend because the segment could not be resolved. A segment question is answered about the segment; provide a trend only when a trend was requested.
- Concluding that a segment does not exist from a 0-row result without first listing the column's own values.

## Routing

- Segment has a code / question names a specific variant → **continue to** `11-kode-segmen.md`
- Free-text segment needing a probe → **continue to** `12-nama-kategori.md`
- Question also mentions country / origin / import / local → **see** `60-asal-produksi.md`
- Question also mentions packaging → **see** `40-kemasan.md`
- Segment is BTP (pewarna, pengawet, perisa) → **see** `70-btp.md`
- A probe returned no rows → list the column's values (`SELECT DISTINCT <col>, COUNT(*) … GROUP BY 1`) before concluding anything.
