# Sub-Packaging — 37 specific material codes

PET, HDPE, PVC, keramik, styrofoam, kaleng, nylon, akrilik.

The `sub_kemasan_id` column, **only in `t_produk_3_erba` and `t_btp_3_erba`** — absent on the ERLA side.

## Code map (code → label; take the counts from a query, not from this page)

| Family | Codes |
|---|---|
| **Kaca/Keramik (1xx)** | `101` Kaca · `102` Keramik |
| **Plastik (2xx)** | `201` PET · `202` HDPE · `203` PVC · `204` LDPE/LLDPE/HDPE/PE · `205` PP/OPP/BOPP/CPP · `206` PS/EPS/Styrofoam · `207` PC · `208` Nylon/PA · `209` PLA · `210` Melamin · `211` PVDC · `212` EVOH · `213` PMMA/Akrilik · `214` Lain-lain |
| **Kertas (3xx)** | `301` Kertas · `302` Karton · `303` Kardus |
| **Komposit (4xx)** | `401` Plastik/Aluminium Foil · `402` Plastik/Aluminium Metalized · `403` Kertas/Plastik · `404` Plastik/Aluminium/Kertas (Karton Laminat) · `405` Kertas/Aluminium (Can Komposit) · `406` Plastik/Plastik (Multilayer/Laminat) · `407` Campuran ≥2 jenis lain |
| **Logam (5xx)** | `501` Kaleng Fe/Baja · `502` Kaleng Aluminium · `503` Aluminium Tunggal · `504` Logam Lainnya |
| **Alami (6xx)** | `601` Kayu · `602` Bambu · `603` Kain · `604` Karet · `605` Lilin/Wax · `606` Lainnya |
| **7xx** | `701` description is **`-`** — not a material; see "sentinels" below |

This list is a cheat sheet, not the universe of codes. For codes missing here:
`SELECT kode, deskripsi FROM data_dictionary WHERE kategori='SUB_KEMASAN_ID'`.

## Usage rules

- **The `4xx` codes collide across categories.** Values `401`–`407` also live in the `STATUS` category (sumber ERLA) and in the `pengolahan` column. Always match a code together with **its category AND column**; a bare code means nothing.
- **Sentinels.** A code whose description is `-`, `''`, or `0` is not a material — it is an unfilled marker, and it often **tops the ranking**. Recognise it by its description, not its size. Exclude it from rankings; report it separately as a data-quality note.
- **Codes with zero rows may stay in the filter** — it costs nothing and survives future fills — but do not present them as contributing members. One `GROUP BY` shows which ones carry rows.
- **Compound concepts take all their members**: specific "plastik" = the whole `2xx` family · "kaleng" = `501`+`502` · "komposit/laminat" = `4xx`. One code from a family undercounts.
- **Parent and child need not sum to the same total.** Some rows carry `kemasan_id` while `sub_kemasan_id` is empty or points elsewhere. When presenting both figures, say so; do not force them to reconcile.

## Routing

- **Back to** `40-kemasan.md` if the question is about generic materials or needs the ERLA side.
- The ERLA side has no such column → specific-material questions are **structurally ERBA-only**; state that limit, do not hunt for a counterpart that does not exist.
