# Varian `september/v3-sep-w1` — Perbaikan Kegagalan Sweep + Netralisasi Klarifikasi

**Dibuat:** 17 September 2026 · **Status:** salinan utuh `v2-sep-w1` + 9 patch hasil audit
kegagalan 14–15 Sep (lihat `docs/audit_context/2026-09-15-v2-sep-w1-compact-sweep/07-RINCIAN-SQL-VS-NOTE-170926.md`).
**Induk:** `september/v2-sep-w1` — disalin utuh (context, skills, ASK, agent.yml), symlink
`seeknal/skills` dipertahankan.

## Apa yang baru di varian ini (delta vs v2)

1. **`00-menghitung.md` + `15-permohonan.md`** — koreksi fakta: revisi tidak mengubah NIE,
   tapi registrasi baru (301/305) dapat menerbitkan NIE tambahan, sehingga satu produk bisa
   memegang beberapa NIE; tersedia aturan pemilihan NIE untuk pertanyaan per-produk.
   Predikat baru: "diperbarui dengan nomor baru" = `status='9999'`, bukan Daftar Ulang 304.
2. **`05-filter-katalog.md`** — keluarga tematik dihitung SEKALI dengan kondisi `nama_kategori`
   yang dipakai saat probe; dilarang konversi ke IN-list `jenis_pangan` karena kode membawa
   banyak nama di luar keluarga.
3. **`11-kode-segmen.md`** — entri panel ber-kode = namespace ERBA; dilarang menyusun
   padanan ERLA dari nama.
4. **`70-btp.md`** — kode dictionary bersifat mengikat; anomali nama dilaporkan ke user,
   kode tidak diganti.
5. **`SEEKNAL_ASK.md` Gate 1** — opsi scope kini NETRAL (Gabungan · ERBA · ERLA) tanpa
   rekomendasi; user yang memilih. (Penanda "recommended" hanya untuk skrip test otomatis.)
6. **`skills/bpom-analyst/SKILL.md` (6.0.1 → 6.1.0)** — closing pair `visualize_chart` +
   `upload_to_s3` dipertegas sebagai aturan yang paling sering dilewati; Gate 5 kini enam
   item: cek SQL-vs-kalimat (entity & tanggal), wajib menyebut limitasi yang pernah
   diklarifikasi, pertanyaan "kenapa/terus naik" wajib uji premis dengan data; opsi
   klarifikasi wajib berasal dari halaman kanon.

Prinsip penulisan varian ini: context mengajarkan **metode dan predikat** dalam bentuk
naratif — tanpa angka hasil hitung, persentase, atau bukti point-in-time. Angka aktual
milik database saat pertanyaan dijawab, bukan milik halaman context.

Belum diubah (menunggu): respon KeyCenter untuk 14 pertanyaan definisi entitas
(`docs/data_docs/definisi-entitas-keycenter.md`) dan refresh note YAML July-era ke kanon v2
(sisi test, bukan sisi context).

---

# Varian `september/v2-sep-w1` — Integrasi Regulasi (PerBPOM 10/2026)

**Dibuat:** 9 September 2026 · **Status:** chunk regulasi terpasang, belum dijalankan
terhadap suite.
**Induk:** `september/v1-sep-w1` — disalin utuh (context, skills, ASK, agent.yml), checkpoint
runtime tidak disalin.

## Apa yang baru di varian ini (delta vs v1)

Varian ini menambahkan **lapisan regulasi** dengan pola: `SEEKNAL_ASK` (pemicu) → skill
`regulasi` (peta + kontrak jawaban) → `context/regulasi/regulasipangan/*` (angka acuan).

1. **Chunk context** `context/regulasi/regulasipangan/` — `00-peta` (indeks + cakupan jujur),
   `01-definisi`, `02-kewajiban-pengecualian` (Pasal 2–4), `03-takaran-saji-aturan`
   (Pasal 8–11), `04a-takaran-saji-susu-es` (Lampiran II 01.7/02.4/03.0 — dasar cross-check
   gelato), `04e-takaran-saji-minuman` (14.1.4.1 = 100–250 ml), `05-nutri-level`
   (Pasal 27–29 + ambang Lampiran V), `06-pilihan-lebih-sehat` (profil gizi: minuman ≤6 g/
   100 ml, es krim 17/10), `07-peralihan-pengkajian` (Pasal 30–34, transisi 24 bulan).
   Semua angka diverifikasi langsung dari PDF regulasi; topik yang belum dipecah
   dinyatakan eksplisit di `00-peta.md` — agent dilarang menebak.
2. **Skill baru** `skills/regulasi/SKILL.md` — peta topik → chunk + answer contract:
   pertanyaan murni regulasi = jawaban definisional (kutip pasal, tanpa SQL/chart);
   regulasi + produk nyata = gabung `bpom-analyst` (nilai acuan × fakta SQL); cross-check
   klasifikasi menghasilkan *flag kandidat*, bukan vonis.
3. **SEEKNAL_ASK.md** — baris skill `regulasi` di tabel Skills, satu baris Page Map
   (pemicu regulasi → `load_skill('regulasi')`), dan paragraf Gate 0 yang menegaskan
   pertanyaan regulasi murni dijawab tanpa SQL.
4. Simbolik `seeknal/skills → ../skills` dari v1 tetap berlaku, jadi skill otomatis
   terbaca oleh engine.

## Catatan

- Kemasan (PerBPOM 11/2026) sengaja belum masuk — fokus isi pangan olahan.
- Cross-check komposisi (susu) **belum bisa dieksekusi live**: skema ERBA/ERLA saat ini
  tidak punya kolom komposisi/bahan. Chunk dan skill sudah menuliskan perilaku jujurnya
  (metodologi tanpa angka karangan).
- Pengujian: tambahkan skenario `REG-*` ke `test_variant_compare.py` (contoh assert:
  "100–250 ml" untuk takaran soda; jawaban gelato-tanpa-susu memakai istilah kandidat
  ketidaksesuaian 01.7 vs 02.4).

---

# Varian `september/v1-sep-w1` — Language Rewrite (Natural English)

**Dibuat:** 4 September 2026 · **Status:** rewrite bahasa, belum dijalankan terhadap suite.
**Induk:** `after-chart-route/route-context-070826-v6` — disalin utuh, lalu **seluruh dokumen
naratif di-rewrite** dengan gaya natural.

## Apa yang berubah di varian ini

Varian ini adalah **rewrite gaya bahasa**, bukan perubahan substansi. Semua aturan, angka, kode,
filter, tabel, SQL, dan urutan gate dipertahankan persis dari v6. Yang berubah adalah cara
penyampaiannya: dari gaya teknis-kaku yang penuh kapitalisasi drastis (`SETIAP`, `TERLARANG`,
`DILEPAS`, `MENYALA`, `PUTUSKAN SEKALI`) menjadi dokumentasi teknis natural yang ditulis seperti
engineer — tetap tegas pada hard rule, tetapi tidak berbunyi seperti perintah mesin.

Prinsip rewrite yang dipakai:

| Gaya lama | Gaya baru |
|---|---|
| `SETIAP`, `SATU keputusan`, `TERLARANG` | kalimat normal; "Do not use…" untuk larangan |
| `DILEPAS` | "drop the status filter" |
| `SEBERANG` / `TURUN` / `KEMBALI` (blok Rute) | blok **Routing**: "see / continue to / back to" |
| `Jangan pernah` | "Common mistakes to avoid" / "Do not…" |
| `Menyala hanya bila…` | "This applies only when…" |
| `nol irisan`, `murah`, `menggelembungkan` | "no overlap", "inexpensive to query", "inflates the result" |

Yang **tidak** diubah (istilah domain tetap): `jenis_pangan`, `kategori_pangan`, `INDUK`/`ANAK`,
`NIE`, `BTP`, `ERBA`/`ERLA`, MR/MT, semua nama tabel & kolom, semua kode, semua SQL, dan seluruh
nilai anchor terverifikasi.

## Peningkatan substansi kecil (2 tempat, dari temuan kegagalan nyata)

Selain rewrite bahasa, dua penyempurnaan substansi ikut dimasukkan — keduanya berasal dari
kegagalan terdokumentasi:

1. **Terjemahan kode tidak lagi berhenti pada "0 baris"** (`SEEKNAL_ASK.md` Gate 2,
   `bpom-analyst` stop rules). Ketika `data_dictionary` mengembalikan 0 baris, itu berarti nama
   `kategori` salah tebak — bukan bukti tidak ada pemetaan. Agen kini diarahkan menampilkan
   daftar kategori (`SELECT DISTINCT kategori, sumber FROM data_dictionary`) sebelum menyimpulkan
   apa pun. Kasus pemicu: jawaban `status_komitmen` yang menyajikan kode mentah 0/1/5/7/8/9
   padahal kategori `STATUS_KOMITMEN` terisi lengkap (termasuk `5` = Komitmen Dibatalkan).
2. **Forecast wajib lewat engine** (`SEEKNAL_ASK.md` Gate 1/3/5, `bpom-forecaster`). Intent
   forecast vs data historis kini menjadi kasus klarifikasi eksplisit (termasuk pertanyaan
   "forecast BTP 2026" pada tahun berjalan); Gate 3 menuntut rencana yang berakhir di
   `run_forecast`; Gate 5 menambah butir #10 — `run_forecast`/`detect_anomaly` harus muncul di
   tool trace, dan jawaban forecast yang dihitung lewat SQL murni tidak lolos gate. Kasus
   pemicu: CB-25 (95 tool calls tanpa memanggil skill forecaster,
   `docs/audit_context/audit_multiturn_18jun2026.md` §6.2) dan FC-RISIKO-4 (estimasi dihitung
   sendiri setelah engine menolak, `docs/planning/2026-07-21-forecast-e2e-round2-deepdive.md` §7).

## Upgrade September — filter katalog, chart/S3 wajib, ground truth baru

Iterasi kedua (4 Sep 2026), setelah verifikasi database via `readonly_user@localhost:5533/rpo_v2`
dan pembelajaran atas audit `docs/audit_context/2026-08-13-v5-route-sql-jalur-mapping/` serta
suite UAT compact I–VIII (angka note-nya valid terhadap kondisi DB kini):

| Perubahan | File | Isi |
|---|---|---|
| **Halaman baru** | `context/05-filter-katalog.md` | Peta panel filter user (12 dimensi + 4 varian tanggal; "Tanggal Terbit SPB" tidak ada di warehouse), katalog label↔kode jenis pangan panel (multi-select `IN`, keluarga tematik via probe nama_kategori, pasangan cair/padat formula, entri BTP yang hanya tampak seperti jenis pangan), label panel tanpa kode di warehouse (Lisensi, Pengemas Kembali, \*Notifikasi), resep kombinasi Mikro×Tinggi, bentuk jawaban daftar-detail, honest gap bahan baku. Prinsip: `05` = peta + resep komposisi; aturan per dimensi tetap di halaman pemiliknya (anti-duplikasi) |
| **Provinsi = derived + divergensi 37/38** | `context/50-pihak-wilayah.md` | Ranking provinsi = `left(daerah_*,2)`; **37 = Jawa Barat, 38 = Jawa Timur** mendominasi data (21.654 / 18.289 NIE terdaftar) tapi tidak ada di dictionary (baris 32/35 kamus kosong) — identifikasi dari seri kabupaten + magnitudo; dilarang melabeli "unmapped" atau membuangnya |
| **Bentuk waktu berulang** | `context/80-waktu-periode.md` | (a) bulan tanpa tahun = semua tahun (fix K1a MEI-MR-1); (b) resep semester S1 vs S2 + label YTD; (c) rolling 12 bulan via `CURRENT_DATE` (bukan hardcode bulan) |
| **Honest gap bahan baku** | `context/95-dimensi-lain.md` | `T_PRODUK_3_BAHAN` hanya di sistem legacy — pertanyaan "mengandung bahan X" NOT COVERED; substitusi jujur = pencarian nama/merk berlabel |
| **Chart + CSV wajib** | `SEEKNAL_ASK.md` Gate 4/5, `skills/bpom-analyst`, `skills/visualize-chart` | Jawaban berdata selalu 3 keluaran (angka + 1 chart + 1 CSV); pengecualian hanya definisional/0 baris; bentuk bawaan: ranking→`horizontal_bar_chart`, kombinasi 2 dimensi→`grouped_bar_chart`/`heatmap`; larangan ekstrapolasi manual dari hasil SQL untuk forecast |
| **Suite UAT baru** | `seeknal/tests/v1/singleturn/UAT-v2-compact-IX/` (7 skenario, ground truth DB 4 Sep 2026) | PROV-TOP-2024 (ranking provinsi; 31=11.188, 37=3.941, 38=3.385…) · MIKRO-TINGGI-2025 (38=709, 31=429…) · SEMESTER-2026 (S1=32.021, S2 YTD=13.717) · ROLLING-12BLN (Sep25–Agu26, total 68.993) · BAHAN-GINSENG (honest gap; nama ILIKE=400) · JP-MULTI-1 (0809+0905=5.365; ERLA 0) · JP-SUSU-FAMILY (probe→1.905 NIE / 27 kode) |

Keputusan arsitektur halaman `05`: katalog panel dan resep komposisi ditempatkan di satu halaman
lintas-dimensi (bukan disebar) karena kelas kegagalan K1b pada audit — resep yang hidup tersebar
di beberapa halaman tidak pernah tersintesis di runtime. `05` sengaja tipis: baris pemetaan
menunjuk halaman pemilik; satu aturan satu file (detail 37/38 tinggal di `50`).

## Hasil run 4 September 2026 (qwen-plus / dens-plus, workers 4)

| Suite | Baseline v5 (13 Agu) | v1-sep-w1 | Catatan |
|---|---|---|---|
| UAT-v2-compact-IX (baru) | — | 1/7 (turn nyata 4/7) | 3 asersi fixture salah format `assert_any_of` (grup=AND) — sudah diperbaiki; 3 kegagalan = turn hantu auto-clarif |
| Compact-I | 16/16 | 9/16 | 3 = note drift komitmen (+6,6–10% vs DB 23 Jul); 2 = harness (AMDK-1 berhenti, AMDK-3 capture); 2 ambiguitas (MT-JP, OFF-4) → sudah ditambal di `30`/`15` |
| Compact-II | 16/17 | 14/17 | 3 = note drift (PIPELINE-EVAL +32,6%, distribusi risiko, kd304 +8%) |
| Compact-III | 16/16 | 15/16 | BAYI-2; **BTP-PEWARNA (34 query di v5) kini PASS bersih** |
| Compact-IV | 14/16 | **14/16** | paritas; **MEI-MR-1 (K1a audit) kini PASS**; MEI26-RISK = turn hantu |
| Compact-V | 13/16 | 12/16 | JP-TREN & LC-DIUBAH gagal juga di v5; MD-1 = prefix `MD` tak dipasang |
| Compact-VI | 13/16 | 11/16 | COM-1/2 + PIPE-VERIF2/DIR = drift; DICABUT-1 = konflik context↔test (agent benar); **CHAR-PANGAN-BAYI-ERLA & DRAFT-1 (audit) kini PASS** |
| Compact-VII | 7/8 | **7/8** | paritas; JP-BARU-VS-REVISI gagal juga di v5 |
| Compact-VIII | 11/11 → 9/11 (v5 dua run); 9/11 (v6) | 6/11 | KOPI-TERBIT = turn degenerat (llm=1, sql=0); PABRIK-TOP & RED-WINE = gagal juga di v5/v6; MI-INSTAN & KOPI-PENDAFTAR = AUTO-turn saja |

**Dekomposisi 34 kegagalan mentah (I–VIII + IX)**: ±13 note drift (note 18–28 Jul vs DB 4 Sep — komitmen/pipeline/kd304 tumbuh 6–33%), ±8 artifact harness auto-clarif/turn hantu, ±6 kegagalan yang sama dengan baseline v5/konflik diketahui, ±7 ambiguitas bacaan (2 sudah ditambal). Perilaku inti tidak menyimpang dari v6; skor mentah turun terutama karena **kesegaran ground truth** dan artifact harness, bukan regresi context.

**Tindak lanjut**: (1) refresh angka note untuk counter cepat atau alihkan ke periode beku; (2) perbaiki harness auto-clarif agar turn hantu tidak menimpa verdict turn asli; (3) re-run komitmen/pipeline setelah note refresh.

**Audit lengkap**: `docs/audit_context/2026-09-04-v1-sep-w1-compact-sweep/` (00-PEMETAAN + 01-rincian-gagal-34) — taksonomi D1–D8, bukti drift terbedakan (status bergerak CDC intraday vs pertumbuhan 2026 vs antrian), validasi SQL note verbatim, dan hipotesis H1–H9 termasuk temuan bahwa `load_skill` gagal di seluruh sweep (symlink skills kini sudah dibuat di varian — re-run diperlukan).

## Struktur

```
v1-sep-w1/
├── README.md                            BARU (dokumen ini)
├── SEEKNAL_ASK.md                       REWRITE gaya + peningkatan (1) & (2)
├── seeknal_agent.yml                    salinan utuh v6
├── context/                             19 halaman — semua REWRITE gaya
│   ├── 00-menghitung.md … 95-dimensi-lain.md
│   ├── data_architecture.md
│   └── forecast_guide.md
└── skills/                              4 skill — REWRITE gaya (versi patch +1)
    ├── bpom-analyst/SKILL.md            6.0.0 → 6.0.1
    ├── bpom-forecaster/SKILL.md         6.2.0 → 6.2.1
    ├── detect-anomaly/SKILL.md          1.1.0 → 1.1.1
    └── visualize-chart/SKILL.md         2.0.0 → 2.0.1
```

## Yang belum dilakukan

- **Belum diuji terhadap suite UAT.** Sebelum dipakai sebagai varian aktif, jalankan sapuan
  penuh (setidaknya `UAT-v2-compact` + batch forecast) untuk memastikan rewrite bahasa tidak
  menggeser perilaku agen.
- Symlink `seeknal/skills → ../skills` dari v6 belum direplikasi di varian ini — prasyarat
  `load_skill`. Buat sebelum run (lihat README v6 §Prasyarat menjalankan).
- `.seeknal/checkpoints` adalah artefak runtime bawaan salinan v6, bukan bagian dari rewrite.
