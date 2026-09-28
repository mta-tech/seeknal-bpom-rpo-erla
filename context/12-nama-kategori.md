# nama_kategori — Free-Text Probing for Segments

kopi instan, sirup berperisa, mi instan, red wine, serbuk.

`nama_kategori` coverage can be thin — check it first (`00-menghitung.md` §5) and probe the values from the data.

## Three-step procedure

```sql
-- 1. DISCOVER the exact value (once, scoped, with a LIMIT)
SELECT nama_kategori, COUNT(*) FROM t_produk_3_rilis_erla
WHERE nama_kategori ILIKE '%<term>%' GROUP BY 1 ORDER BY 2 DESC LIMIT 20;

-- 2. DECIDE the width from the question (see "closing the set" below)

-- 3. COUNT with exact matches
WHERE nama_kategori = '<exact value>'          -- or IN ('<a>','<b>') for a compound concept
```

Do not count through ILIKE. A pattern also captures neighbouring values the question never asked about, and the difference is invisible in the result — the query succeeds and the number looks plausible.

## Exact values

| Concept | `nama_kategori` value |
|---|---|
| Kopi instan | `'Kopi Instan'` (also exists: `'Kopi Instan Dekafein'`) |
| Sirup berperisa | `'Sirup Berperisa'` — has a sibling `'Sirup Encer Berperisa'`, possibly larger |
| Mi instan | `'Mi Instan'` (also `'Mi Instan Lainnya'`) |
| Anggur merah | `'Still Grape Wine Merah / Anggur Merah (Red Wine)'` — long, contains slashes and brackets |
| Minuman serbuk berperisa | `'Minuman Serbuk Berperisa (Tidak Berkarbonat)'` |

## Closing the set — where this most often goes wrong

One ILIKE usually matches several values, and the values are not equivalent:

- **Too narrow.** "Kopi" covers Kopi Bubuk, Kopi Instan, Minuman Kopi, Biji Kopi, Kopi Celup, Minuman Serbuk Kopi. Answering "how many coffee products" with the Kopi Instan row alone is the free-text version of picking one code from a set.
- **Too wide.** Sibling categories differing by one word can be larger than the one requested — widening a pattern can flip the ranking. A pattern also catches names that merely contain the word ("Bumbu Penguat Rasa dan Garam" is not garam).
- **Spelling varies within the same column** (e.g. *i/y* variants of loanwords) — two values, one concept; a pattern anchored to one spelling loses the other. Read the probe list; do not assume uniform spelling.

The width is decided by the **question**, not by string similarity. State the values used in the answer so the scope is visible. Genuinely ambiguous → ask (Gate 1); do not guess.

## When you get zero rows

Do not reshuffle keywords. Zero after a reasonable probe → answer "not found" honestly.

For sensitive segment answers (pencabutan, pembatalan), skim the `nama`/`merk` of matching rows and report any that clearly fall outside the segment.

## Routing

- Exact value found and maps cleanly to a code → **back to** `11-kode-segmen.md`; count by code (cheaper, reusable).
- **Back to** `10-segmen-produk.md` for cross-dimension rules (segment × country, segment × kemasan).
