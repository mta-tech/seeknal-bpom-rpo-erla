# v4 — `september/v4-sep-w1` (ERLA-only correction)

**Created:** 22 September 2026 · **Status:** the content of `september/v3-sep-w1` plus the
verified corrections below. **The v3 variant is not modified.**

This repository is **ERLA only** (the old `ereg` registration system). Queries may touch only
four tables — `t_produk_3_rilis_erla`, `t_btp_3_erla`, `m_trader_rla`, `data_dictionary`. Tables
from other systems are **not available here** (no grant), so there is no cross-system split and
no UNION.

## Delta vs v3

1. **Entity corrections** (`SEEKNAL_ASK.md` Gate 3/5, `skills/bpom-analyst`).
   - NIE / izin edar / produk → `COUNT(DISTINCT nomor)`, date `tanggal`.
   - Permohonan / pengajuan / registrasi → `COUNT(DISTINCT produk_id)`, date `tanggal_bayar`.
   - **Persetujuan → `COUNT(DISTINCT nomor)`** (treated as an issued NIE), date `tanggal` —
     not `produk_id`.
   - Perusahaan → `COUNT(DISTINCT trader_id)`.
   - **Surat keputusan counts `nomor_surat`** — in ERLA `nomor_surat` is the real letter number
     (a `PN.…` value), **not** `produk_id` (the two coincide only rarely).
     This differs from the other system.
   - ERLA `produk_id` values begin with `EREG…`.

2. **`jenis_permohonan`** (the user's "layanan"). "baru" → `301` + `305` · "baru notifikasi" →
   `305` · "modifikasi/perubahan/variasi" → `302` + `303` · "mayor" → `302` · "minor" → `303` ·
   "daftar ulang" → `304` · no service named → no filter. A persetujuan uses the same service
   filter but the entity is `nomor`.

3. **Date panel — four choices.** `tanggal_aju` (Tanggal Permohonan) · `tanggal_bayar` (Bayar
   SPB) · `tanggal` (Tanggal Terbit NIE) · **`tanggal_hprspb` (Tanggal Terbit SPB)**.
   `tanggal_exp` also exists.

4. **`status_produk` (ERLA).** `301` Diproduksi Sendiri · `302` Impor · `303` (no dictionary
   label) · `304` Berdasarkan Kontrak · `305` (no dictionary label) · `306` Single MD Induk ·
   `307` Single MD Anak. The panel reads `303`/`305` as "Berdasarkan Kontrak Notifikasi" and
   "Diproduksi Sendiri Notifikasi", but **that mapping is not yet verified — treat it as a panel
   label only.** The **"Diproduksi Sendiri" family in ERLA = `301` + `303` + `305`.** No status
   mentioned → no status filter.

5. **Risk schema — `jenis_dokumen`, not `kategori_dokumen`.** `000` Belum Dikategorikan · `301`
   Pangan Low Risk · `302` Pangan High Risk · `303` Pangan Medium Risk; the data also carries
   `304`, undocumented. **"Risiko Tinggi" in ERLA = `jenis_dokumen = '302'`.** ERLA
   has only three levels — there is **no "Tinggi Notifikasi"**. ERLA has **no commitment columns**
   (`status_komitmen`, `jenis_penolakan_komitmen` are absent); no SQL example may use them.

6. **Region by entity.** Persetujuan/NIE per province → `daerah_pabrik`; permohonan/pengajuan per
   province → `daerah_trader`. `daerah_trader` is valid for MD and ML; `daerah_pabrik` is
   meaningful only for MD. The ERLA import marker in `daerah_pabrik` is a uniform **`'9999'`**, not `'NULL'`; `negara_pabrik` is the cleanest domestic/import separator.

7. **Scope: ERLA-only, no UNION.** `SEEKNAL_ASK.md` gained a **Scope** block that lists the four
   queryable tables and forbids naming or attempting any table outside them. The two-system
   framing was removed from the gate prose (Gate 1/3/4/5), the skills, and `seeknal_agent.yml`'s
   `prompt.custom`; the single "one final query against the ERLA tables" wording replaces the
   per-system split.

8. **Native types — no casts.** All ERLA columns are native (`timestamp` / `bigint`). Carrying a
   `NULLIF(...)::timestamp` or `::bigint` cast from the other schema is the single biggest source
   of failed queries in this repository.

## Files touched

- `SEEKNAL_ASK.md` — Scope block, Gate 1 option removed, Gate 2 P4, Gate 3 entity/tables, Gate 4
  wording, Gate 5 entity–date pairing and `jenis_permohonan` branches, follow-up example.
- `skills/bpom-analyst/SKILL.md`, `skills/bpom-forecaster/SKILL.md`,
  `skills/detect-anomaly/SKILL.md` — system-scope wording neutralized; forecast/anomaly methods
  otherwise unchanged (`visualize-chart` and `regulasi` needed no system change).
- `seeknal/skills/**` — byte-identical copies of `skills/**` after the edit.
- `seeknal_agent.yml` — `prompt.custom` only (plus one neutralized comment).
- `README.md` — this note.

## Notes

- `skills/**` and `seeknal/skills/**` are kept in sync; the copies are not symlinks in this
  variant.
- The `context/**` pages were **not** part of this change set.


## v5 (24 September 2026)

Perubahan dari v4 — tidak ada angka absolut, semua gaya mengajar metode:

1. **Komitmen Case B resmi**: pertanyaan lifecycle komitmen TIDAK menumpuk filter status NIE; `00-menghitung.md` §3 kini punya pengecualian eksplisit (menguasai bacaan NIE-subject saja).
2. **"Diubah" = `status 9999`** (kedua domain; dictionary STATUS: 9999 = Sudah Diubah; `status '0'` = Dihapus/Dibatalkan dan 0 baris).
3. **MD/ML**: MD = dalam negeri, ML = impor — prefix `nomor` menandai ASAL produk, bukan sistem; pemisah terbersih tetap `negara_pabrik`.
4. **ERLA**: label skala proses ERBA (Menengah Tinggi/Menengah Rendah) tidak ada → klarifikasi dulu, jangan dipetakan diam-diam ke Low/Medium/High.
5. **Gate 5**: butir anti-halu (setiap angka dari `execute_sql`/`run_forecast`) dan konsistensi gambar-angka-CSV dikembalikan sebagai butir 6–7.
