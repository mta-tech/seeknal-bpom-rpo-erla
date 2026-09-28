# seeknal-bpom-neo: Kontrak Konsistensi Jawaban ↔ Ekspor ↔ Chart

**Document type:** Audit Findings + Context Change Plan
**Project:** seeknal-bpom-neo (BPOM RPO Analytics Agent)
**Status:** draft — belum diterapkan ke varian mana pun
**Date:** 2026-08-13
**Scope berkas (usulan):** `SEEKNAL_ASK.md` · `skills/bpom-analyst/SKILL.md` · `skills/visualize-chart/SKILL.md` · `context/00-menghitung.md` · `context/80-waktu-periode.md` · `context/90-kualitas-data.md` · `context/12-nama-kategori.md` · `context/35-klasifikasi-sifat.md`
**Varian sasaran:** `docs/context_recap/after-chart-route/route-context-070826-v4` (varian lain sengaja tidak disentuh sebagai pembanding)
**Spec engine pendamping:** `iba-deploy-runbook/specs/2026-08-13-spec-oi1-output-integrity-export-chart-fallback.md`
**Mengamandemen:** `2026-07-20-csv-store-contract.md` (memperluas kontrak dari "berapa kali mengekspor" ke "apa isi yang diekspor")
**Melanjutkan:** `2026-08-07-routed-context-pages-architecture.md` (arsitektur aktif) · `2026-08-05-sql-execution-path-and-column-type-context.md` (pola pembagian peran context vs engine)
**Bukti:** 4 CSV unduhan produksi 12 Agu 2026 · ±35 query verifikasi langsung ke `rpo_v2` · pembacaan kode engine & frontend

---

## 1. Ringkasan Eksekutif

Audit atas empat berkas CSV produksi dan dua tangkapan layar jawaban menemukan enam belas cacat pada jalur jawaban → ekspor → chart. Setelah dipisahkan menurut lapisannya:

| Lapis | Jumlah | Ke mana |
|---|--:|---|
| **Context / skill** | 12 | dokumen ini, R1–R8 |
| **Kode** | 3 | spec OI1 §3.2–§3.4 |
| **Deployment** | 1 | spec OI1 §3.1 — **di luar cakupan dokumen ini** |

Dua belas cacat context/skill lahir dari satu celah yang sama. Satu turn menghasilkan tiga keluaran — kalimat jawaban, berkas CSV, dan gambar chart — lewat tiga jalur berbeda, dan **tidak ada satu aturan pun yang mengharuskan ketiganya berbicara tentang populasi yang sama**.

### 1.1 Pemotongan 5.000 baris: bukan urusan context/skill

Penelusuran git menuntaskan pertanyaannya. Konstanta `_CSV_EXPORT_ROW_LIMIT = 5000` hidup di `upload_to_s3.py` antara `0b86873a` (2026-07-23) dan `707feb96` (2026-08-03). **Perbaikannya sudah ada di repo; image worker di VM yang belum diperbarui** — CSV bermasalah diunduh 2026-08-12, sembilan hari setelahnya.

Karena itu:

- Rancangan aturan `no LIMIT` untuk `bpom-analyst/SKILL.md` yang sempat saya usulkan **ditarik dari dokumen ini.** Ia tidak akan memperbaiki apa pun terhadap build lama, dan menambah satu aturan yang tidak menjawab masalahnya justru mengencerkan aturan lain (lihat §5.1).
- **Versi `seeknal-worker` tidak dibahas lagi di sini**, termasuk sebagai catatan. Seluruhnya milik spec OI1 §3.1.
- Pertahanan sisi-agent terhadap `LIMIT` tetap dibangun, tetapi **di kode** — deteksi struktural pada SQL ekspor (spec OI1 §3.1c), bukan sebagai kalimat prosa yang bergantung kepatuhan.

**Koreksi terhadap laporan lisan saya sebelumnya:** saya sempat menyatakan `LIMIT 5000` adalah karangan agent karena angka itu tidak ada di context/skill/planning mana pun. Itu keliru — angkanya ada di kode, dan bukti dari CSV tidak bisa memisahkan cap sisi-tool dari `LIMIT` sisi-agent (pembungkus `SELECT * FROM (<sql>) AS _q LIMIT 5000` di `repl.py:618` mempertahankan `ORDER BY` bagian dalam, jadi hasilnya identik).

### 1.2 Koreksi kedua: `klasifikasi_id='3'`

Temuan awal saya menyebut `3` sebagai **induk hierarki** dari `301`/`302` sehingga barisnya **saling menghitung ulang**. Diuji ulang ke database, **klaim double-count itu salah dan saya cabut**:

- Hanya **125 dari 207.625 NIE** (0,06 %) yang pernah membawa `3` *dan* `301`/`302`. Pada tingkat entity, bucket-nya praktis saling lepas.
- `data_dictionary` hanya punya kolom `id, sumber, kategori, kode, deskripsi` — **tidak ada kolom parent**. Hierarki itu inferensi dari penomoran kode, bukan fakta tercatat.

Yang **terbukti** dan menjadi dasar R5 justru berbeda, dan lebih umum — lihat §2.4.

---

## 2. Bukti

Seluruh angka berasal dari query langsung ke `rpo_v2` atau dari CSV yang diunduh pengguna. Sesuai disiplin isi varian route, **angka-angka ini tinggal di dokumen perencanaan; nol di antaranya boleh masuk ke `context/` atau `skills/`.**

### 2.1 Mi instan — populasi sebenarnya

Filter: `nama_kategori IN ('Mi Instan','Mi Instan Lainnya')`, `TRIM(status)='0999'`, `nomor<>''`, eksklusi akun uji.

| Sistem | Baris mentah | NIE aktif | Masih berlaku | Sudah lewat |
|---|--:|--:|--:|--:|
| ERBA | 1.932 | 479 | 479 | 0 |
| ERLA | 5.699 | 2.105 | 575 | 1.530 |
| **Total** | **7.631** | **2.584** | **1.054** | **1.530** |

Irisan `nomor` antar sistem = **0**, jadi penjumlahan lintas sistem sah.

Tiga fakta yang menjelaskan cacat jawaban:

- **15 NIE punya beberapa baris aktif dengan `tanggal_exp` berbeda.** Menghitung dengan dua `FILTER` terpisah menghasilkan 575 + 1.545 = 2.120, melebihi populasi 2.105. Inilah sebab jawaban turn-1 menyebut 1.545 dan turn-2 menyebut 1.530 untuk pertanyaan yang sama.
- **Semua 7.631 baris berstatus `0999`.** Duplikasinya terjadi *di dalam* tier aktif, bukan antara versi lama (`9999`) dan baru. Contoh `MD 331528340001`: delapan baris berurutan dengan `kemasan_id` 33, "Cup Kertas (75 g)", tanggal terbit dan `tanggal_exp` identik — hanya `produk_id` yang berbeda.
- **2.197 dari 2.584 NIE (85 %) punya tepat satu baris.** Pembengkakan datang dari 301 NIE (11,6 %) yang menyumbang 5.051 dari 7.631 baris.

Ekspor memilih tujuh kolom dan membuang `produk_id` serta `tanggal` — satu-satunya kolom yang membedakan baris. Hasilnya **4.768 dari 7.631 baris identik byte-per-byte**.

### 2.2 Permohonan 2026 — mayoritas populasi tidak terkategori

| | Jumlah |
|---|--:|
| Permohonan 2026 gabungan (`COUNT(DISTINCT produk_id)`) | **42.040** |
| `nama_kategori` kosong | **25.595 (60,9 %)** |
| — ERBA | 25.595 dari 41.609 |
| — ERLA | 1 dari 433 |

CSV menampilkan bucket `Lainnya` = 25.408 sebagai baris terbesar; jawaban di layar menyatakan *"Kategori Tertinggi: Roti Isi (643)"* — deskripsi atas kira-kira **9 %** populasi, tanpa satu kalimat tentang 61 % sisanya.

Dua cacat menyertainya: chart memuat batang **"Air Mineral" 318** padahal lingkup yang diumumkan *"Selain Kopi Instan & AMDK"*; dan headline **33.391** tidak cocok dengan sumber mana pun yang bisa dibentuk (jumlah CSV 42.251 · DB semua 42.040 · baru `301` 26.967 · berkategori 16.445 · distinct `nomor` 37.402 · ERBA saja 41.607).

### 2.3 Chart tren — 22 % sumbu tidak ada datanya

Seri bulanan yang diperiksa: **115 baris untuk 148 bulan kalender** → 33 bulan hilang (22,3 %), 23 lubang. Karena `mark.interpolate: "monotone"`, garis ditarik lurus melintasi lubang: 2018-04 yang sebenarnya **0** digambar ≈ 2,5; 2016-04 yang sebenarnya **0** digambar ≈ 1,7. Rata-rata dari baris hasil (3,99/bln) melebihi rata-rata terhadap kalender sebenarnya (3,10/bln) sebesar **+28,7 %**.

Sumbunya sendiri benar — encoding `temporal`, posisi tiap titik akurat. Yang berlubang datanya.

### 2.4 Prioritas pengawasan — bucket kasar dan komposisi sistem yang timpang

Kolom berjudul `kategori_pangan` berisi "Makanan", "Minuman", "Deputi 3 (Pangan)". Nilai-nilai itu **bukan** `kategori_pangan` — mereka deskripsi `klasifikasi_id`. `kategori_pangan` yang asli berisi kode 8–12 digit (`140104030002` di ERBA, `06040309` di ERLA).

**Fakta 1 — `klasifikasi_id='3'` bukan kategori sejajar, melainkan bucket kasar.** Isinya membentang ke seluruh ruang produk yang juga dicakup `301`/`302`:

| `nama_kategori` teratas pada baris `klasifikasi_id='3'` (ERBA) | Baris |
|---|--:|
| *(kosong)* | 41.158 |
| Kopi Bubuk | 2.073 |
| Kukis | 1.602 |
| Konsentrat Minuman Rasa/Berperisa … | 1.557 |
| Air Mineral | 1.042 |
| Roti Isi | 610 |

Makanan **dan** minuman duduk di bawah label yang sama. `3` = tingkat kedeputian, bukan kelas produk.

**Fakta 2 — komposisi sistem per label sangat timpang.** Ini yang membuat tabelnya tidak bisa dibaca sebagai perbandingan:

| `klasifikasi_id` | ERBA | ERLA | % ERBA |
|---|--:|--:|--:|
| `3` Deputi 3 (Pangan) | 93.737 | 11 | **100,0 %** |
| `302` Minuman | 59.580 | 93.062 | 39,0 % |
| `301` Makanan | 106.048 | 269.509 | 28,2 % |
| `304` Minuman Beralkohol | 0 | 17.009 | **0,0 %** |

**Fakta 3 — akibatnya, kolom milik satu sistem terbaca sebagai sifat kategori.** Di CSV, `batal_komitmen > 0` hanya muncul pada baris "Deputi 3 (Pangan)" (113 dari 249). Penjelasan yang benar bukan "komitmen ERBA-only" secara umum — di dalam ERBA, `status_komitmen` terisi hampir 100 % pada **semua** klasifikasi (301: 106.047/106.048 · 3: 93.736/93.737 · 302: 59.580/59.580). Penjelasannya adalah Fakta 2: baris "Deputi 3" **100 % ERBA**, sedangkan "Makanan"/"Minuman" didominasi **ERLA** yang secara struktural tidak punya kolom itu. Pembaca menyimpulkan "pembatalan komitmen terjadi di Deputi 3, tidak di Makanan/Minuman" — kesimpulan tentang sistem yang menyamar jadi kesimpulan tentang pangan.

**Fakta 4 — kolom berbeda lingkup waktu dalam satu baris.** **179 dari 729 baris** punya `nie_risiko_tinggi` lebih besar daripada `nie_2024 + nie_2025`: kolom sepanjang-masa disandingkan dengan kolom dua-tahun, tanpa label.

---

## 3. Akar: satu kontrak yang tidak pernah ditulis

### 3.1 Kontrak ekspor berhenti di "berapa kali", tak pernah sampai "apa isinya"

`2026-07-20-csv-store-contract.md` menempuh enam ronde amandemen (§5b–§5f) — posisi ekspor, jumlah ekspor, self-check anti-duplikat, larangan Mode 2. Semuanya **aturan proses**. Tidak satu pun menyentuh **isi**: entity mana yang diekspor, urutannya apa, kolomnya harus memuat apa.

Pada kasus mi instan, agent memanggil `upload_to_s3` tepat sekali, sebagai tool call terakhir, tanpa `data=`, tanpa URL mentah — **patuh 100 % pada setiap kalimat yang ada** — dan berkasnya tetap tidak mewakili jawabannya. Menambah tekanan kepatuhan tidak akan menolong.

### 3.2 Kelengkapan periode tidak pernah ditulis, bukan hilang saat diringkas

Delapan generasi `visualize-chart/SKILL.md` diperiksa dengan pencarian `missing|gap|kosong|zero|continuous|complete|calendar|dense`. Seluruh kecocokan tidak relevan ("zero rows", "missing chart", "missing number"). Klaim README v2 bahwa peringkasan 181 → 122 baris "mempertahankan semua aturan" **benar** — tidak ada yang hilang, karena aturan ini memang tidak pernah ada.

Ironisnya sistem ini **sudah punya konsepnya**. `run_forecast` menyimpulkan grain dari modus selisih bulan, menoleransi lubang sebagai kelipatan, menghitung `gap_months`, dan melaporkannya sebagai baris **"✓ Tidak ada bulan kosong dalam riwayat data"** — ada sejak `2026-06-18-llm-forecaster-skill.md`. Kemampuan itu tidak pernah menyeberang ke jalur jawaban biasa.

### 3.3 Aturan cakupan sudah ada di planning, tidak pernah menyeberang ke halaman route

`2026-06-12-dimension-reasoning-and-data-coverage.md` §3 sudah merumuskannya:

> **Coverage-aware column choice:** … If a result is dominated by NULL / "Tanpa Kategori" / "unidentified", stop and switch to a more complete column for the same concept. If coverage is still low, **state the limitation honestly** — never present "Tanpa Kategori" as the answer.

Aturan itu tidak ada di `90-kualitas-data.md` maupun `12-nama-kategori.md` pada varian route mana pun. §2.2 adalah aturan itu dilanggar persis seperti yang diramalkan empat bulan sebelumnya.

---

## 4. Perubahan yang diusulkan

Delapan perubahan, semuanya **general topic-level**: tidak boleh terikat pada mi instan, kopi instan, atau pertanyaan mana pun yang memicu audit ini. Analisis risiko regresi per butir ada di §5.

### R1 · Granularitas ekspor = granularitas jawaban — `skills/bpom-analyst/SKILL.md`

> Berkas ekspor memakai **entity yang sama dengan jawaban**. Jawaban yang dihitung dengan `COUNT(DISTINCT <entity>)` diekspor **satu baris per entity**, bukan satu baris per baris tabel. Bila beberapa baris mentah runtuh menjadi satu, bawa kolom penghitungnya (mis. `jml_versi`) supaya peringkasannya terlihat, bukan disembunyikan.
>
> **Kecuali pertanyaannya memang meminta tingkat baris** — "daftar pengajuan", "riwayat perubahan", "per produk_id". Maka ekspornya di tingkat itu, dan SELECT-nya **wajib** memuat kolom yang membedakan baris (`produk_id`, tanggal). Membuang kolom pembeda lalu mengekspor baris mentah menghasilkan duplikat yang tidak bisa ditafsirkan siapa pun.

Menutup: F3, separuh F11.

### R2 · Kontrak pengurutan ekspor — `skills/bpom-analyst/SKILL.md`

> Setiap ekspor punya `ORDER BY` yang **mengikuti maksud pertanyaan**, bukan urutan alami tabel:
>
> | Bentuk pertanyaan | Urutan |
> |---|---|
> | peringkat / "terbanyak" / "tertinggi" | metrik **menurun**, lalu label |
> | tren / deret waktu | periode **menaik** |
> | lintas dimensi (wilayah × kategori) | dimensi utama, lalu metrik menurun di dalamnya |
> | daftar / lookup | pengenal entity |
>
> Tambahkan satu kolom tie-break deterministik di akhir, supaya dua kali menjalankan pertanyaan yang sama menghasilkan berkas identik.

Menutup: F5.

### R3 · Resolusi verdikt lintas-versi — `context/00-menghitung.md`

> Tabel produk berversi: satu entity bisa menempati banyak baris **yang sama-sama aktif**. Aturan ini berlaku **hanya bila** jawaban membaca kolom turunan per entity (tanggal, status, kelas) **dan** baris-baris itu tidak sepakat. Cacah entity biasa (`COUNT(DISTINCT nomor)`) tidak terpengaruh sama sekali.
>
> Bila terjadi: tetapkan **versi penentu sekali** — versi aktif dengan tanggal paling akhir — lalu pakai versi itu untuk seluruh kolom turunan.
>
> **Jangan memecah populasi dengan dua `FILTER` terpisah pada baris mentah.** Entity yang punya baris di kedua sisi tercacah dua kali, dan pecahannya melebihi populasinya sendiri. Uji satu baris: apakah jumlah pecahan sama dengan total? Lebih besar → ada entity di dua kelompok; resolusikan, jangan sajikan.

Menutup: F10.

### R4 · Ambang cakupan sebelum menyajikan peringkat — `context/90-kualitas-data.md`, rujukan silang dari `context/12-nama-kategori.md`

> Sebelum menyajikan peringkat atau "yang tertinggi" atas sebuah dimensi, periksa **keterisian kolomnya pada populasi yang ditanya** — bukan keterisian umumnya.
>
> - Kosong / tak terklasifikasi **menguasai hasil** → jangan sajikan Top-N seolah menggambarkan populasi. **Sebutkan porsinya lebih dulu sebagai bagian dari jawaban.** Berpindah ke kolom lain untuk konsep yang sama hanya bila kolom itu memang menjawab pertanyaan yang sama — dan perpindahannya dinyatakan, tidak pernah diam-diam.
> - Bucket kosong **tidak pernah** menjadi "kategori tertinggi", dan tidak pernah diam-diam dibuang dari chart sementara ia tetap ada di ekspor.
> - Keterisian bisa **sangat berbeda antar sistem** untuk kolom yang sama. Periksa per sistem; angka gabungan menyembunyikan bahwa persoalannya khas satu sisi.

Menutup: F12.

### R5 · Bucket kasar & komposisi sistem per kelompok — `context/35-klasifikasi-sifat.md`

Dirumuskan ulang dari temuan yang **terbukti** (§2.4), bukan dari hipotesa hierarki yang dicabut:

> Beberapa keluarga kode memuat **nilai berbutir kasar** berdampingan dengan nilai spesifik — satu kode yang menandai tingkat organisasi/payung, dan kode-kode lain yang menandai kelas sebenarnya. Kenali dari isinya: bila satu nilai memuat produk yang **juga** dicakup nilai lain di kolom yang sama, ia bukan kategori sejajar melainkan **penanda "belum dispesifikkan"**. Perlakukan seperti kekosongan (R4): boleh disebut, tidak boleh diperingkat bersama kelas spesifik seolah setara.
>
> **Periksa komposisi sistem tiap kelompok sebelum membandingkan antar-baris.** Nilai yang sama bisa berasal hampir seluruhnya dari satu sistem pada satu baris dan dari sistem lain pada baris berikutnya. Bila kelompok-kelompoknya berbeda komposisi sistem, **kolom yang hanya ada di salah satu sistem akan terbaca sebagai sifat kelompoknya** — padahal ia sifat sistemnya. Dua jalan keluar, pilih dan nyatakan: pecah jawaban per sistem, atau batasi kolom yang ditampilkan pada kolom yang ada di kedua sistem.

Tambahan di `skills/bpom-analyst/SKILL.md` (Presentasi):

> **Nama kolom ekspor menyebut kolom sumbernya.** Memberi nama satu dimensi dengan nama dimensi lain membuat analis hilir menggabungkannya ke kamus yang salah dan mendapat nol baris tanpa error.

Menutup: F6, F14.

### R6 · Kelengkapan periode pada deret waktu — `skills/visualize-chart/SKILL.md` + `context/80-waktu-periode.md`

> Deret waktu yang dijawab atau digambar harus **rapat di kalendernya**. `GROUP BY date_trunc(...)` hanya mengembalikan periode yang punya baris; periode tanpa baris hilang dari sumbu, dan garis chart ditarik melintasinya seolah ada nilai di sana.
>
> **Tentukan grain lebih dulu — jangan mengasumsikan bulanan.** Agregasi tahunan selalu jatuh di 1 Januari tiap tahun; membacanya sebagai deret bulanan yang bolong 11 bulan akan menyisipkan nol palsu dan merusak trennya. Simpulkan grain dari selisih yang paling sering muncul antar periode berurutan, dan perlakukan selisih yang merupakan kelipatannya sebagai **lubang**, bukan grain yang berbeda.
>
> **Isi nol hanya untuk metrik arus** — cacah peristiwa per periode. Periode tanpa peristiwa memang nol. **Untuk metrik stok** (mis. "yang aktif per bulan") periode tanpa baris bukan nol; jangan diisi.
>
> **Batasi rentang isian** pada rentang yang diminta, atau pada periode paling awal dan paling akhir yang benar-benar ada. Jangan melar ke masa depan: periode berjalan yang belum lengkap ditandai sebagai berjalan, bukan digambar sebagai keruntuhan.
>
> **Pastikan lubangnya benar-benar nol, bukan tanggal kosong.** Baris yang kolom tanggalnya kosong hilang dari semua periode; menyulapnya menjadi "0" mengubah *tidak diketahui* menjadi klaim tentang bisnis. Periksa keterisian sebelum merapatkan.

**Prasyarat teknis:** butir D3 pada spec OI1 harus mendarat lebih dulu — lihat §5.2.
Menutup: F7.

### R7 · Sisakan jatah untuk menulis jawaban — `SEEKNAL_ASK.md` Gate 4

Ketika jatah tool habis sebelum model sempat menulis, ada **dua jalur**, dan hanya satu yang bisa disentuh context:

- **Jalur A** (`gateway/server.py:815-837`) — pass sintesis memanggil model lagi dengan `message_history` utuh. Context berlaku di sini, dan aturannya sudah ada di Gate 4.
- **Jalur B** (`server.py:838-839`, atau `:837` bila output kosong) — teks mentah `synthesize_evidence_fallback()`, **tanpa pemanggilan model sama sekali**. Context tidak bisa menyentuhnya; tampilannya milik spec OI1 §3.4.

Maka tugas context bukan memperbaiki tampilan kegagalan, melainkan **mencegah Jalur B tercapai**. Ditulis sebagai perluasan kalimat yang sudah ada di Gate 4 (*"If the headline number is out of reach, answer with what resolved and name what did not"*), bukan sebagai blok baru:

> **Sisakan jatah untuk menulis jawaban.** Menggali sampai langkah terakhir habis berarti turn berakhir tanpa jawaban sama sekali — hasil terburuk dari semua kemungkinan. Selagi jatah masih tersisa, berhenti menggali dan tulis jawaban dari bukti yang sudah terkumpul.
>
> Jawaban itu **bukan kegagalan**: sebut angka yang sudah didapat, lalu sebut bagian yang belum sempat dihitung sebagai catatan — dimensi mana, metrik mana, dan apa yang dibutuhkan untuk menyelesaikannya. Pertanyaan yang menuntut banyak metrik sekaligus wajar tidak selesai dalam satu turn; yang tidak wajar adalah berakhir tanpa apa pun.

Ada preseden bahwa pencegahan lewat prosa bekerja: `2026-07-20` §5e mencatat varian dengan frame *"Budget ledger — count everything, hard-STOP"* punya rate double-call **9,1 %** vs **37,5 %** pada varian tanpa frame itu.

**Menggantikan rancangan sebelumnya** yang mewajibkan pertanyaan ≥3 metrik dijawab bertahap. Rancangan itu ditarik: §5.2 menandainya sebagai regresi paling halus di seluruh paket — ia bisa membuat pertanyaan dua-metrik yang hari ini dijawab lengkap menjadi dijawab sebagian. Aturan pengganti ini tidak menyentuh pertanyaan yang hari ini sudah selesai.

Menutup: pemicu F15.

### R8 · Gerbang rekonsiliasi — `SEEKNAL_ASK.md` Gate 5

> **Tiga keluaran satu turn harus bisa saling diterangkan.** Ini pemeriksaan atas angka yang **sudah ada di tangan** — tidak menambah query:
> - Jumlah baris ekspor **dapat diterangkan** terhadap angka headline — sama dengannya, atau selisihnya disebut sebabnya. Selisih yang tidak bisa diterangkan berarti salah satu dari keduanya salah.
> - **Lingkup yang diumumkan terlihat di dalam SQL ekspor dan SQL chart**, bukan hanya di kalimat jawaban. Menyatakan "selain X" lalu menampilkan X di chart adalah kontradiksi yang terlihat pengguna.
> - Chart dan angka berasal dari **rows yang sama**; jumlah titiknya dapat diterangkan terhadap rentang periode yang dijawab.
> - **Tabel markdown tidak pernah mengulang isi chart.** Bila tabel ringkas membantu, tulis dengan setiap baris pada barisnya sendiri dan baris kosong sebelum tabel — tabel yang ditulis dalam satu baris di dalam butir daftar tidak ter-render dan tampil sebagai teks mentah.

Menutup: F13, F16, dan jaring pengaman untuk sisanya.

---

## 5. Analisis risiko regresi

Ini bagian yang paling menentukan apakah perubahan ini boleh diterapkan. Preseden di proyek ini jelas: **v3 mengaktifkan skill dan pass-rate justru turun 95 → 92/116** karena satu aturan terlalu agresif membuang filter segmen yang diminta (BAYI 110 → 395). Delapan aturan sekaligus punya delapan peluang mengulanginya.

### 5.1 Risiko sistemik — berlaku untuk seluruh paket

| Risiko | Mekanisme | Mitigasi |
|---|---|---|
| **Pengenceran instruksi** | Makin banyak aturan, makin rendah kepatuhan pada aturan yang sudah ada. Ini bukan hipotesa: v3 membuktikannya | Terapkan **bertahap per gelombang** (§5.3), ukur tiap gelombang sebelum menambah |
| **Duplikasi aturan → tool call ganda** | Terbukti di `2026-07-20` §5c: kontrak yang sama ditulis di `SEEKNAL_ASK.md` **dan** skill membuat agent memanggil `upload_to_s3` dua kali | Tiap aturan hidup di **tepat satu berkas**. R1/R2/R5-presentasi → skill analis. R3/R4/R5-data/R6-data → halaman context. R7/R8 → `SEEKNAL_ASK.md`. **Nol pengulangan lintas berkas** |
| **Jalur panas membengkak** | Arsitektur route dibangun untuk memangkas jalur panas 62 %. R1/R2/R5-presentasi/R6-chart dimuat pada setiap pertanyaan data | Anggaran: pertumbuhan jalur panas **≤ 10 %**. Bila terlampaui, ringkas aturan lama yang sudah terbukti, bukan tambah halaman |
| **Aturan menyala di kasus yang salah** | Aturan bersyarat yang syaratnya longgar akan menyala di pertanyaan yang hari ini sudah benar | Tiap aturan wajib punya **kalimat kapan ia TIDAK berlaku**, ditulis eksplisit (sudah dilakukan di R1, R3, R7) |

### 5.2 Risiko per aturan

| # | Yang bisa jadi lebih buruk | Kenapa | Pagar yang sudah dipasang | Cara mendeteksi |
|---|---|---|---|---|
| **R1** | Pertanyaan yang memang minta detail per pengajuan dijawab dengan berkas yang sudah diringkas | Aturan "satu baris per entity" dibaca sebagai universal | Klausa "kecuali pertanyaannya meminta tingkat baris", dengan tiga contoh pemicu | Skenario "daftar/riwayat" — ekspor **wajib** masih memuat `produk_id` |
| **R1** | `DISTINCT ON` / window menambah kompleksitas SQL → error baru | Bentuk SQL baru yang belum pernah dipakai agent | Aturan menyebut bentuk, tidak mewajibkan satu sintaks | Hitung SQL gagal per turn; harus **tidak naik** |
| **R2** | `ORDER BY` pada UNION besar memaksa sort → lebih lambat, berisiko timeout | Batch 2026-08-05 menghasilkan **12 timeout dari 48 run** karena jalur SQL | Urutan hanya pada **SQL ekspor**, tidak pada query pencacahan | Durasi turn median; timeout **tidak naik** |
| **R3** | **Risiko tertinggi.** Diterapkan terlalu luas, ia mengubah cacah yang hari ini sudah benar | "Versi penentu" bisa dibaca sebagai wajib untuk semua query | Kalimat pembatas eksplisit: berlaku **hanya** bila kolom turunan dibaca per entity **dan** versinya tidak sepakat; `COUNT(DISTINCT …)` dinyatakan tidak terpengaruh | Bandingkan headline seluruh skenario yang hari ini PASS — **tidak boleh ada yang bergerak** |
| **R4** | Agent jadi ragu-ragu dan membubuhi peringatan cakupan di jawaban yang cakupannya baik | Ambang tidak ternyatakan → dipakai di mana-mana | Aturan berbunyi "menguasai hasil", bukan "ada kekosongan" | Skenario yang kolomnya penuh — **tidak boleh** muncul kalimat cakupan |
| **R4** | Perpindahan kolom merusak jalur kanonik yang sudah divalidasi | `2026-08-04` F-11 menetapkan `nama_kategori` sisi **ERLA** (96,2 % terisi) sebagai jalur sah untuk enam berkas uji | Perpindahan kolom **bukan langkah pertama** — menyebut porsi yang dulu; pindah hanya bila kolom lain menjawab pertanyaan yang sama, dan dinyatakan | Enam berkas uji `GARAM/BAYI` — **tetap PASS** |
| **R5** | Agent menghindari `klasifikasi_id` sama sekali, atau memecah tiap jawaban per sistem sehingga bertele-tele | "Periksa komposisi sistem" dibaca sebagai "selalu pecah" | Dua jalan keluar ditawarkan setara; pemecahan hanya bila komposisinya memang berbeda | Panjang jawaban dan jumlah SQL — **tidak naik** |
| **R5** | Bertabrakan dengan aturan yang sudah ada | `35-klasifikasi-sifat.md` sudah punya "kategori makanan/minuman punya dua arti" | R5 ditulis sebagai **butir baru**, tidak menyentuh butir dua-arti | Baca ulang halaman utuh setelah edit |
| **R6** | Nol palsu pada metrik stok | "Isi nol" dibaca universal | Klausa arus-vs-stok, memakai konsep yang **sudah ada** sejak R4/2026-07-13 | Skenario "masih berlaku per periode" — **tidak boleh** ada nol sisipan |
| **R6** | Bentuk tabel jawaban berubah → assertion UAT yang menghitung baris ikut berubah | Baris nol menambah baris pada tabel jawaban | Nyatakan bahwa perapatan untuk **chart**; tabel teks tetap menyajikan periode bermakna | Diff `note` GT, bukan kolom PASS |
| **R6** | Date-spine menambah `JOIN`/`generate_series` → biaya, dan pg-routing menolak projection tanpa alias | `sql_routing._has_unnamed_projection()` | Aturan menyebut kewajiban alias | Rasio SQL yang jatuh ke fallback DuckDB — **tidak naik** |
| **R6** | **Membangunkan cacat laten.** 5 seri × 148 bulan = 740 baris > `CHART_MAX_ROWS` 500 → pemangkas global tidak sadar-seri memotong periode berbeda per seri | Perapatan menaikkan jumlah baris payload | **R6 tidak boleh mendarat sebelum spec OI1 §3.3.** Urutan ini wajib | Chart multi-seri bulanan rentang panjang — tiap garis utuh |
| **R6** | Melanggar kontrak `run_forecast` | `forecast.py:47-64` menolak `JOIN`, `GENERATE_SERIES`, `WITH RECURSIVE` termasuk *"no date-spine joins"* | Aturan ditulis di `visualize-chart` + `80-waktu-periode`, **tidak** di `forecast_guide.md`; disebut eksplisit bahwa forecast mengurus lubangnya sendiri | Skenario forecast — **nol** penolakan pola SQL |
| **R7** | Agent berhenti terlalu dini pada pertanyaan yang sebenarnya masih muat dalam satu turn | "Sisakan jatah" dibaca sebagai ambang angka | Aturan berbunyi "selagi jatah masih tersisa", bukan angka; ledger yang ada tetap mengatur kapan menggali | SQL per turn **tidak turun** pada skenario yang hari ini sudah lengkap |
| **R7** | Catatan "belum selesai" muncul pada jawaban yang sebenarnya sudah lengkap | Kewajiban mencatat dibaca sebagai selalu berlaku | Aturan hanya menyala bila memang ada yang belum dihitung | Skenario satu-metrik — **nol** catatan tambahan |
| **R8** | Gerbang memicu query verifikasi tambahan → anggaran habis, ironisnya memicu F15 | "Rekonsiliasi" dibaca sebagai "hitung ulang" | Ditulis sebagai pemeriksaan atas angka **yang sudah di tangan**; "tidak menambah query" ada di kalimat aturannya | Jumlah SQL per turn — **tidak naik** |
| **R8** | Agent menolak menjawab ketika selisih tak bisa diterangkan | "Berarti salah satu salah" dibaca sebagai perintah berhenti | Aturan menyuruh **memperbaiki**, dan jalur "jawab dengan yang ada + sebutkan batasnya" tetap terbuka di Gate 4 | Turn tanpa jawaban — **0** |

### 5.3 Penerapan bertahap

Jangan menerapkan delapan sekaligus. Tiga gelombang, tiap gelombang diukur sebelum lanjut:

| Gelombang | Aturan | Alasan urutan | Gerbang lanjut |
|---|---|---|---|
| **1** | R1, R2, R7, R8 | Risiko terendah; tidak menyentuh cara menghitung sama sekali. R7 versi baru hanya memperluas kalimat berhenti yang sudah ada di Gate 4 | PASS tidak turun · nol ekspor yang tak bisa direkonsiliasi · nol turn kehabisan jatah |
| **2** | R3, R4, R5 | Menyentuh cara menghitung dan menyajikan → butuh pembandingan headline yang ketat | PASS tidak turun · **nol headline yang bergerak** pada skenario yang sudah benar |
| **3** | R6 | Menunggu spec OI1 §3.3 mendarat lebih dulu — perapatan kalender sebelum itu justru membangunkan pemangkas yang tidak sadar-seri | PASS tidak turun · chart multi-seri bulanan rentang panjang utuh per garis |

### 5.4 Cara mengukur, dan jebakannya

Bandingkan varian bersanding dengan v4 apa adanya sebagai kontrol.

**Kolom PASS/FAIL harness tidak cukup.** `_token_in_answer` memindai seluruh teks jawaban; fragmen kode status dan tanggal bisa meluluskan jawaban yang salah — README v4 sendiri sudah memperingatkan ini, dan audit compact-VIII menemukan 71 % PASS palsu. **Bandingkan headline jawaban dengan `note` GT**, bukan kolom PASS.

Tiga prasyarat harness yang harus dicek dulu, kalau tidak angkanya tidak bermakna: symlink `seeknal/skills → ../skills` ada · `read_max_lines` tidak memotong halaman · `origin` ≠ `status` di trace.

Regresi yang **tidak** akan tertangkap suite dan harus diperiksa manual: isi berkas CSV, dan bentuk chart. Keduanya tidak masuk assertion mana pun — itulah sebabnya enam belas cacat ini baru ketahuan lewat audit manual.

---

## 6. Sengaja tidak dikerjakan di sini

| Item | Ke mana |
|---|---|
| Pemotongan ekspor 5.000 baris & versi image worker | spec OI1 §3.1 — **tidak dibahas di dokumen ini** |
| Deteksi `LIMIT` pada SQL ekspor | spec OI1 §3.1c — di kode, bukan prosa |
| Pergeseran satu hari pada `toIsoDate` | spec OI1 §3.2 |
| `_downsample` tidak sadar-seri | spec OI1 §3.3 — **prasyarat R6** |
| Tampilan jalur kegagalan budget (Jalur B, tanpa pemanggilan model) | spec OI1 §3.4 — R7 hanya mencegah jalur itu tercapai, tidak bisa mengubah teksnya |
| Menampilkan cacah baris ekspor & `notices` chart ke pengguna | spec OI1 §3.5 — seeknal memancarkan, iba-web menampilkan; R8 memakai angka yang sama untuk gerbang rekonsiliasi |
| Hook PRE_TOOL_USE anti-ekspor-ganda (H-C1) | tetap di luar cakupan, `2026-07-20` §4 |
| Perbaikan harness `test_variant_compare.py` | di luar cakupan varian |
| `bpom-forecaster` · `detect-anomaly` | nol perubahan |

---

## 7. Rambu saat menerapkan

1. **R6 sesudah spec OI1 §3.3, bukan sebelum.** Urutan terbalik menciptakan cacat baru pada chart multi-seri.
2. **Jangan sentuh `forecast_guide.md`.** `run_forecast` menolak date-spine dan mengisi lubangnya sendiri.
3. **Batas hari ini adalah pilihan, dan harus dinyatakan.** Dengan `exp > CURRENT_DATE`, entity yang kedaluwarsa tepat hari ini masuk "sudah lewat" — sah, tetapi tidak boleh diam-diam.
4. **Setiap projection wajib punya alias**, kalau tidak passthrough PostgreSQL ditolak dan query jatuh ke DuckDB yang jauh lebih lambat.
5. **Kolom tanggal ERBA bertipe teks.** Perbandingan `>= '2026-01-01'` kebetulan bekerja secara leksikografis, tetapi `date_trunc` memerlukan `NULLIF(...,'')::timestamp`.
6. **Disiplin isi halaman tetap berlaku.** Halaman mengajarkan **cara menemukan data**, tidak pernah **jawabannya**. Seluruh angka §2 tinggal di dokumen ini; nol di antaranya boleh masuk ke `context/` atau `skills/`.
7. **Uji klaim dengan eksekusi, bukan pembacaan.** Dua koreksi di §1 lahir karena inferensi dari pola penamaan tidak diuji ke database lebih dulu. Aturan yang ditulis di atas inferensi yang belum diuji adalah cara paling efisien menanam regresi.

---

## 8. Verifikasi

### 8.1 Statis (sebelum pilot)

- [ ] Nol cacah, nol persentase, nol perbandingan besaran bocor ke `context/` atau `skills/`.
- [ ] Setiap aturan bisa dibaca tanpa mengetahui pertanyaan yang memicunya.
- [ ] Setiap aturan bersyarat punya kalimat **kapan ia tidak berlaku**.
- [ ] Tiap aturan hidup di **tepat satu** berkas — nol pengulangan lintas berkas.
- [ ] Pertumbuhan jalur panas ≤ 10 %.
- [ ] Tidak ada tautan mati, halaman yatim, atau symlink `seeknal/skills → ../skills` yang hilang.

### 8.2 Gerbang pilot per gelombang

| Metrik | v4 sekarang | Gerbang |
|---|---|---|
| PASS pada suite compact I + II | baseline v4 | **tidak turun** |
| Headline bergerak pada skenario yang sudah benar | — | **0** |
| SQL per turn · durasi median · timeout | baseline v4 | **tidak naik** |
| Baris ekspor ≠ populasi jawaban tanpa keterangan | terjadi | **0** |
| Angka berbeda untuk pertanyaan sama lintas turn | terjadi | **0** |
| Peringkat disajikan di atas bucket kosong dominan | terjadi | **0** |
| Turn berakhir tanpa jawaban karena jatah tool habis | terjadi | **0** |
| Tabel markdown gagal render | terjadi | **0** |

### 8.3 Uji ulang manual — berkas dan gambar, bukan hanya jawaban

Empat pertanyaan pemicu dijalankan ulang **setelah rollout OI1 §3.1**, dan yang diperiksa adalah unduhannya:

1. "izin edar mi instan … masih berlaku semua atau ada yang sudah lewat" → CSV memuat kedua sistem, satu baris per NIE, jumlahnya cocok dengan headline.
2. "analisis tren permohonan untuk kategori produk" → CSV mencakup seluruh rentang tahun; chart tidak melompati periode kosong.
3. "kalau seputar tahun 2026" → porsi tak terkategori dinyatakan sebelum Top-N; lingkup yang diumumkan cocok dengan isi chart.
4. "wilayah dan kategori pangan mana yang seharusnya menjadi prioritas pengawasan…" → dijawab bertahap tanpa kehabisan jatah; bucket kasar tidak diperingkat bersama kelas spesifik; komposisi sistem dinyatakan; lingkup waktu kolom seragam.

---

## 9. Tata kelola & rujukan silang

`docs/planning/README.md` melarang menambah dokumen bertanggal baru yang menduplikasi keputusan arsitektur. Dokumen ini karena itu diposisikan sebagai **amandemen** `2026-07-20-csv-store-contract.md` — memperluasnya dari "berapa kali mengekspor" menjadi "apa isi yang diekspor" — bukan sebagai spesifikasi paralel. Ia masuk ke Active Design Set hanya setelah gelombang 1–3 benar-benar diterapkan dan lolos gerbangnya.

Pemisahan context vs kode mengikuti pola `2026-08-05-sql-execution-path-and-column-type-context.md`: satu temuan, dua dokumen, tiap lapis memperbaiki apa yang hanya bisa diperbaikinya sendiri.

- **Spec engine pendamping:** `iba-deploy-runbook/specs/2026-08-13-spec-oi1-output-integrity-export-chart-fallback.md`
- **Kontrak ekspor yang diamandemen:** `2026-07-20-csv-store-contract.md` §2, §4, §5b–§5f
- **Arsitektur aktif:** `2026-08-07-routed-context-pages-architecture.md`
- **Preseden regresi akibat aturan agresif:** README varian `route-context-070826-v4` §"Perubahan v3 → v4"
- **Asal aturan cakupan (R4):** `2026-06-12-dimension-reasoning-and-data-coverage.md` §2 H1, §3
- **Jalur kanonik `nama_kategori` ERLA yang tidak boleh dirusak R4:** `2026-08-04-context-mapping-fidelity-and-coverage-closure.md` F-11
- **Asal konsep arus-vs-stok (R6):** `2026-07-13-forecast-anomaly-and-csv-era.md` §4.2
- **Preseden pelaporan kelengkapan periode:** `2026-06-18-llm-forecaster-skill.md` — blok "Kondisi Data"
