---
name: regulasi
description: "Routing & answer contract for regulation questions (PerBPOM 10/2026 Informasi Nilai Gizi): kewajiban ING, takaran saji, nutri-level, logo pilihan lebih sehat, klasifikasi kategori (es krim vs non-dairy). Maps each topic to its chunk under context/regulasi/regulasipangan/."
tags: [regulasi, bpom, ing, takaran-saji, nutri-level]
version: "1.0.0"
---

# Regulasi — PerBPOM No. 10/2026 (Informasi Nilai Gizi)

This skill is the **map and the answer contract** for regulation questions. The facts live in
the chunks under `context/regulasi/regulasipangan/`; this skill decides which chunk to open
and how answers must be built. `SEEKNAL_ASK.md` only routes here.

## When this skill fires

Question touches: informasi nilai gizi · label gizi · takaran saji · nutri-level · gula /
garam / lemak sebagai **batas regulasi** · logo pilihan lebih sehat · profil gizi · %AKG ·
klaim "sesuai regulasi/tidak sesuai" · es krim / gelato / dairy vs non-dairy · PerBPOM
10/2026 · kewajiban mencantumkan ING · masa peralihan.

## Map: topic → chunk (open with `read_project_file`)

| Question is about | Open |
|---|---|
| arti istilah (ING, PKMK, PKGK, dsb.) | `regulasi/regulasipangan/01-definisi.md` |
| wajib ING atau tidak · pengecualian · minuman beralkohol | `regulasi/regulasipangan/02-kewajiban-pengecualian.md` |
| aturan takaran saji (per saji, per kemasan, satuan) | `regulasi/regulasipangan/03-takaran-saji-aturan.md` |
| takaran saji es krim / gelato / non-dairy / yogurt | `regulasi/regulasipangan/04a-takaran-saji-susu-es.md` |
| takaran saji minuman (berkarbonat, tidak berkarbonat, konsentrat, kopi/teh) | `regulasi/regulasipangan/04e-takaran-saji-minuman.md` |
| nutri-level (kewajiban, ambang A–D, BTP pemanis) | `regulasi/regulasipangan/05-nutri-level.md` |
| logo pilihan lebih sehat & profil gizinya | `regulasi/regulasipangan/06-pilihan-lebih-sehat.md` |
| pengkajian, transisi 24 bulan, peraturan yang dicabut | `regulasi/regulasipangan/07-peralihan-pengkajian.md` |
| apa saja chunk regulasi yang tersedia | `regulasi/regulasipangan/00-peta.md` |

**Topic not in the map** → open `00-peta.md` first; if the topic is listed there as
"belum dipecah" (format tabel ING, zat gizi wajib, %AKG/ALG, toleransi analisis, kategori
Lampiran II lain), say the regulation chunk for that topic is not yet available — **never
supply the number from memory.**

## Answer contract

1. **Regulation-only question** → answer from the chunk + cite the basis (Pasal / Lampiran /
   kode kategori) in the answer. No SQL, no chart, no export — this is a definitional answer.
2. **Regulation + real product** (produk terdaftar, NIE, produsen) → also load
   `bpom-analyst` and follow `SEEKNAL_ASK.md` gates. The verdict = chunk's reference value +
   SQL fact + conclusion with its basis. Example: takaran saji produk vs rentang kategori.
3. **Classification cross-check** (mis. "gelato tanpa susu") → base it on the 01.7 vs 02.4
   definitions in `04a-…`. Output a **flagged candidate** ("kandidat ketidaksesuaian
   klasifikasi kategori"), never a verdict of violation; note that hard composition rules
   (SNI/product standards) are outside this regulation. If the needed product column
   (komposisi/bahan) is not in the connected schema, say the live check cannot run and give
   the method — never invent a count.
4. Numbers about the regulation (100–250 ml, ≤6 g/100 ml, …) are quoted **with their
   sumber**. Numbers about the dataset come only from `execute_sql` this turn.

## Routing (back)

- Data side of a mixed question → back to `SEEKNAL_ASK.md` Gate 0–5 via `bpom-analyst`.
- Ingredient-based questions on the current schema → `05-filter-katalog.md` (honest gap).
