# Packaging

botol, kaleng, plastik, kaca, keramik, karton, kertas, komposit, ganda, aluminium, PET, HDPE.

Two levels: `kemasan_id` (INDUK, 16 codes) → `sub_kemasan_id` (ANAK, 37 codes).

## Parent — `kemasan_id`, a different namespace per system

| System | Codes |
|---|---|
| **ERBA** | `1` kaca · `2` plastik · `3` kertas · `4` komposit · `5` logam · `6` lainnya · `7` ganda |
| **ERLA** | `31` kaca · `32` plastik · `33` kertas/karton · `34` karton laminat · `35` kaleng · `36` aluminium foil · `37` komposit · `38` ganda · `39` lainnya |

**No overlap.** Never use one system's code on the other — the result is 0, and that 0 means wrong namespace, not that the packaging is absent. The granularity differs too: ERLA separates kaleng from aluminium foil, ERBA merges both into `5` logam.

## When to step down to the child

**Generic materials** (kaca, plastik, kertas, logam) → stop at `kemasan_id`.
**Specific materials** (keramik, PET, HDPE, PVC, styrofoam, kaleng aluminium, nylon) → **step down** to `sub_kemasan_id`.

The reason is visible in the codes' own descriptions: the ERBA parent label `1` reads "Kaca **ATAU** Keramik" — one code shelters two materials. Whenever a parent description contains "atau" / "dan lain-lain" / names more than one material, **the parent cannot answer a question naming one of them**. To see the composition before deciding:

```sql
SELECT sub_kemasan_id, COUNT(DISTINCT nomor) FROM t_produk_3_erba
WHERE kemasan_id='<induk>' GROUP BY 1 ORDER BY 2 DESC;
```

When one child dominates its parent, answering at the parent level means answering about the dominant child — not about what was asked.

⚠️ **`sub_kemasan_id` exists only in `t_produk_3_erba` and `t_btp_3_erba`.** It is absent on the ERLA side. Specific-material questions are therefore **structurally ERBA-only** — say so, do not present them as national figures, and **do not hunt for a counterpart that is not there**: the "Lain-Lain" code in ERLA is a residual bucket, not a specific material; claiming it as a counterpart turns a catalog gap into a claim about the business.

## Parent and child need not sum to the same total

Some rows carry `kemasan_id` while `sub_kemasan_id` is empty or points elsewhere. When presenting both figures, say so; do not force them to reconcile.

## Sentinels

A code whose **description is `-`, empty, or `0`** is not a material — it is an unfilled marker, and it can top the ranking. Recognise it by its **description**, not by the size of its count. Exclude it from rankings; report it separately as a data-quality note.

## Routing

- Need the 37 child codes → **continue to** `41-sub-kemasan.md`
- Question also mentions product segments → **see** `10-segmen-produk.md`
- Question is about BTP → **see** `70-btp.md` (same packaging columns, different tables)
- Entity: packaging questions are almost always about NIE → `COUNT(DISTINCT nomor)`, `00-menghitung.md`
