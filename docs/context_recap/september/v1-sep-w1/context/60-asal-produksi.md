# Origin & Production Method

negara asal, impor, lokal, dalam negeri, makloon, single MD, any country name.

## Country of origin — two candidate columns, choose by checking fill rates

`negara_pabrik` and `negara_produsen` both exist and both sound right. **What separates them is not the name but how filled they are** — and that must be checked, not remembered:

```sql
SELECT '<system>' sys,
       COUNT(*) FILTER (WHERE NULLIF(TRIM(negara_pabrik),'')   IS NOT NULL) pabrik,
       COUNT(*) FILTER (WHERE NULLIF(TRIM(negara_produsen),'') IS NOT NULL) produsen,
       COUNT(*) total
FROM <that table>;
```

Pick the widely-filled column. Using the sparsely-filled one discards most of the population and returns a far-too-small number — with no error, no warning. **Name the column you used in the answer.** If each system turns out to use a different column for the same concept, that is not a reason to combine them in one expression — resolve each side against its own column and say so.

The values are 2-letter ISO codes. Dictionary category `NEGARA_PABRIK dan NEGARA_PRODUSEN`, `sumber` "ERLA dan ERBA" — **the same codes work in both systems**; one of the few columns where that holds, so do not generalise it. A few rows appear twice in the dictionary; the duplicates are harmless.

`ID` = Indonesia/dalam negeri · anything else = impor.

**The answer spells out country NAMES, not raw codes** — users do not read ISO. Translate through the dictionary before presenting.

## Production method — the `status_produk` column

| Code | Meaning |
|---|---|
| `301` | Produsen sendiri |
| `302` | Impor |
| `304` | Makloon (kontrak) |
| `306` | Single MD Induk |
| `307` | Single MD Anak |

Catalogued as ERBA-only, **but ERLA fills it too with the same meanings**, plus `303` and `305` which no dictionary row describes. Before reporting 0 or "none" for one system, list that system's own values first.

⚠️ **This is a DIFFERENT case from `kategori_dokumen`** (`30-risiko-komitmen.md`), and the difference decides whether a UNION is allowed. There, the column is catalogued ERBA and the ERLA values are **a different schema**; here, the column is catalogued ERBA and the ERLA values are **the same schema**. From `information_schema` and from fill rates, the two look identical.

**How to tell them apart: cross-check against an independent column that should agree.** `status_produk` `302` Impor should agree with non-Indonesian `negara_pabrik` — if the two sets coincide, the code really does mean what it says on that side. When no independent column exists to cross-check (as with risk), **assume different schemas and do not UNION** — a wrong assumption in this direction only narrows the answer; in the other direction it makes it wrong.

**`status_produk` is not a substitute for `negara_pabrik`.** "Asal Indonesia" is decided by the **factory's location**, not the production status. Using `status_produk <> '302'` as a proxy for "not imported" captures a different, wider population. When the question asks about the **production method** (makloon, single MD, produsen sendiri), `status_produk` is the column; when it asks about **origin**, use the country column.

## Scope is often lopsided across systems — check, then split

The legacy system holds years of registration history while the newer one started later, so many segments — imports especially — are uneven. Some segments are **structurally single-system**: zero rows on one side.

Prove it from the data before labelling an answer "gabungan":

```sql
SELECT 'ERBA' sys, COUNT(DISTINCT nomor) FROM t_produk_3_erba WHERE <filter>
UNION ALL
SELECT 'ERLA', COUNT(DISTINCT nomor) FROM t_produk_3_rilis_erla WHERE <ERLA-side filter>;
```

- One side is zero → say the entire figure comes from the other side. Running the UNION still gives the right number, but calling it "gabungan" without a note makes the user believe both systems contributed.
- Heavily uneven → present the split, not just the combined figure; the imbalance itself is often the information being sought (a system migration, not a market change).

**Prove the scope from the data; never assume "gabungan" is automatically right.**

## Routing

- Origin combined with a product segment ("kopi instan asal Indonesia") → **see** `10-segmen-produk.md`; resolve each part in its own column, then AND them in ONE WHERE — do not drop either.
- Factories/companies from a specific country → **see** `50-pihak-wilayah.md`
- Country code unknown → dictionary category `NEGARA_PABRIK dan NEGARA_PRODUSEN`,
  `deskripsi ILIKE '%<country name>%'` WITHIN that category (P3 path, Gate 2).
