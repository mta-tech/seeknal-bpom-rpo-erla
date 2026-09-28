# Parties & Regions

perusahaan, pendaftar, pabrik, produsen, importir, industri, KBLI, skala, daerah, provinsi.

## Three different parties — do not confuse them

| Party | Column | Meaning |
|---|---|---|
| **Pendaftar** | `nama_trader` / `trader_id` | the company that REGISTERED the izin edar |
| **Pabrik** | `nama_pabrik` | where the product is MADE |
| **Produsen** | `nama_produsen` / `produsen_id` | check its fill rate before using — often nearly empty |

**Ranking all three gives different results, not similar ones.** Brand owners commission production to third-party factories (the makloon pattern) — that is why pendaftar and pabrik are separate in the first place. Answering a factory question with `nama_trader` yields a completely different list of names, even though the numbers look plausible.

To confirm which column is meant, run the ranking on both columns once and compare the names. If they differ, the question really does distinguish them — name the column you used.

Foreign factories count too, and that is correct when the question does not restrict the country. Restricting to domestic requires `negara_pabrik` → `60-asal-produksi.md`.

## Names are formatted DIFFERENTLY across systems — check, then normalise

One system stores company names **with** the legal-form prefix ("PT. …"), the other **without** it. The same company therefore appears as two separate entries when combined.

Check first; do not assume the direction:
```sql
SELECT '<system>' sys, COUNT(*) FILTER (WHERE nama_trader ~* '^PT[. ]') berprefiks,
       COUNT(*) total FROM (SELECT DISTINCT nama_trader FROM <that table>) x;
```

If the two sides differ in pattern, normalise **before** any cross-system `GROUP BY`:
```sql
regexp_replace(upper(btrim(nama_trader)), '^PT\.?\s+', '')
```

Without normalisation, one company splits in two and its ranking is **mis-ordered** — not just untidy, because a split entry can lose to an unsplit one.

**`trader_id` does not work across systems.** The same company carries a different id in each system — the id belongs to the system, not to the company. `COUNT(DISTINCT trader_id)` over a UNION therefore counts every company **twice**. Within ONE system, the company entity is always `trader_id`; do not dedupe by name within a single system (names collide across branches). Dedupe by name **only** for the cross-system combined headline, and still show the per-system `trader_id` counts as labelled rows alongside it.

## Company population — master or via products?

The default for "how many companies of scale X / produsen / importir" is the **trader master** (`m_trader_rba` / `m_trader_rla`), WITHOUT joining the product tables. Join to products only when the question says "yang punya produk/NIE". When both readings are live, show both labelled ("terdaftar: X · punya produk: Y") — the master figure **must** appear.

`is_status_industri_produsen` / `is_status_industri_importir` in `m_trader_rba` are TEXT `'1'`/`'0'` (not boolean), entity `trader_id`. **Structurally ERBA-only** — `m_trader_rla` has no such columns, so there is no ERLA figure and no combined figure. State that limit.

⚠️ **The roles do not cancel each other out.** One company can be a produsen **and** an importir at the same time — these are two independent flags, not one categorical column.

```sql
COUNT(*) FILTER (WHERE is_status_industri_produsen='1') AS produsen,   -- CORRECT
COUNT(*) FILTER (WHERE is_status_industri_importir='1')  AS importir
```
```sql
CASE WHEN is_status_industri_produsen='1' THEN 'Produsen'              -- WRONG
     WHEN is_status_industri_importir='1'  THEN 'Importir' END
```

The `CASE WHEN` chain gives dual-role rows **only to the first branch**, silently undercounting the second group. If totals matter, say that the per-role counts exceed the company count because of overlap — do not "tidy" it by making the roles exclusive.

**This applies generally to every `'1'`/`'0'` flag column** in this database, not just these two. The tell: several boolean columns sitting side by side for one concept. Count each flag independently. `status_usaha` (`31` produsen · `33` importir) on the product tables counts **PRODUCTS**, not companies — use it only when the subject really is products.

## Business field vs industry scale — two different things

| Concept | Column | Notes |
|---|---|---|
| **Bidang usaha (KBLI)** | `kode_kbli` on **`t_produk_3_erba` / `t_btp_3_erba`** | the trader tables have no such column — querying them errors. Counts PRODUCTS per business field; to count companies, aggregate to `trader_id` and say so |
| **Skala industri** | `m_trader_rba.skala_industri_id` / `m_trader_rla.skala_industri` (different column names) | `1` mikro · `2` kecil · `3` menengah · `4` besar; UMKM = 1+2+3 |

"Industri apa yang paling banyak mendaftarkan" means **bidang usaha (KBLI)**, not scale. Answering with scale gives "Besar/Menengah/Kecil" — the right categories for a different question. When unsure which is meant, both are valid answers to different questions → Gate 1, ask.

**Scale of WHAT decides the query shape** (panel dimension "Skala Industri", `05-filter-katalog.md`):

- The question counts **products / NIEs in a period** ("persetujuan produk skala Mikro", "struktur
  skala industri 2025") → **JOIN** `m_trader_rba t ON t.trader_id::bigint = p.trader_id::bigint`
  and filter `t.skala_industri_id='1'` — the count stays `COUNT(DISTINCT p.nomor)` on the product
  table with the decision-table branch (`00-menghitung.md` §1).
- The question counts **companies** ("berapa perusahaan mikro terdaftar") → count the **master**
  directly: `SELECT skala_industri_id, COUNT(DISTINCT trader_id) FROM m_trader_rba …`.
  `COUNT(*)` over the master without a product join is the wrong population for any product/period
  question — companies ≠ registrations, and the master has no period column to filter.

**KBLI sentinel.** `kode_kbli` stores `'0'` as an unfilled marker — not in the dictionary and not a business field, yet it can **top the ranking**. Recognise sentinels by their nature (no dictionary row, or an empty/`-` description), not by their count size. Exclude `'0'` and `''` from rankings; report them separately as a data-quality note.

**Empty scale means Importir**, and it is stored differently per system (one uses a space, the other an empty string/NULL). Always `COALESCE(NULLIF(TRIM(col::text),''),'Importir')` — never `GROUP BY` the raw column, or the "Importir" category splits into two distinct empty-looking rows.

## Regions — three columns that do NOT behave alike

`daerah_trader` · `daerah_pabrik` · `daerah_produsen` · `kotakab_id`. The dictionary stores dotted forms (`31.75`), the column stores them **without dots** (`3175`) — join with `REPLACE(kode,'.','')`. The category only holds kabupaten/kota, so **`provinsi_id` has no rows of its own** and some `kotakab_id` values fall outside it — report unmapped regions rather than discarding them.

**Choose the column from the party being asked about**, and know what empty means — it differs for each:

| Column | Whose region | What empty means |
|---|---|---|
| `daerah_trader` | the registrant | evenly filled; emptiness marks no group |
| `daerah_pabrik` | the production site | **empty exactly when the factory is abroad** |
| `daerah_produsen` | the produsen | rarely filled in either group — most of the population has no value |

⚠️ **`daerah_pabrik IS NOT NULL` is a "local products only" filter disguised as data cleaning.** Placing it in a query that did not ask about regions discards every imported product with no error and no trace in the answer sentence. If "local" really is the intent, the honest filter is `negara_pabrik` (`60-asal-produksi.md`) — not the region fill rate.

**For that reason, `daerah_* <> 'NULL' / <> '9999'` guards are CONDITIONAL, not mandatory.** Use them only when the question actually groups or filters by region, and only on the asked party's column. The exclusion-vs-narrowing test is in `00-menghitung.md` §3.

⚠️ **Empty is not unmapped.** An empty column means the factory is abroad. A code that is **filled but missing from the dictionary** is a **domestic** region whose label does not exist yet — not a foreign one. Folding both into one "Luar Negeri" basket inverts the meaning of the rows. What decides domestic/foreign is `negara_pabrik`; present the code as-is and note the label is unmapped — do not guess origin from the number.

## Province rankings — derive, name, and respect the 37/38 divergence

"Ranking 10 provinsi" has no province column. Derive it: **province = the 2-digit prefix of the
region code** (`left(daerah_trader,2)`; the column is always 4 digits in ERBA). Name it by
joining the full 4-digit code to the dictionary's kabupaten rows, falling back to the standard
province names. Two things the naive version gets wrong:

- **The dictionary does not know the data's #2 and #3 provinces.** Prefixes **`37` (Jawa Barat)
  and `38` (Jawa Timur)** dominate the data (21.654 / 18.289 registered NIEs) but exist in no
  dictionary row, while the dictionary's `32`/`35` kabupaten rows hold no data. The kabupaten
  series and volumes identify them (3701≈Bogor, 3716≈Bekasi, 3878≈Surabaya, 3815≈Sidoarjo).
  Name them with the divergence stated — dropping them or calling them "unmapped" turns the
  #2/#3 provinces into a wrong answer.
- **Pick the region column by the party asked** (registrant vs factory — see the table above);
  a province ranking of "persetujuan" reads `daerah_trader` unless the question says the factory
  side.

Full recipe, the worked "Mikro × Tinggi" combination, and the panel's filter mapping live in
`05-filter-katalog.md`.

## Routing

- Country of origin / import / local mentioned → **see** `60-asal-produksi.md`
- Product segments mentioned ("perusahaan yang mendaftarkan kopi instan") → **see** `10-segmen-produk.md`, then AND both into one WHERE
- Entity: perusahaan → `trader_id` **within one system**; across systems → normalised name (`00-menghitung.md` §1)
- Another party/region column not covered here → `95-dimensi-lain.md`
