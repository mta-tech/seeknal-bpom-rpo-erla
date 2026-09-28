# Packaging

botol, kaleng, plastik, kaca, keramik, karton, kertas, komposit, ganda, aluminium, PET, HDPE.

The source schema has two levels: `kemasan_id` (INDUK) and `sub_kemasan_id` (ANAK). **In this repo only the parent `kemasan_id` exists in the product table** — see the warning below.

## Parent — `kemasan_id`

The ERLA product table uses these codes:

| Code | Material |
|---|---|
| `31` | kaca |
| `32` | plastik |
| `33` | kertas/karton |
| `34` | karton laminat |
| `35` | kaleng |
| `36` | aluminium foil |
| `37` | komposit |
| `38` | ganda |
| `39` | lainnya |

⚠️ **`sub_kemasan_id` does NOT exist in `t_produk_3_rilis_erla`.** Only `kemasan_id` is available, so most **specific materials** (PET, HDPE, PVC, styrofoam, kaleng aluminium, nylon, …) have no readable column here — state that limit rather than substituting a neighbour.

**The one exception is keramik.** The dictionary label for `kemasan_id='39'` reads **"Lain-Lain (Drum, Keramik dan Lain-Lain)"**, so a keramik question *is* reachable — but only as `kemasan_id='39'`, which is a **residual bucket that also holds drum and other materials**. Answer it as an upper bound and say so; never present `39` as a clean keramik count, and do not extend the same trick to other materials.

Because the finer level is absent, the parent cannot be narrowed to a single material for a question naming one. Answer at the parent level and say that the finer granularity is out of scope here.

## Sentinels

A code whose **description is `-`, empty, or `0`** is not a material — it is an unfilled marker, and it can top the ranking. Recognise it by its **description**, not by the size of its count. Exclude it from rankings; report it separately as a data-quality note.

## Routing

- Question also mentions product segments → **see** `10-segmen-produk.md`
- Question is about BTP → **see** `70-btp.md`
- Specific-material (child-level) packaging → **see** `41-sub-kemasan.md` — out of scope in this repo
- Entity: packaging questions are almost always about NIE → `COUNT(DISTINCT nomor)`, `00-menghitung.md`
