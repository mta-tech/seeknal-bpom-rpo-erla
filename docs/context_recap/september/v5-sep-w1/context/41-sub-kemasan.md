# Sub-Packaging — not available in this repo

PET, HDPE, PVC, keramik, styrofoam, kaleng, nylon, akrilik.

The `sub_kemasan_id` column **does not exist** in this repo's product table (`t_produk_3_rilis_erla`). The child level of the packaging dimension is therefore **out of scope here**: a question naming a specific material (PET, HDPE, PVC, styrofoam, aluminium foil, nylon, akrilik, …) **cannot be answered** against the ERLA table — say so. **Keramik is the single exception**: it appears in the dictionary label of `kemasan_id='39'` ("Lain-Lain (Drum, Keramik dan Lain-Lain)"), so it can only be approximated by that residual bucket, never isolated.

- Answer packaging questions at the **parent** level (`kemasan_id`) instead → `40-kemasan.md`.
- Do **not** substitute the parent's `39` "Lain-Lain" residual code for a specific material — that turns a catalog gap into a claim about the business.
- Do **not** go looking for the column under another name; it is simply absent.

## Reference — the child codes (context only, not queryable here)

| Family | Codes |
|---|---|
| **Kaca/Keramik (1xx)** | `101` Kaca · `102` Keramik |
| **Plastik (2xx)** | `201` PET · `202` HDPE · `203` PVC · `204` LDPE/LLDPE/HDPE/PE · `205` PP/OPP/BOPP/CPP · `206` PS/EPS/Styrofoam · `207` PC · `208` Nylon/PA · `209` PLA · `210` Melamin · `211` PVDC · `212` EVOH · `213` PMMA/Akrilik · `214` Lain-lain |
| **Kertas (3xx)** | `301` Kertas · `302` Karton · `303` Kardus |
| **Komposit (4xx)** | `401` Plastik/Aluminium Foil · `402` Plastik/Aluminium Metalized · `403` Kertas/Plastik · `404` Plastik/Aluminium/Kertas (Karton Laminat) · `405` Kertas/Aluminium (Can Komposit) · `406` Plastik/Plastik (Multilayer/Laminat) · `407` Campuran ≥2 jenis lain |
| **Logam (5xx)** | `501` Kaleng Fe/Baja · `502` Kaleng Aluminium · `503` Aluminium Tunggal · `504` Logam Lainnya |
| **Alami (6xx)** | `601` Kayu · `602` Bambu · `603` Kain · `604` Karet · `605` Lilin/Wax · `606` Lainnya |
| **7xx** | `701` description is **`-`** — not a material |

This list is a reference sheet only. Because the column is absent, none of these codes can be joined or filtered in this repo.

**Sentinels.** A code whose description is `-`, `''`, or `0` is not a material — it is an unfilled marker. Recognise it by its **description**, not by its size.

## Routing

- Back to `40-kemasan.md` for the parent `kemasan_id` values that **are** available here.
- Specific-material questions are **out of scope** in this repo — state that limit rather than answering at the parent level silently.
