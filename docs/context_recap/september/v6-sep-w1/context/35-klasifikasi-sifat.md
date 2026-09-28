# Classification & Product Attributes

kategori makanan, kategori minuman, berklaim, organik, diet, herbal, iradiasi, GMO, peruntukan khusus.

## "Kategori makanan / minuman" has two meanings — pick one before writing SQL

The same words point at two entirely different columns:

| Reading | Column | When |
|---|---|---|
| **Coded class** (this page) | `klasifikasi_id` `301`/`302` | "kategori makanan", "produk minuman", "klasifikasi", the food-vs-drink split |
| **Free-text food segment** (`10-segmen-produk.md`) | `nama_kategori` free text | concrete food types: kopi, roti, AMDK, susu, mi |

The test: is the question asking for an official CLASS, or a food TYPE? Class → the coded column on this page. Type → free text on page `10`. `nama_kategori ILIKE '%makanan%'` is not a way to count the Makanan class — it scans product names rather than classifications, and misses every product whose name does not contain the word.

If both readings seem equally plausible → Gate 1, ask.

## `klasifikasi_id` — more classes than the food/drink split suggests

| Code | Class |
|---|---|
| `3` | Deputi 3 (Pangan) — **a residual bucket, not a business class** |
| `301` | Makanan |
| `302` | Minuman |
| `303` | Bahan Tambahan Pangan |
| `304` | Minuman Beralkohol |
| `305` | **Pangan Berklaim** |
| `306` | Pangan Dengan Herbal |
| `307` | Pangan Iradiasi |
| `308` | Pangan Rekayasa Genetika |
| `309` | Organik — **a decoy**, see the bindings below |
| `310` | Pangan Diet |
| `311` | Pangan Bayi & Anak |
| `312` | Pangan Ibu Hamil & Menyusui |

Some classes may hold no rows at all yet. That does not make them nonexistent — a single `SELECT klasifikasi_id, COUNT(*) … GROUP BY 1` shows which ones; empty codes may stay in the filter but must **not** be presented as contributing members in the answer.

## The residual bucket — a two-sided trap, avoid both sides at once

`klasifikasi_id='3'` "Deputi 3 (Pangan)" is not a business class; it is the organisational unit that owns the record. How to recognise one: the **description names an organisational unit or a default, not a product attribute**, and its count is large next to the surrounding classes (a single `GROUP BY` shows it).

- Do not answer a class question with the residual code. "How many Makanan products" = `klasifikasi_id='301'` and nothing more. Folding the directorate bucket into Makanan inflates the answer with records that were never classified as food.
- Do not present the individual classes as if they exhaust the population either. Makanan + Minuman is not all of the registered products, because one large block sits unclassified in `3`.

**The way out:** compute the residual share **in the same query** as the breakdown and present it as a labelled row. The answer stays honest about coverage, and every figure still comes from this turn's query.

The general shape: when a code holds a large share of its family **and** its description names an organisational unit or a default (not a business class), treat it as a residual bucket — not an answer, and not invisible either: a **labelled remainder**. `pemrosesan='300'` "Tanpa Proses Tertentu" is the same case.

## Processing attributes — `pemrosesan`

`300` Tanpa Proses Tertentu (residual) · `301` **Organik** · `302` Rekayasa Genetik (GMO) ·
`303` — · `304` **two different descriptions** in the dictionary ("Pangan Very Low Risk" and "Iradiasi")
— an internal collision; mention it if you use the code.
Catalogued in dictionary category `PEMROSESAN`.

## Peruntukan — two valid readings, one headline

`peruntukan`: `0000` umum · `0201` **khusus**. The data also carries undocumented non-general codes (`0103`/`0104`/`0105`/`0106`, plus `010101` in ERLA) that are neither.

Lead with **`peruntukan='0201'`** — that is the business-defined code. If the question asks for *all* specially-designated products, attach "everything except `0000`" as a labelled companion figure and name the undocumented codes it picks up. What is not allowed is silently choosing the wider reading — the number moves and nothing explains why.

## Fixed bindings — never swap these

The wrong side returns a **plausible but wrong** number, not an error:

| Concept | Use | Not |
|---|---|---|
| berklaim | `klasifikasi_id='305'` | the `klaim` column (free text) |
| organik | `pemrosesan='301'` | `klasifikasi_id='309'` — same name, very different population |
| peruntukan khusus | `peruntukan='0201'` | `'0000'` — that is the **umum** code, the opposite of what was asked |
| impor | `status_produk='302'` | `302` in other columns (`jenis_permohonan`=mayor) |
| makloon / kontrak | `status_produk='304'` | the `status` column (workflow) |
| company ranking | `m_trader_rla.nama` via `trader_id` | `nama_perusahaan` in the product tables (does not exist) |

## Routing

- Concept is impor / makloon / country of origin → **see** `60-asal-produksi.md`
- Concept is **risk** category (not classification) → **see** `30-risiko-komitmen.md` — `jenis_dokumen` and `klasifikasi_id` are different things; never mix them in one query
- Concept is a food segment (roti, kopi, garam) → **see** `10-segmen-produk.md`
- A `klasifikasi_id` code returning zero rows is not "does not exist" — some classes are simply not filled yet; say so instead of widening into a neighbouring class.
