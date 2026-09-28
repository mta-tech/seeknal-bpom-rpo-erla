# Regulasi Pangan — PerBPOM No. 10/2026 Informasi Nilai Gizi (ING)

Indeks chunk regulasi ini. Semua angka acuan di folder ini adalah **data regulasi** (nilai
resmi dari peraturan), bukan angka dataset — jadi sah untuk dikutip langsung, tetapi tetap
harus disebut sumbernya (Pasal/Lampiran). Angka tentang produk nyata (jumlah NIE, dsb.)
tetap hanya boleh datang dari `execute_sql`.

Sumber: Peraturan Badan POM Nomor 10 Tahun 2026 tentang Informasi Nilai Gizi pada Label
Pangan Olahan. Ditetapkan di Jakarta, 9 Juni 2026. Mencabut PerBPOM 9/2016, 16/2020,
26/2021 (lihat `07-peralihan-pengkajian.md`).

## Peta chunk yang sudah ada

| Topik | File |
|---|---|
| Kapan dibuka, kamus istilah | `01-definisi.md` |
| Wajib / pengecualian / larangan ING | `02-kewajiban-pengecualian.md` |
| Aturan takaran saji (bukan angka per kategori) | `03-takaran-saji-aturan.md` |
| Takaran saji kategori susu & es krim | `04a-takaran-saji-susu-es.md` |
| Takaran saji kategori minuman | `04e-takaran-saji-minuman.md` |
| Nutri-Level (depan kemasan minuman) | `05-nutri-level.md` |
| Logo pilihan lebih sehat + profil gizi | `06-pilihan-lebih-sehat.md` |
| Pengkajian & masa peralihan | `07-peralihan-pengkajian.md` |

## Cakupan (jujur)

Chunk di atas adalah **pecahan pertama** dari 73 halaman dokumen: topik yang paling sering
ditanya (kewajiban, takaran saji minuman/es krim, nutri-level, profil gizi). Kelompok
kategori Lampiran II lainnya (04–06 buah/konfeksi, 07–10 bakeri/daging/ikan/telur, 12–13
bumbu/dietetis, 15–16 snack), Pasal 5–7 (format tabel ING), Pasal 12–20 (zat gizi, %AKG,
ALG), dan Pasal 21–25 (batas toleransi analisis) **belum dipecah** — jika pertanyaan
menyentuhnya, katakan bahwa chunk regulasi untuk topik itu belum tersedia; jangan mengarang
nilainya dari ingatan.

## Routing

- Pertanyaan regulasi + produk nyata (data ERBA/ERLA) → kembali ke `bpom-analyst` dan
  `00-menghitung.md`; nilai acuan diambil dari folder ini, fakta produk dari SQL.
- Klasifikasi kategori es krim vs non-dairy → `04a-takaran-saji-susu-es.md`.
