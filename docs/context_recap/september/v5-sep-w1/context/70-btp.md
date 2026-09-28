# BTP — Bahan Tambahan Pangan

bahan tambahan pangan, pewarna, pengawet, antioksidan, perisa, bentuk sediaan, tunggal, campuran.

BTP lives in a **separate table**: `t_btp_3_erla`. For counting purposes it behaves like the product table — same entity, same status tiers (`00-menghitung.md`).

**This page's canonical entity matches the product table: "berapa BTP / berapa NIE BTP / izin edar BTP" counts with `COUNT(DISTINCT nomor)`.** `produk_id` is for permohonan questions only. The BTP table is versioned too — one `nomor` spans many rows — so swapping the entity here inflates the answer exactly as it does on the product table.

| Concept | Column | Codes |
|---|---|---|
| Jenis BTP | `jenis_btp` | **no label anywhere** — see the warning below |
| Bentuk sediaan | `bentuk_sediaan` | `101` cair/pasta · `102` serbuk · `103` bahan penolong · `104` gas · `105` padat |
| Jenis produk BTP | `jenis_produk_btp` | `301` tunggal · `302` campuran · `303` perisa · `304` bahan penolong |

## The `jenis_btp` values carry no label

`t_btp_3_erla.jenis_btp` uses the code range **777–805**, and **no dictionary category describes those values** — the concept has **no label anywhere** in this repo. The BTP class therefore **cannot be filtered by code**: there is no code→label mapping to build a filter from, and no BTP-class code is interchangeable with any other namespace's code.

→ Before reporting 0 or "none", list the table's own values:
`SELECT DISTINCT jenis_btp, COUNT(*) FROM t_btp_3_erla GROUP BY 1`.
→ Because there is no label mapping, reporting a filter result of 0 would turn a catalog gap into a claim about the business. State the limit instead: the concept is present in the data but unreadable here.

**Do not borrow a code from another namespace or category.** If product names look contradictory to any code you do resolve, report the anomaly in one sentence — never switch to a code inferred from names: name-based re-resolution silently replaces the asked population with a different one.

## Other notes on this table

- **Column types.** `t_btp_3_erla` is fully **native** (`timestamp` / `bigint`) — **no casts**. Carrying a product-side cast over **breaks the query** (`00-menghitung.md` §4).
- **Status set.** `t_btp_3_erla` lacks `0099` and carries `0299`. Of the Verifikator 2 trio only `0502` appears. Do not copy the product table's stage list unchecked (`20-status-pipeline.md`).
- **The BTP table is not final-state only** — unlike the product ERLA table, its pipeline rows live in the same table.
- About a third of the registered `JENIS_BTP` codes hold no rows at all. They may stay in the filter, but must not be presented as contributing members.

## Products vs products+BTP scope

"How many permohonan/produk" without qualification is **ambiguous** about BTP. Present two labelled figures (products-only and products+BTP, each with its source table), or state the scope used and why. Adding the BTP table unasked is a recurring source of discrepancies.

## Routing

- Compound BTP concepts ("pewarna atau pengawet") → read the whole dictionary category, not one code (P2 path, Gate 2); repeated descriptions make a single code lose its siblings.
- BTP process stages mentioned → **see** `20-status-pipeline.md`
- BTP packaging mentioned → **see** `40-kemasan.md` (the finer `sub_kemasan_id` level is not available here)
