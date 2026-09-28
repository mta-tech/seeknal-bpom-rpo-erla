# seeknal-bpom-neo: Keterjangkauan Halaman Anak & Penutupan Kegagalan UAT v4

**Document type:** Audit Findings + Context Change Plan
**Project:** seeknal-bpom-neo (BPOM RPO Analytics Agent)
**Status:** draft — belum diterapkan
**Date:** 2026-08-13
**Varian sasaran:** `docs/context_recap/after-chart-route/route-context-070826-v5` (v4 tidak disentuh, dipakai sebagai kontrol)
**Scope berkas (usulan):** `SEEKNAL_ASK.md` · `context/11-kode-segmen.md` · `context/20-status-pipeline.md` · `context/30-risiko-komitmen.md` · `context/80-waktu-periode.md`
**Bukti:** run `seeknal/tests/outputs/2026-08-12/v4-route-qwen-plus/` (117 turn, 9 batch) · `files_read` + `sqls` per turn · ±15 query verifikasi langsung ke `rpo_v2` (13 Agu 2026)
**Melanjutkan:** `2026-08-07-routed-context-pages-architecture.md` (temuan §2.1 adalah cacat pada arsitektur itu sendiri) · audit `docs/audit_context/2026-08-12-v3-skill-execution/`
**Berdampingan dengan:** `2026-08-13-answer-export-chart-consistency-contract.md` (subjek berbeda: jalur ekspor/chart)

---

## 1. Ringkasan Eksekutif

Run v4 pada compact I–VIII: **95 PASS / 22 FAIL (81,2 %)** — pulih dari v3 (92/116) berkat perbaikan
aturan "drop unrequested narrowing", tetapi 22 kegagalan tersisa.

Ke-22 diklasifikasi ulang dengan **`files_read` + SQL mentah per turn**, lalu **setiap klaim
diverifikasi ke `rpo_v2`**. Hasilnya membalik sebagian dugaan awal:

| Kelompok | Jumlah | Sifat |
|---|--:|---|
| **Bisa ditutup context/skill** | **9** | §2 — enam temuan |
| GT/fixture basi (SQL agent identik dengan GT) | 3 | §3 — **tidak boleh diajarkan** |
| Turn `[AUTO]` yang turn 0-nya lulus | 9 | sebagian §2.5, sebagian halusinasi/kosakata |
| Halusinasi model | 1 | dikesampingkan atas permintaan pemilik produk |

Temuan terbesarnya **struktural, bukan soal isi halaman**: halaman anak yang memuat kode kanonik
praktis tak pernah terbuka. Satu perbaikan routing menyentuh lima sampai tujuh skenario sekaligus.

---

## 2. Temuan terverifikasi

### 2.1 Halaman anak tak terjangkau dalam praktik — akar terbesar

Dihitung dari `files_read` seluruh 117 turn:

| Halaman | Dibuka | Peran |
|---|--:|---|
| `00-menghitung.md` | 117× | selalu |
| `15-permohonan.md` | 52× | topik |
| `80-waktu-periode.md` | 45× | topik |
| `10-segmen-produk.md` | **32×** | **induk segmen** |
| `20-status-pipeline.md` | 30× | topik |
| `30-risiko-komitmen.md` | 29× | topik |
| … | | |
| **`11-kode-segmen.md`** | **1×** | **anak — kode kanonik ada di sini** |
| `12-nama-kategori.md` · `21-kode-status.md` · `41-sub-kemasan.md` | **0×** | anak |

**Sebabnya struktural.** PAGE MAP di `SEEKNAL_ASK.md` **tidak memuat satu baris pun** untuk halaman
anak; keempatnya hanya terjangkau lewat blok "Rute" di kaki induknya (`TURUN`). Itu **hop kedua**,
baru mungkin setelah panggilan baca pertama selesai — sementara Gate 2 memerintahkan membuka semua
halaman yang menyala **dalam SATU panggilan**. Dua instruksi itu tidak kompatibel, dan yang menang
adalah Gate 2: agent membaca induknya, lalu langsung menulis SQL.

Pemeriksaan statis arsitektur v1 melaporkan *"keterjangkauan 115/116 = 99 %"* — itu benar sebagai
**keterjangkauan teoretis**, dan justru itu titik butanya: halaman yang bisa dijangkau ≠ halaman
yang benar-benar dibuka.

Akibatnya agent menyusun set kodenya sendiri. Terverifikasi DB:

```
UAT-BAYI-1   agent  jenis_pangan IN ('1301','1302','1303','1305','1306')  →  205   (ERBA)
             benar  jenis_pangan IN ('1301','1302')                        →   61   = GT 61
UAT-AMDK-2   benar  jenis_pangan IN ('1401','1402'), terbit 2023           → 1.774  = GT 1.774
```

Menyentuh: `BAYI-1` · `BAYI-3` · `AMDK-2` · `AMDK-3` · `GARAM-2`; kemungkinan `RED-WINE-ASAL-1` ·
`KOPI-INSTAN-PENDAFTAR-1` (dua terakhir butuh `12-nama-kategori.md` yang dibuka 0×).

### 2.2 `11-kode-segmen.md` memuat binding ERLA yang salah

Merutekan ke halaman itu saja **tidak cukup** — barisnya sendiri keliru:

> baris 13 · `| Formula bayi (ketat) | ERBA jenis_pangan IN ('1301','1302') | ERLA jenis_pangan IN ('604','622','624') |`

Sisi ERLA memberi **804 NIE**, bukan 88. Isinya:

| Kode | NIE | `nama_kategori` di dalamnya |
|---|--:|---|
| `604` | 34 | murni `Formula Bayi *` |
| `622` | **703** | Formula Bayi **+ Formula Lanjutan + Formula Pertumbuhan + Formula Khusus Untuk Anak** |
| `624` | 80 | Formula untuk Keperluan Medis Khusus — bukan formula bayi |

Bacaan ketat yang benar: `nama_kategori ILIKE 'Formula Bayi%'` → **88** = GT.

Halaman itu **sudah memperingatkan dirinya sendiri** di baris 15 — *"Formula bayi ≠ produk bayi &
anak (lebih luas, mencakup jauh lebih banyak kode)"* — lalu memberi binding yang persis melanggarnya.

Binding lain di halaman yang sama **sudah benar** (diperiksa, agar perbaikan tidak melebar):
AMDK ERLA `('651','652','655')` = 9.128 ≈ `nama_kategori` air 9.110 · garam ERLA
`kategori_pangan='12010103'` = 1.446 ≈ `nama_kategori ILIKE '%garam%'` 1.479.

### 2.3 Komitmen — tabrakan kosakata, kode hilang, entity salah

`UAT-KOMITMEN-DRAFT-MR-1` membuka **dua** halaman dan mengambil yang salah:

```
20-status-pipeline.md:13  | Draft | 0910, 0912 |     ← kolom `status`, siklus hidup NIE
20-status-pipeline.md:33  | "draft", "belum disubmit" | Draft |
30-risiko-komitmen.md     kode 0 = Draft TIDAK terdaftar
```

Terverifikasi DB:

```
agent  kategori_dokumen='303' AND status IN ('0910','0912')         →  9.756
benar  kategori_dokumen='303' AND ROUND(status_komitmen)::int = 0   → 30.343   (GT 30.026)
```

**Context aktif mengajarkan jawaban yang salah**: tidak ada satu kalimat pun yang menyatakan bahwa
"draft" yang diikuti "pemenuhan komitmen" berpindah kolom. `30-risiko-komitmen.md` §Komitmen hanya
mendaftar kode 4/5/7/8; **0 (Draft Pemenuhan Komitmen) dan 9 (Variasi Komitmen) tidak ada di halaman
mana pun.**

Entity-nya juga tak pernah dinyatakan. Pada `UAT-KOMITMEN-VARIASI-1` rumus normalisasi ROUND sudah
benar, entity-nya yang salah:

```
agent  COUNT(DISTINCT nomor)      → 4.406
benar  COUNT(DISTINCT produk_id)  → 5.064   (GT 4.931)
```

Komitmen menempel pada **pengajuan**, bukan pada NIE — satu NIE bisa punya banyak peristiwa komitmen.

### 2.4 Dua taksonomi risiko tertukar (`UAT-RISK-1`)

Ditanya *"perbandingan jumlah NIE untuk kategori **Risiko Rendah** dan Risiko Tinggi"*. Agent
menjawab dengan taksonomi ERBA 4-kelas (`kategori_dokumen`) — yang **tidak punya kelas "Rendah"** —
lalu menyajikan *"Risiko Menengah Rendah (303)"* sebagai jawabannya. GT memakai `jenis_dokumen IN
('301','302')`, taksonomi 3-kelas.

`30-risiko-komitmen.md` sudah menjelaskan bahwa ada dua kolom dan padanannya bertukar, tetapi tidak
menyatakan **kelas mana yang hanya hidup di taksonomi mana**, sehingga istilah pengguna "risiko
rendah" tidak punya jangkar dan jatuh ke kelas terdekat yang bunyinya mirip.

### 2.5 Jawaban klarifikasi diperlakukan sebagai pertanyaan baru

**9 dari 22 kegagalan** adalah turn `[AUTO]` yang **turn 0-nya LULUS**: `PIPELINE-TOTAL-1` ·
`MT-MINOR-MEI26-1` · `JP-TREN-1` · `LC-DIUBAH-1` · `MD-1` · `KOPI-INSTAN-PENDAFTAR-1` ·
`PABRIK-TOP-1` · `RED-WINE-ASAL-1` · `JP-BARU-VS-REVISI-1`.

Agent menghitung ulang dari nol dan mendarat di metode lain. `LC-DIUBAH-1`:

```
GT     status='9999'  ("sudah diubah")
agent  jenis_permohonan 302/303/304  → 158.543
```

`SEEKNAL_ASK.md` §Follow-ups hanya mengenal dua bentuk — *"narrows down from the previous answer"*
dan *"widens a dimension"*. **Bentuk "menjawab klarifikasi yang saya ajukan sendiri" tidak ada**,
padahal itu bentuk yang paling sering muncul di suite ini.

Catatan jujur: aturan context **tidak akan menutup kesembilannya**. Sebagian gagal karena assert
kosakata (`MD-1` "missing: 'MD'" padahal angkanya wajar) dan satu karena halusinasi (§3).
Perkiraan realistis: 3–4 yang benar-benar tertutup.

### 2.6 Periode telanjang tanpa tahun (`UAT-MEI-MR-1`)

Prompt: *"NIE yang terbit pada **bulan Mei** untuk kategori risiko Menengah Rendah"*.

```
GT     EXTRACT(MONTH …)=5, semua tahun  → 3.717
agent  Mei 2026 saja                    → 1.329   — dan menyebut "Mei 2026" di jawabannya
```

Pemilik produk menegaskan bacaan **tahun berjalan** yang tepat. Maka **agent tidak salah**, dan ia
bahkan transparan. Yang perlu diperbaiki: **kanonkan bacaannya** supaya deterministik lintas sesi
(kontrak konsistensi), dan **GT-nya diperbarui**. Aturan ini juga menutup arah sebaliknya — agent
yang diam-diam menyempit ke satu tahun tanpa mengatakannya.

---

## 3. Yang TIDAK boleh diajarkan

Tiga kegagalan di mana **SQL GT-nya valid dan kanonik — tetapi nilai yang dipatok sudah melenceng
di luar toleransinya sendiri.** Dibuktikan dengan menjalankan SQL dari `note` **apa adanya**:

| Skenario | `assert_any_of` | tol 10 % | **SQL GT dijalankan 13 Agu** | agent (12 Agu) |
|---|--:|---|--:|--:|
| `UAT-COM-2` | 37 | 33–41 | **108** | 104 |
| `UAT-PIPE-VERIF2-1` | 206 | 185–227 | **341** | 297 |
| `UAT-PIPELINE-DIR-1` | 508 | 457–559 | **981** | 1.310 |

**Query kanonik milik tes itu sendiri gagal memenuhi assertion-nya sendiri.** Tidak ada aturan
context yang bisa mengubah 341 menjadi 206.

SQL agent **identik** dengan SQL GT; satu-satunya klausa tambahannya (`TRIM(status)<>''`) diuji dan
berpengaruh nol (108/341/981 dengan maupun tanpa).

**Bukti independen** bahwa penyebabnya pergerakan antrean, bukan filter yang salah:

- Antrean `0600` memuat pengajuan **2023-10-23 s/d 2026-07-31**; **35 dari 981** masuk setelah
  `verification_date` (27 Jul) — tahap ini aktif diisi.
- Komposisi `VERIF2` bergeser di dalam kode yang sama: `0500` 86 → **237**, `0502` 113 → **92** —
  butir berpindah antar-tahap.
- **Bentuk strukturnya tetap persis seperti deskripsi GT**: `0501`/`0503` masih nol baris,
  `PIPELINE-DIR` masih 100 % berasal dari `0600` saja. Yang berubah isinya, bukan bindingnya.
- Note-nya sendiri merekam ayunan sebelumnya: DIR *"Nilai lama 872"* (872 → 508 → 981) ·
  VERIF2 *"beberapa jam sebelumnya totalnya 214"* · COM-2 *"Nilai lama 42"*.

Menambahkan aturan context supaya agent menghasilkan angka yang dipatok berarti **mengajarkan angka
yang salah demi lolos tes** — kebalikan dari tujuan latihan ini. → **refresh nilai fixture**
(SQL-nya dipertahankan), bukan perubahan context.

**Batas yang tidak diklaim:** apakah angka agent pada 12 Agu benar saat itu **tidak dapat
diverifikasi** — tidak ada tabel riwayat status pipeline, jadi keadaan masa lalu tidak bisa
direkonstruksi. Pada `PIPELINE-DIR` angka agent (1.310) bahkan lebih tinggi daripada hari ini (981).
Yang dibuktikan di sini hanya satu hal, dan itu cukup: **tes ini gagal hari ini bahkan bila agent
menghasilkan SQL kanoniknya dengan sempurna.**

### 3.1 Uji simetris — metode yang sama diterapkan ke kelompok "agent salah"

Klaim "fixture melenceng" hanya sah bila metodenya juga diuji pada fixture yang diklaim **sehat**.
SQL GT verbatim, dijalankan hari yang sama ke DB yang sama:

| Skenario | SQL GT (13 Agu) | `assert_any_of` (tol) | Di dalam? |
|---|--:|---|:--|
| `UAT-BAYI-1` | **149** | 149 (142–156) | ✅ **persis** |
| `UAT-AMDK-2` | **1.774** | 1.774 (1.685–1.863) | ✅ **persis** |
| `UAT-KOMITMEN-DRAFT-MR-1` | 30.808 | 30.026 (27.023–33.029) | ✅ |
| `UAT-KOMITMEN-VARIASI-1` | 5.064 | 4.931 (4.438–5.424) | ✅ |
| `UAT-COM-2` | 108 | 37 (33–41) | ❌ 2,9× |
| `UAT-PIPE-VERIF2-1` | 341 | 206 (185–227) | ❌ 1,65× |
| `UAT-PIPELINE-DIR-1` | 981 | 508 (457–559) | ❌ 1,93× |

Empat fixture masih tepat pada assertion-nya sendiri setelah tiga minggu — dua **persis**. Bila
metode verifikasinya cacat, keempatnya ikut melenceng; nyatanya tidak. Pembedanya bukan usia
fixture melainkan **sifat populasinya**: NIE terbit itu stabil, antrean pipeline bergerak dua arah.

Maka pembagiannya berdiri di atas bukti simetris, bukan asumsi: **fixture sehat → agent yang salah,
bisa diajarkan** (§2.1–§2.3); **fixture melenceng → tidak ada yang bisa diajarkan** (§3).

### 3.2 Fixture sudah diperbarui (13 Agu 2026) — SQL dipertahankan, nilai disegarkan

| Berkas | assertion | toleransi | `verification_date` |
|---|--:|--:|---|
| `UAT-v2-compact-VI/UAT-COM-2.yml` | 37 → **108** | 10 % | 27 Jul → **13 Agu** |
| `UAT-v2-compact-VI/UAT-PIPE-VERIF2-1.yml` | 206 → **341** | 10 → **15 %** | 27 Jul → **13 Agu** |
| `UAT-v2-compact-VI/UAT-PIPELINE-DIR-1.yml` | 508 → **981** | 10 → **15 %** | 27 Jul → **13 Agu** |

SQL kanonik, `assert_contains`, dan struktur kunci **tidak diubah** — yang disegarkan hanya nilai,
toleransi, tanggal, dan rincian di `note`. Toleransi dua skenario pipeline dinaikkan ke 15 % karena
riwayatnya menunjukkan ayunan ratusan baris dalam hitungan jam; 10 % terlalu ketat untuk antrean
yang bergerak dua arah.

Setiap `note` kini membawa **riwayat nilainya** (COM-2: 42 → 37 → 108 · VERIF2: 159 → 214 → 206 →
341 · DIR: 872 → 508 → 981) dan satu kalimat penutup: *bila angka meleset jauh, jalankan SQL di atas
sebelum menyalahkan agent.* Itu mencegah audit berikutnya mengulang kesimpulan yang salah.

### 3.3 Yang justru TERVALIDASI oleh penyelidikan ini

Ketiga skenario ini menunjukkan context **bekerja**, bukan gagal:

| Yang dipakai agent | Sumbernya di context |
|---|---|
| `status IN ('0500','0502','0504')` | `20-status-pipeline.md:10` — *"persis ini; `0501, 0503` tak berbaris di ERBA"* |
| `status IN ('0600','0601','0666')` | `20-status-pipeline.md:11` |
| `ROUND(status_komitmen::numeric)` pada kode 8 | `30-risiko-komitmen.md:68` — kode 8 sudah terdaftar terdampak |

Aturan ROUND bahkan **terbukti secara prediktif**: per 27 Jul kode 8 belum punya varian `'8.0'`, dan
normalisasi tetap ditulis "karena bisa muncul kapan saja". Per 13 Agu varian itu muncul — **10 dari
108 baris**. Tanpa ROUND jawaban kehilangan 9,3 %. Ini alasan konkret untuk mempertahankan gaya
menulis aturan yang defensif terhadap bentuk data, bukan reaktif terhadap keadaan hari ini.

**Kesimpulan: nol perubahan context untuk ketiga skenario ini.** Menambah aturan di atas aturan yang
sudah benar hanya mengencerkan yang sedang bekerja.

`UAT-PABRIK-TOP-1` turn 1 memproduksi daftar merek terkenal dengan angka fiktif (Mayora 2.847,
Indofood 2.156, Wings 1.923) — pola yang sama persis dengan yang dikutip
`specs/2026-08-12-spec-ms1-seeknal-model-settings-temperature.md` §1. Dikesampingkan atas permintaan
pemilik produk; ranahnya `model_settings`, bukan context.

---

## 4. Perubahan yang diusulkan

Tujuh perubahan, seluruhnya **aditif**, di varian v5.

| # | Berkas | Isi |
|---|---|---|
| **P1** | `SEEKNAL_ASK.md` PAGE MAP | Baris pemicu untuk `11-kode-segmen` · `12-nama-kategori` · `21-kode-status` · `41-sub-kemasan`, **plus** satu kalimat: halaman anak dibuka **bersama** induknya di panggilan yang sama, bukan sesudahnya. Menutup ketidakcocokan Gate 2 ↔ blok Rute |
| **P2** | `context/11-kode-segmen.md` | Baris Formula bayi dipecah dua: **ketat** = ERBA `('1301','1302')` · ERLA `nama_kategori ILIKE 'Formula Bayi%'`; **keluarga luas** (formula lanjutan/pertumbuhan/khusus anak) = ERLA `('604','622','624')`, dengan kalimat kapan memakai yang mana |
| **P3** | `context/30-risiko-komitmen.md` | Lengkapi daftar kode `status_komitmen`: **0** Draft Pemenuhan Komitmen · **9** Variasi Komitmen. Nyatakan **entity komitmen = `produk_id`**. Satu baris pembeda: kata tahapan ("draft", "variasi", "dibatalkan") yang **diikuti kata komitmen** milik `status_komitmen`, bukan `status` |
| **P4** | `context/20-status-pipeline.md` | Penunjuk balik dua baris: "draft/dibatalkan **pemenuhan komitmen**" bukan tahapan pipeline → **SEBERANG** `30-risiko-komitmen.md`. Dipasang di halaman yang **memenangkan** routing, bukan hanya di halaman tujuan |
| **P5** | `context/30-risiko-komitmen.md` | Jangkar istilah: kelas **"Rendah"** hanya ada pada taksonomi 3-kelas `jenis_dokumen`; taksonomi 4-kelas `kategori_dokumen` tidak memilikinya. "Risiko rendah" **tidak boleh** dijawab dengan "Menengah Rendah" — bila hanya taksonomi 4-kelas yang tersedia, katakan keterbatasannya |
| **P6** | `SEEKNAL_ASK.md` §Follow-ups | Bentuk ketiga: **menjawab klarifikasi turn sebelumnya**. Bukan pertanyaan baru — bawa subjek, entity, dan filter yang sudah tegak di turn 0; ubah **hanya** sumbu yang ditanyakan. Angka tetap dari query turn ini |
| **P7** | `context/80-waktu-periode.md` | Periode telanjang tanpa tahun = **tahun berjalan**; sebutkan tahun yang dipakai. Kata "sepanjang/semua tahun/sejak" membatalkan default |
| **P8** *(opsional, prioritas rendah)* | `context/20-status-pipeline.md` | Angka yang menggambarkan **antrean atau tahap antara** disertai tanggal keadaannya. Populasi seperti ini bergerak dua arah dalam hitungan jam (§3), jadi angka tanpa tanggal menyesatkan pembaca dan membuat dua jawaban yang sama-sama benar tampak bertentangan. **Perluasan** kalimat yang sudah ada di `SEEKNAL_ASK.md:189` ("state the as-of date") ke pemicu baru — bukan aturan baru |

**Di luar lingkup context**, dicatat sebagai tindak lanjut terpisah: refresh fixture `COM-2` ·
`PIPE-VERIF2-1` · `PIPELINE-DIR-1` · `MEI-MR-1`; kasus halusinasi `PABRIK-TOP-1`.

---

## 5. Analisis risiko regresi

Preseden yang wajib dijaga: **v3 mengaktifkan skill dan pass-rate turun 95 → 92** karena satu aturan
terlalu agresif. Tujuh perubahan berarti tujuh peluang mengulanginya.

### 5.1 Risiko per perubahan

| # | Yang bisa jadi lebih buruk | Mekanisme | Pagar | Cara mendeteksi |
|---|---|---|---|---|
| **P1** | **Risiko tertinggi.** Empat halaman anak (204 baris) ikut terbuka pada pertanyaan yang tidak membutuhkannya → biaya token & waktu naik | Pemicu di PAGE MAP terlalu longgar | Pemicunya **sempit dan spesifik** — `11` hanya untuk segmen berkode/varian spesifik, `12` hanya untuk teks bebas yang perlu probe, `21` hanya saat kode↔label perlu dipetakan, `41` hanya saat butuh 37 kode anak kemasan. **Bukan** kata segmen umum | Rata-rata halaman/turn **naik ≤ 1**; durasi median & SQL/turn **tidak naik** |
| **P1** | Agent membuka anak lalu mengabaikan induk | Dua halaman sekaligus, aturan bertabrakan | Halaman anak sudah punya blok Rute **KEMBALI**; tidak ada aturan induk yang dihapus | Skenario segmen yang hari ini PASS **tetap PASS** |
| **P2** | Pertanyaan "produk bayi & anak" (luas) jadi dijawab sempit 88 | Dua baris tertukar | Label eksplisit **ketat** vs **keluarga luas** + kalimat pemilih; `10-segmen-produk.md` sudah menyuruh **bertanya** bila ambigu — tidak disentuh | `BAYI-1`/`BAYI-3` → 149; skenario bayi-luas (bila ada) tidak menyempit |
| **P3/P4** | Pertanyaan pipeline murni dialihkan ke halaman komitmen | Pemicu "draft" sendirian | Pemicunya wajib **dua kata** ("draft **komitmen**", "dibatalkan **komitmen**") — kata tunggal tetap milik pipeline | `PIPE-VERIF2-1` & `PIPELINE-DIR-1` **tetap** memakai kolom `status` (bukan `status_komitmen`) |
| **P3** | Entity `produk_id` diterapkan ke pertanyaan komitmen yang memang menanyakan NIE | Aturan dibaca universal | Dinyatakan sebagai **default untuk peristiwa komitmen**, dan tetap tunduk pada §1 `00-menghitung.md` (entity dari SUBJEK) | Skenario komitmen yang hari ini PASS **tidak bergerak** |
| **P5** | Agent jadi menolak menjawab pertanyaan risiko yang sebenarnya bisa dijawab | "Tidak boleh dijawab dengan Menengah Rendah" dibaca sebagai larangan menjawab | Aturan menyuruh **menyatakan keterbatasan**, bukan berhenti; jalur "jawab dengan yang ada + sebut batasnya" tetap terbuka | Turn tanpa jawaban = **0**; 8 skenario risiko lain tetap PASS |
| **P6** | Agent enggan mengklarifikasi karena takut turn kedua | Aturan disalahartikan sebagai kritik atas klarifikasi | Aturan mengatur **cara menjawab** klarifikasi, **bukan kapan bertanya**; Gate 1 tidak disentuh sama sekali | Jumlah `ask_user_calls` **tidak turun** |
| **P7** | Pengguna memaksudkan seluruh riwayat tapi dijawab tahun berjalan | Default terlalu kaku | Klausa pembatal eksplisit ("sepanjang tahun", "semua tahun", "sejak"); tahun yang dipakai **selalu disebut** sehingga salah baca terlihat pengguna | Skenario tren multi-tahun **tidak menyempit** |

### 5.2 Risiko sistemik

| Risiko | Pagar |
|---|---|
| **Pengenceran instruksi** — tujuh aturan menurunkan kepatuhan pada aturan lama | v5 sudah memuat 8 aturan dari kontrak ekspor/chart. **Terapkan dua gelombang**: P1+P2 dulu (satu akar, dampak terbesar, paling mudah diatribusi), P3–P7 setelah gelombang 1 terukur |
| **Duplikasi aturan → tool call ganda** (terbukti `2026-07-20` §5c) | Tiap aturan hidup di **tepat satu** berkas. P4 adalah **penunjuk**, bukan salinan P3 |
| **Jalur panas membengkak** | P1 & P6 menambah ~8 baris di `SEEKNAL_ASK.md`; P2/P3/P5/P7 seluruhnya di halaman **bersyarat**. Jalur panas naik < 2 % |
| **Aturan menyala di kasus yang salah** | Tiap aturan bersyarat wajib punya **kalimat kapan ia TIDAK berlaku** (sudah dirancang di P2, P3, P4, P7) |
| **Mengajarkan angka yang salah demi lolos tes** | §3 dikunci: tiga skenario GT-basi **tidak boleh** memicu aturan apa pun |

### 5.3 Perubahan bersifat aditif

Nol baris v5 yang ada dihapus atau ditulis ulang. Konsekuensinya: aturan yang hari ini menghasilkan
95 PASS tetap berbunyi persis sama. Satu pengecualian yang **disengaja** — P2 memperbaiki baris
Formula bayi ERLA yang terbukti salah (§2.2); itu koreksi fakta, bukan penambahan aturan, dan
skenario yang bergantung padanya (`BAYI-1`, `BAYI-3`) memang sedang GAGAL.

---

## 6. Verifikasi

### 6.1 Statis (sebelum run)

- [ ] Nol cacah/persentase/perbandingan besaran bocor ke `context/` atau `skills/`.
- [ ] Tiap aturan bersyarat punya kalimat "kapan tidak berlaku".
- [ ] Tiap aturan hidup di satu berkas; P4 hanya menunjuk.
- [ ] Jalur panas naik < 2 %.
- [ ] Symlink `seeknal/skills → ../skills` hidup; keempat skill terbaca.
- [ ] v4 tetap tak tersentuh (pembanding).

### 6.2 Ulang-jalan suite

v5 vs v4 sebagai kontrol, compact I–VIII, model dan `workers` sama:

```bash
cd seeknal-bpom-neo && uv run python scripts/test_variant_compare.py \
  --variants-path docs/context_recap/after-chart-route \
  --variants route-context-070826-v5 --variants route-context-070826-v4 \
  --test-path seeknal/tests/v1/singleturn/UAT-v2-compact  … hingga -VIII \
  --workers 1 --timeout 400
```

| Gerbang | Target |
|---|--:|
| **PASS keseluruhan** | **> 95/117** (baseline v4) |
| **95 skenario yang hari ini PASS** | **nol berubah jadi FAIL** ← paling menentukan |
| Headline pada skenario yang sudah benar | **nol bergerak** |
| `11-kode-segmen.md` dibuka pada pertanyaan bersegmen | **≥ 25/32** (kini 1/32) |
| `BAYI-1` · `BAYI-3` · `AMDK-2` · `GARAM-2` | **PASS** |
| `KOMITMEN-DRAFT-MR-1` · `KOMITMEN-VARIASI-1` · `RISK-1` | **PASS** |
| `PIPE-VERIF2-1` · `PIPELINE-DIR-1` | tetap memakai kolom `status` (bukan komitmen) |
| SQL/turn · durasi median · halaman dibuka/turn | **tidak naik** (halaman ≤ +1) |
| `ask_user_calls` | **tidak turun** |

### 6.3 Verifikasi silang ke `rpo_v2`

Kolom PASS/FAIL harness tidak cukup — `_token_in_answer` memindai seluruh teks jawaban dan bisa
meluluskan jawaban salah (audit compact-VIII: 71 % PASS palsu). Bandingkan **headline jawaban**
dengan `note` GT, lalu cek angka kuncinya langsung:

| Skenario | Angka yang harus muncul |
|---|--:|
| `BAYI-1` / `BAYI-3` gabungan | **149** (ERBA 61 + ERLA 88) |
| `AMDK-2` | **1.774** |
| `KOMITMEN-DRAFT-MR-1` | **≈ 30.343** |
| `KOMITMEN-VARIASI-1` | **≈ 5.064** |

---

## 7. Tata kelola & rujukan silang

`docs/planning/README.md` melarang dokumen bertanggal baru yang menduplikasi keputusan arsitektur.
Dokumen ini **tidak menduplikasi** — ia melaporkan cacat pada arsitektur yang sudah ditetapkan
(`2026-08-07` §keterjangkauan) dan mengusulkan perbaikannya, jadi ia **melanjutkan** dokumen itu.
Masuk Active Design Set hanya setelah gelombang 1–2 lolos gerbang §6.

- **Arsitektur yang dikoreksi:** `2026-08-07-routed-context-pages-architecture.md`
- **Audit pendahulu:** `docs/audit_context/2026-08-12-v3-skill-execution/` (R1-nya sudah mendarat di v4)
- **Kontrak berdampingan di varian yang sama:** `2026-08-13-answer-export-chart-consistency-contract.md`
- **Ranah `model_settings` (halusinasi):** `iba-deploy-runbook/specs/2026-08-12-spec-ms1-seeknal-model-settings-temperature.md`
- **Sumber bukti run:** `seeknal/tests/outputs/2026-08-12/v4-route-qwen-plus/`
