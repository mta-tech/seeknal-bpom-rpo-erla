# Data Quality — "belum / tanpa / kosong / tidak punya"

belum ditetapkan, belum dikategorikan, tidak terisi.

A question saying "belum / tanpa / kosong / tidak punya" asks about the raw state of the population.

## The one rule governing this entire class

**Drop the valid-NIE status filter. Drop `jenis_permohonan`. Drop date ranges the question did not ask for.**

The reason is structural: unclassified rows have generally **not reached NIE issuance yet**, so stacking a valid-NIE filter on top filters out exactly the rows being sought.

The entity follows the subject as usual (`00-menghitung.md` §1); the test-account exclusion still applies.

## Which column for "belum dikategorikan" — answer the column the USER named

Several columns can equally mean "category", and their fill rates differ enormously — the same question can come back "none" or rank among the largest groups, purely because of which column was read. So: map the user's term to its column, then check that column's fill rate.

**This is the one page where filtering for emptiness is actually correct** — the question is about the empty column, so it passes the test in `00-menghitung.md` §3. On other pages, a fill-guard is unrequested narrowing. The difference lies not in the column but in what is being asked.

| User term | Column |
|---|---|
| "belum ditetapkan **kategori risiko**" | `jenis_dokumen = '000'` — this is the business concept, not an empty column |
| "belum dikategorikan **jenis pangan**" | `jenis_pangan` / `kategori_pangan` empty |
| its free-text catalog not filled in | `nama_kategori` empty |
| a data artifact | `kategori_dokumen` empty — stored as **NULL, not an empty string**; test with `IS NULL` |

Check before answering:
```sql
SELECT COUNT(*) total,
       COUNT(*) FILTER (WHERE NULLIF(TRIM(<col>),'') IS NULL) kosong
FROM <table>;
```

**Name the column you used** — that is what makes the answer verifiable. And distinguish a **migration artifact** from a **business state**: a column empty across almost an entire legacy system usually means it was never filled during migration, not that the products are genuinely unclassified. If that is the pattern, say so.

## Zero is an answer

A correct query returning zero rows → say plainly "none / not found". That is an honest result, not a failure to fix. Do not invent a number to fill the gap, and do not widen the filter until "something" appears.

## Sentinels are not categories

Values like `'0'`, `''`, a `'-'` description, or a date far older than the system itself are **unfilled markers**, not categories — and they often **top the ranking**.

How to recognise one, rather than memorise them:
- no row in `data_dictionary` for that column's category, **or** an empty/`-` description;
- its meaning is "no particular treatment" or it names an organisational unit, not a product attribute;
- a large, disproportionate count next to the other members (one `GROUP BY` shows it).

Exclude them from rankings; report them separately as a data-quality note. A ranking built without excluding them crowns "unfilled" as the winner — an answer that means nothing.

**Check fill rates on the population being asked about, not in general**, before presenting a ranking or a "largest". One `GROUP BY` is enough to see it.

When the empty/unfilled group **dominates** the result, excluding it is not enough — **state its share as part of the answer**, before the Top-N. A Top-N over the small remainder describes a sliver of the population while reading as if it described all of it. Switch columns only when the other column answers the same question, and say so when switching.

**Fill rates can differ sharply between systems for the same column.** Check per system; a combined figure hides the fact that the issue is specific to one side.

## Routing

- Concept is the risk category → **see** `30-risiko-komitmen.md` (its column is `jenis_dokumen`, not `kategori_dokumen`)
- Concept is a food segment → **see** `10-segmen-produk.md`
- The column being asked about appears on no page → `95-dimensi-lain.md`
