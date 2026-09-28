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

## Company population — master or via products?

The default for "how many companies of scale X / produsen / importir" is the **trader master** (`m_trader_rla`), WITHOUT joining the product tables. Join to products only when the question says "yang punya produk/NIE". When both readings are live, show both labelled ("terdaftar: X · punya produk: Y") — the master figure **must** appear.

**The produsen/importir status flags are not available in this repo.** `m_trader_rla` carries no `is_status_industri_*` columns, so there is no produsen-vs-importir company flag to count — say so rather than substituting a product-side column. `status_usaha` (`31` produsen · `33` importir) on the product tables counts **PRODUCTS**, not companies — use it only when the subject really is products.

⚠️ **When several flags describe one concept, the roles do not cancel each other out.** Count each flag independently, never with a single `CASE WHEN` chain — a `CASE … WHEN … THEN … END` chain gives rows that satisfy two flags **only to the first branch**, silently undercounting the rest. The tell: several boolean-like columns sitting side by side for one concept. This applies generally to every `'1'`/`'0'` flag column in this database.

## Business field vs industry scale — two different things

| Concept | Column | Notes |
|---|---|---|
| **Bidang usaha (KBLI)** | **not available here** | no `kode_kbli` column exists in this repo's tables — a KBLI / "industri apa" question is out of scope; say so instead of answering with scale |
| **Skala industri** | `m_trader_rla.skala_industri` | `1` mikro · `2` kecil · `3` menengah · `4` besar; UMKM = 1+2+3 |

"Industri apa yang paling banyak mendaftarkan" means **bidang usaha (KBLI)**, not scale. Here KBLI cannot be read at all, so state the limit; when the question is genuinely about scale, answer with `skala_industri`. When unsure which is meant, both are valid answers to different questions → Gate 1, ask.

**Scale of WHAT decides the query shape** (panel dimension "Skala Industri", `05-filter-katalog.md`):

- The question counts **products / NIEs in a period** ("persetujuan produk skala Mikro", "struktur
  skala industri 2025") → **JOIN** `m_trader_rla t ON t.trader_id = p.trader_id`
  and filter `t.skala_industri='1'` — the count stays `COUNT(DISTINCT p.nomor)` on the product
  table with the decision-table branch (`00-menghitung.md` §1).
- The question counts **companies** ("berapa perusahaan mikro terdaftar") → count the **master**
  directly: `SELECT skala_industri, COUNT(DISTINCT trader_id) FROM m_trader_rla …`.
  `COUNT(*)` over the master without a product join is the wrong population for any product/period
  question — companies ≠ registrations, and the master has no period column to filter.

**Empty scale means Importir.** It is stored as an empty string/NULL — always `COALESCE(NULLIF(TRIM(col::text),''),'Importir')`, never `GROUP BY` the raw column, or the "Importir" category splits into two distinct empty-looking rows.

## Regions — three columns that do NOT behave alike

`daerah_trader` · `daerah_pabrik` · `daerah_produsen` · `kotakab_id`. The dictionary stores dotted forms (`31.75`), the column stores them **without dots** (`3175`) — join with `REPLACE(kode,'.','')`. The category only holds kabupaten/kota, so **`provinsi_id` has no rows of its own** and some `kotakab_id` values fall outside it — report unmapped regions rather than discarding them.

**Which column answers the question — pick by entity, then by MD/ML:**

| Question | Column | Applies to |
|---|---|---|
| Persetujuan / NIE per province | `daerah_pabrik` | domestic (MD) only |
| Permohonan / pengajuan per province | `daerah_trader` | MD and ML |

### MD / ML point of view

- `daerah_trader` is **always an Indonesian region code**, for MD *and* ML (for ML it is the importer's region) — it is the **only** region column that is valid across MD/ML.
- `daerah_pabrik` is only **meaningful for MD**; for ML its content is an import marker, not a locality.
- **In ERLA the import marker in `daerah_pabrik` is the uniform value `'9999'`** — not the string `'NULL'`, not SQL NULL, not an empty string. So `daerah_pabrik IS NOT NULL` does **not** exclude imports.
- `negara_pabrik` is the cleanest domestic/import separator (`ID` vs a foreign code) and is filled on every row — use it whenever the split matters (`60-asal-produksi.md`).

**Fill rates and what "empty" means — it differs per column:**

| Column | Whose region | What empty means |
|---|---|---|
| `daerah_trader` | the registrant | evenly filled; emptiness marks no group |
| `daerah_pabrik` | the production site | **an import marker (`'9999'`), not a missing value** |
| `daerah_produsen` | the produsen | rarely filled — only a small subset carry a value |

⚠️ **`daerah_pabrik IS NOT NULL` is a "local products only" filter disguised as data cleaning.** Placing it in a query that did not ask about regions keeps every imported product (the column is `'9999'`, never NULL) while the answer reads as if it had narrowed to domestic sites. If "local" really is the intent, the honest filter is `negara_pabrik` (`60-asal-produksi.md`) — not the region fill rate.

**For that reason, the `daerah_* <> '9999'` guard is CONDITIONAL, not mandatory.** Use it only when the question actually groups or filters by region, and only on the asked party's column (in practice `daerah_pabrik`, where `'9999'` marks an import). The exclusion-vs-narrowing test is in `00-menghitung.md` §3.

⚠️ **Empty is not unmapped.** A code that is **filled but missing from the dictionary** is a **domestic** region whose label does not exist yet — not a foreign one. What decides domestic/foreign is `negara_pabrik`; present the code as-is and note the label is unmapped — do not guess origin from the number.

## Province rankings — derive, name, and respect the 37/38 divergence

"Ranking 10 provinsi" has no province column. Derive it: **province = the 2-digit prefix of the
region code** (`left(daerah_trader,2)`; the column is always 4 digits in ERLA). Name it by
joining the full 4-digit code to the dictionary's kabupaten rows, falling back to the standard
province names. Two things the naive version gets wrong:

- **The dictionary does not know the data's #2 and #3 provinces.** Prefixes **`37` (Jawa Barat)
  and `38` (Jawa Timur)** dominate the data but exist in no
  dictionary row, while the dictionary's `32`/`35` kabupaten rows hold no data. The kabupaten
  series and volumes identify them (3701≈Bogor, 3716≈Bekasi, 3878≈Surabaya, 3815≈Sidoarjo).
  Name them with the divergence stated — dropping them or calling them "unmapped" turns the
  #2/#3 provinces into a wrong answer.
- **Pick the region column by the party asked.** A province ranking of **permohonan / pengajuan**
  reads `daerah_trader` (MD and ML); **persetujuan / NIE** reads `daerah_pabrik` and is meaningful
  for domestic (MD) only — see the table above.

Full recipe, the worked "Mikro × Tinggi" combination, and the panel's filter mapping live in
`05-filter-katalog.md`.

## Routing

- Country of origin / import / local mentioned → **see** `60-asal-produksi.md`
- Product segments mentioned ("perusahaan yang mendaftarkan kopi instan") → **see** `10-segmen-produk.md`, then AND both into one WHERE
- Entity: perusahaan → `trader_id` (`00-menghitung.md` §1)
- Another party/region column not covered here → `95-dimensi-lain.md`
