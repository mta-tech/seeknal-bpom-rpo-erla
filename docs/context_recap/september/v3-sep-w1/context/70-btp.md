# BTP — Bahan Tambahan Pangan

bahan tambahan pangan, pewarna, pengawet, antioksidan, perisa, bentuk sediaan, tunggal, campuran.

BTP lives in **separate tables**: `t_btp_3_erba` and `t_btp_3_erla`. For counting purposes they behave like the product tables — same entity, same status tiers (`00-menghitung.md`).

**This page's canonical entity matches the product tables: "berapa BTP / berapa NIE BTP / izin edar BTP" counts with `COUNT(DISTINCT nomor)`.** `produk_id` is for permohonan questions only. The BTP tables are versioned too — one `nomor` spans many rows — so swapping the entity here inflates the answer exactly as it does on the product tables.

| Concept | Column | Codes |
|---|---|---|
| Jenis BTP | `jenis_btp` | resolve via the dictionary — **see the namespace warning below** |
| Bentuk sediaan | `bentuk_sediaan` | `101` cair/pasta · `102` serbuk · `103` bahan penolong · `104` gas · `105` padat |
| Jenis produk BTP | `jenis_produk_btp` | `301` tunggal · `302` campuran · `303` perisa · `304` bahan penolong |

## The `jenis_btp` namespace — the biggest trap on this page

The dictionary catalogues `JENIS_BTP` as "ERLA dan ERBA" with codes **13–52**. In reality `t_btp_3_erla.jenis_btp` **does not use that range at all** — it uses **777–805**, values no dictionary category describes.

Running `jenis_btp='47'` (Pewarna) against the ERLA table returns **0 rows**. Read literally, that says "ERLA has no pewarna" — and that is wrong; it means "wrong code system for this table".

→ Before reporting 0 or "none" for one system:
`SELECT DISTINCT jenis_btp, COUNT(*) FROM <that table> GROUP BY 1`.
→ The 777–805 range **has no label anywhere**, so the concept cannot be filtered on that side. Answer for the system that maps and **state the limit**. Reporting the unmapped side as zero turns a catalog gap into a claim about the business.

**The resolved dictionary code is binding.** If product names look contradictory to the resolved code, keep the code and report the anomaly in one sentence — do not switch to a code inferred from names: name-based re-resolution silently replaces the asked population with a different one.

## Other differences from the product tables

- **Column types.** `t_btp_3_erba` is **not** all-TEXT: the four date columns are already `timestamp` and `trader_id` is already `bigint`. Carrying the product casts over **breaks the query** (`00-menghitung.md` §4). `t_btp_3_erla` is fully native.
- **Smaller status set.** `t_btp_3_erba` lacks `0009`; `t_btp_3_erla` lacks `0099` and carries `0299` (an ERLA-namespace code). Of the Verifikator 2 trio only `0502` appears. Do not copy the product tables' stage list unchecked (`20-status-pipeline.md`).
- **The BTP pipeline lives in BOTH tables** — unlike the product ERLA table, which is final-state only.
- About a third of the registered `JENIS_BTP` codes hold no rows at all. They may stay in the filter, but must not be presented as contributing members.

## Products vs products+BTP scope

"How many permohonan/produk" without qualification is **ambiguous** about BTP. Present two labelled figures (products-only and products+BTP, each with its source table), or state the scope used and why. Adding the BTP tables unasked is a recurring source of discrepancies.

## Routing

- Compound BTP concepts ("pewarna atau pengawet") → read the whole dictionary category, not one code (P2 path, Gate 2); repeated descriptions make a single code lose its siblings.
- BTP process stages mentioned → **see** `20-status-pipeline.md`
- BTP packaging mentioned → **see** `40-kemasan.md`
  (`sub_kemasan_id` exists in `t_btp_3_erba`, not in `t_btp_3_erla`)
