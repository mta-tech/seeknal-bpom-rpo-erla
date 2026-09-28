# The User Filter Panel — mapping filter labels to columns, codes, and pages

Users pick filters from a panel ("Filter Data Berdasarkan"), not from column names. This page
maps every panel dimension to its warehouse column, its codes, and the page that owns it.
Panel labels always carry the code in parentheses — "Sosis Daging (0809)" — so extract the code
when it is shown; the catalog below covers the cases where it is not.

## Date filters — four panel names, three warehouse columns

| Panel label | Column | Notes |
|---|---|---|
| Tanggal Permohonan | `tanggal_aju` | submission date; the earliest event |
| Tanggal Bayar SPB | `tanggal_bayar` | the canonical permohonan date (`15-permohonan.md`) |
| Tanggal Terbit NIE | `tanggal` | issuance; filled only for issued rows |
| Tanggal Terbit SPB | **does not exist** | legacy-panel only; the warehouse has no such column — do not invent one, offer `tanggal_bayar` or `tanggal` and say which |

`tanggal_berkas` / `tanggal_diambil` are processing dates and never count (`80-waktu-periode.md`).

## The twelve filter dimensions

| Panel dimension | Column(s) | Codes / values | Details |
|---|---|---|---|
| Jenis Permohonan | `jenis_permohonan` | `301` baru · `302` mayor · `303` minor · `304` daftar ulang · `305` notifikasi | `15-permohonan.md` |
| Jenis Kemasan | `kemasan_id` → `sub_kemasan_id` | parent 1–7 (ERBA) / 31–39 (ERLA); 37 child codes | `40-kemasan.md` · `41-sub-kemasan.md` |
| Risiko Penilaian | `kategori_dokumen` | `301` Tinggi · `302` Menengah Tinggi · `303` Menengah Rendah (MR) · `304` Tinggi Notifikasi — ERBA schema | `30-risiko-komitmen.md` |
| Kota/Kabupaten | `daerah_trader` / `daerah_pabrik` / `kotakab_id` | 4-digit undotted codes; dictionary stores dotted `31.75` — join with `REPLACE(kode,'.','')` | `50-pihak-wilayah.md` |
| Nama Pabrik | `nama_pabrik` | free text; the factory is NOT the registrant | `50-pihak-wilayah.md` |
| Status Produk | `status_produk` | **exactly 5 codes exist**: `301` Diproduksi Sendiri · `302` Impor · `304` Berdasarkan Kontrak · `306` Single MD Induk · `307` Single MD Anak | `60-asal-produksi.md` |
| Negara Pabrik | `negara_pabrik` | 2-letter ISO; same codes both systems; empty ≈ domestic for trader side | `60-asal-produksi.md` |
| Provinsi Pabrik | `left(daerah_pabrik,2)` | derived, not stored — province recipe & 37/38 divergence on the region page | `50-pihak-wilayah.md` |
| Skala Industri | `m_trader_rba.skala_industri_id` | `1` Mikro/IRTP · `2` Kecil · `3` Menengah · `4` Besar; **ERBA-only** (the ERLA master has no such column) | `50-pihak-wilayah.md` |
| Nama Perusahaan | `nama_trader` | normalise `PT.` prefixes before cross-system ranking | `50-pihak-wilayah.md` |
| Status Perusahaan | `is_status_industri_produsen` / `..._importir` | independent TEXT flags `'1'`/`'0'`, ERBA-only; `status_usaha` `31`/`33` on products counts PRODUCTS | `50-pihak-wilayah.md` |
| Jenis Pangan | `jenis_pangan` / `kategori_pangan` | panel catalog below | `10`/`11`/`12` + this page |

**Panel labels that exist in the panel but have NO code in the warehouse** — answer honestly
rather than guessing: Status Produk **Lisensi**, **Pengemas Kembali**, and both **Notifikasi**
variants (the dictionary and the data carry only the five codes above).

## The province recipe — owned by `50-pihak-wilayah.md`

Province is **derived, not stored**: the 2-digit prefix of the region code
(`left(daerah_trader,2)`). Naming, the ranking recipe, and the **37/38 divergence** (the data's
#2/#3 "provinces" — Jawa Barat and Jawa Timur — are absent from the dictionary while its 32/35
rows hold no data) are documented once, on the region page. Filter-specific note only: the
panel's "Provinsi Pabrik" reads the **factory** side (`daerah_pabrik`), whose emptiness means
"factory abroad" (`00-menghitung.md` §3).

## Jenis pangan — panel catalog and multi-select

**The catalog below covers the PANEL, not the universe.** The panel lists ≈90 labelled entries;
the data carries **212 `jenis_pangan` codes in ERBA and 304 in ERLA**. So: a question naming a
panel label resolves through this catalog; a question naming a code in parentheses
("Sosis Daging (0809)") resolves by extracting the code; anything beyond the catalog resolves
through the `nama_kategori` probe on both systems (P4, `12-nama-kategori.md`). The warehouse
has no label table for these codes — `nama_kategori` is empty for most of them — so the panel
itself is the only label source for the entries it carries.

**When `data_dictionary` applies — and when it never does.** `jenis_pangan` and
`kategori_pangan` have **no category in `data_dictionary`** (23 categories exist, none for food
codes). A dictionary lookup for a food code is therefore always the wrong path — it returns 0
rows and says nothing. The dictionary serves the *other* coded columns (`STATUS`,
`STATUS_KOMITMEN`, `KLASIFIKASI_ID`, `JENIS_BTP`, regions, …). Resolution order for food codes:
1. the code in parentheses in the question;
2. the panel catalog below;
3. a verified anchor in `11-kode-segmen.md`;
4. the `nama_kategori` probe, both systems (P4).

ERLA shares none of the ERBA codes — the panel catalog is the **ERBA namespace**; on ERLA
resolve through `nama_kategori` instead.

- **Susu & olahan (01xx/02xx):** 0101 Bubuk whey & produknya · 0102 Buttermilk (plain) pasteurisasi · 0103 Buttermilk bubuk · 0104 Cairan whey · 0105 Es krim · 0106 Es susu · 0131/0132 Susu dan krim bubuk dari bahan baku cair (panel lists the same label twice) · 0201 ColdPressedOils · 0202 Non-DairyWhippedCream · 0206 ButterOilSubstitute · 0207 Suet
- **Buah (04xx):** 0401 KulitBuahBergula · 0402 KoktilBuahDalamKemasan · 0403 TepungBuah · 0404 ToppingBuah · 0405 CincauHitam · 0406 EmpingMelinjo · 0409 Dodol/LempokBuah · 0410 ManisanBuah · 0411 WajitBuah · 0420 WortelDalamKemasan · 0421 TomatDalamKemasan · 0422 SayurAsin
- **Kakao & gula (05xx/11xx):** 0501 Truffles · 0510 SirupCokelat · 0512 CokelatPasta · 0513 CBS Non-Lauric · 0514 Saus/OlesanCokelat · 1111 TetesTebu/Molases
- **Bakeri & olahan biji (06xx/07xx/15xx):** 0601 Sorgum · 0602 LembagaGandum · 0603 BuburInstan · 0604 DegermedMaizeMeal · 0605 Dekstrin · 0607 Wajik/Wajit · 0612 WholeMaizeMeal · 0614 Pati/Protein Olahan (Meat/Fish/Seafood Analog) · 0615 SohunLainnya · 0620 Tempe · 0702 Mantao · 0703 WaterBiscuit · 0704 TepungRoti/BreadCrumb · 0705 Adonan/produk bakeri beku · 1502 Slondok
- **Daging & unggas (08xx):** 0801 DagingdalamKaleng · 0803 UsusAyamGoreng · 0804 PrimeRib · 0805 SosisCina/LupCheong · 0806 Urutan · 0809 Sosis Daging
- **Perikanan (09xx):** 0902 TiramDalamKemasan · 0904 UdangKeringTanpaKulit(Ebi) · 0905 UdangSegar · 0908 StikKepitingAnalog (pasteurisasi)
- **Telur & lain (10xx):** 1001 TepungCustard · 1006 TelurOlahanSteril
- **Bumbu & saus (12xx):** 1201 ArakMasak · 1202 RagiKering · 1203 CukaPengenceranAsamAsetat · 1206 SausTarTar · 1209 Wijen · 1217 RempahBubuk · 1219 SausTopping/SausSiram · 1226 Campuran Saus, Gravies, Dressing
- **Minuman (14xx):** 1401 AMDK/Mineral/DemineralBeroksigen · 1402 AirMinumpHTinggi · 1403 Air Soda · 1407 MinumanBerkafeinFormulasi · 1408 Whisky · 1410 SodaKrim · 1412 Minuman ringan non-karbonasi · 1417 SariWortel

Rules:
- **Multi-select is an `IN` list.** "Sosis Daging dan Udang Segar" = `jenis_pangan IN ('0809','0905')` — never two queries OR-ed in the answer, and never one value only.
- **Thematic families** ("yang mengandung susu") have no single code. Resolve them by probing
  `nama_kategori` (regex like `susu|whey|buttermilk|krim`) and reading which `jenis_pangan`
  codes carry the matches — today that yields 0101–0106, 0112–0118, 0122, 0128, 0129, 0131, 0132,
  0202, 0301, 0501, 0619, 0703, 1310, 1414, among others. State the resolved code set in the
  answer; the panel has no "susu" entry to click.
- **Formula pairs are cair/padat variants, not duplicates:** 1301/1302 Formula Bayi, 1303/1304
  Formula Lanjutan, 1305/1306 Formula Pertumbuhan. For "formula bayi" strictly, `11-kode-segmen.md`
  anchors to `jenis_pangan IN ('1301','1302')` (ERBA).
- **Panel entries `Antibuih (44)`, `Antioksidan (23)`, `BTP Campuran (48)` are JENIS_BTP codes** —
  they live in the BTP tables, not in `jenis_pangan`. Route to `70-btp.md`.

### Kategori pangan — the ANAK level

When the question says **kategori pangan** rather than jenis pangan, decide which level it
means: a family ("pangan bayi & anak") stays at INDUK (`jenis_pangan`, or `klasifikasi_id='311'`
for the official class), while a concrete variant ("garam beryodium", a sub-type of a family) is
the ANAK level (`kategori_pangan`). Both are absent from `data_dictionary` — same resolution
order as above.

- **The namespaces differ in shape, not just values**: ERBA `kategori_pangan` is 12-digit
  (1.168 distinct codes), ERLA mixes 2–10 digit (1.557 distinct). Nothing beyond the structure
  is comparable across systems — resolve each side independently.
- **Derive the children from the parent, never from memory**:
  `SELECT kategori_pangan, COUNT(*) … WHERE jenis_pangan='<induk>' GROUP BY 1`, then count with
  exact codes (or `nama_kategori` when the spread is messy). Choosing an ANAK value for a
  family-level question silently drops its siblings (`11-kode-segmen.md`).
- Entity, status tiers, and exclusions are unchanged — only the filter column moves.

## Combining filters — the worked example

"Rekomendasi area: kombinasi Skala Mikro/IRTP × Risiko Tinggi, 2025, Jenis Permohonan baru":

- skala lives only in `m_trader_rba` → **LEFT JOIN** on `trader_id`, filter `skala_industri_id='1'`;
- risiko = ERBA schema → `kategori_dokumen IN ('301','304')` (Tinggi includes Tinggi Notifikasi);
- `jenis_permohonan='301'`; period on `tanggal_bayar`; valid status set; entity `nomor`.
- ERBA-only **structurally** (the ERLA master has no scale column) — say so; group by the
  registrant's province prefix. Live 4 Sep 2026 (top): `38` 709 · `31` 429 · `37` 397 · `33` 336.

## Ingredient questions — honest gap

The legacy system joined a **`T_PRODUK_3_BAHAN`** table (ingredients: `NAMA_BAHAN`,
`JENIS_BAHAN`). **No such table exists anywhere in this warehouse** — "produk yang mengandung
bahan baku X" (e.g. akar ginseng) cannot be answered from ingredient data. Offer the honest
substitute: a **product-name search** (`nama`/`merk ILIKE '%ginseng%'` — 400 ERBA rows today),
explicitly labelled as a name search, not an ingredient analysis. Never present it as "produk
yang mengandung".

## Detail-list answers (the legacy export shape)

When the question asks for the product list, the answer mirrors the legacy export: label the
columns in the user's terms (Nomor Pengajuan = `produk_id`, NIE = `nomor`, Nama Pabrik =
`nama_pabrik`, …), show `produk_id` **and** `nomor` side by side, collapse newlines in free-text
columns (`regexp_replace(col, '\s+', ' ', 'g')`), keep the valid status set per side, and let the
CSV carry the same columns. Why one product can own several `produk_id` — and which reading
"surat keputusan" implies — is the versioning story in `15-permohonan.md`.

## Routing

- Per-dimension traps → the pages listed in the twelve-dimension table.
- Free-text segment probing → `12-nama-kategori.md`; code anchors → `11-kode-segmen.md`.
- Region/province details → `50-pihak-wilayah.md`; origin/status_produk → `60-asal-produksi.md`.
- A panel dimension not covered here → `95-dimensi-lain.md`.
