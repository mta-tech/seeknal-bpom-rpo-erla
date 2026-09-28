# Origin & Production Method

negara asal, impor, lokal, dalam negeri, makloon, single MD, any country name.

## Country of origin — two candidate columns, choose by checking fill rates

`negara_pabrik` and `negara_produsen` both exist and both sound right. **What separates them is not the name but how filled they are** — and that must be checked, not remembered:

```sql
SELECT COUNT(*) FILTER (WHERE NULLIF(TRIM(negara_pabrik),'')   IS NOT NULL) pabrik,
       COUNT(*) FILTER (WHERE NULLIF(TRIM(negara_produsen),'') IS NOT NULL) produsen,
       COUNT(*) total
FROM t_produk_3_rilis_erla;
```

Pick the widely-filled column. Using the sparsely-filled one discards most of the population and returns a far-too-small number — with no error, no warning. **Name the column you used in the answer.** `negara_pabrik` is filled on every row and is the cleanest domestic/import separator (`ID` vs a foreign code) — the default here.

The values are 2-letter ISO codes, catalogued under dictionary category `NEGARA_PABRIK dan NEGARA_PRODUSEN`. A few rows appear twice in the dictionary; the duplicates are harmless.

`ID` = Indonesia/dalam negeri · anything else = impor.

**The answer spells out country NAMES, not raw codes** — users do not read ISO. Translate through the dictionary before presenting.

## Production method — the `status_produk` column (ERLA)

The ERLA table uses **seven** `status_produk` codes. Five are documented in `data_dictionary`; two have **no dictionary row at all**:

| Code | Meaning | Source |
|---|---|---|
| `301` | Diproduksi Sendiri | dictionary |
| `302` | Impor | dictionary |
| `303` | *(unlabelled)* | **no dictionary row** |
| `304` | Berdasarkan Kontrak | dictionary |
| `305` | *(unlabelled)* | **no dictionary row** |
| `306` | Single MD Induk | dictionary |
| `307` | Single MD Anak | dictionary |

⚠️ **The panel lists more labels than the data carries, and its labels for `303` and `305` are "Berdasarkan Kontrak Notifikasi" and "Diproduksi Sendiri Notifikasi".** Those labels come from the **panel**, not from `data_dictionary` — the code→label mapping for `303`/`305` is **not verified** against the data. Report them as panel labels and say the mapping is unverified; do not present them as a data fact.

⚠️ **The remaining two panel labels — "Lisensi" and "Pengemas Kembali" — carry no code and no rows in the data.** No `status_produk` value in `t_produk_3_rilis_erla` maps to them. If a question names one of those two, say the class is not represented in the data; never invent a code for it.

**The "Diproduksi Sendiri" family in ERLA = `301` + `303` + `305`.** A question asking for products "Diproduksi Sendiri" in this repo therefore filters `status_produk IN ('301','303','305')` — name the three components when you answer.

**If the question does not name a product status / jenis permohonan, do NOT filter on `status_produk` at all** — count the entity as it stands. Adding the filter unasked silently narrows the population.

**Single MD Anak:** one company with a single parent sets up new manufacturing elsewhere; the NIE is the same. Without an explicit `status_produk` filter it is treated the same as its parent.

**`negara_pabrik` is the origin determinant, not `status_produk`.** "Asal Indonesia" is decided by the **factory's location**, not the production status. Using `status_produk <> '302'` as a proxy for "not imported" captures a different, wider population. When the question asks about the **production method** (makloon, single MD, produsen sendiri), `status_produk` is the column; when it asks about **origin**, use `negara_pabrik`.

**The panel's "Status Produk" filter reads THIS column** (`05-filter-katalog.md`): Impor = `302` ·
Berdasarkan Kontrak = `304` · Diproduksi Sendiri = `301` (with its `303`/`305` family) · Single MD Induk/Anak = `306`/`307`.
A question naming those panel labels — including **"Impor"** and **"Berdasarkan Kontrak"** — is a
production-method question: filter `status_produk`, even though "impor" also sounds like an origin
word. "Produk impor yang disetujui" therefore = `status_produk='302'` + the approval branch of the
decision table (`00-menghitung.md` §1), **not** `negara_pabrik <> 'ID'`.

## Routing

- Origin combined with a product segment ("kopi instan asal Indonesia") → **see** `10-segmen-produk.md`; resolve each part in its own column, then AND them in ONE WHERE — do not drop either.
- Factories/companies from a specific country → **see** `50-pihak-wilayah.md`
- Country code unknown → dictionary category `NEGARA_PABRIK dan NEGARA_PRODUSEN`,
  `deskripsi ILIKE '%<country name>%'` WITHIN that category (P3 path, Gate 2).
