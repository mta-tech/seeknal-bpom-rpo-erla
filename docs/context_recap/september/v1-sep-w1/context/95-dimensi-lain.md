# Other Dimensions

Columns no page covers — how to discover them — plus peruntukan & pengolahan.

The other pages name the traps, not every column. This database has **152 columns**; a dimension appearing on no page does **not** mean it is absent from the data — discover it before falling back to a familiar column.

## Dimension discovery procedure

**1. Does the column exist?** `describe_table`.
`t_produk_3_rilis_erla` is a **pure subset** of `t_produk_3_erba`: 94 shared columns, **0** ERLA-only columns, and only **6** ERBA-only columns —
`ecolabel` · `jenis_penolakan_komitmen` · `kode_kbli` · `sni_sukarela` · `status_komitmen` ·
`sub_kemasan_id`. A column existing in only one system makes the question **structurally single-system**; name that system, do not imply both sides contribute.

**2. Coded or free text?** A bare code means nothing. Resolve it in `data_dictionary` via **`kategori` AND `sumber`**. Find the category with `WHERE deskripsi ILIKE '%<term>%'`, then read that entire category.

**3. No matching category?** The code is **undocumented**. Report its distribution, state that its meaning is not recorded, offer verification with the data owner — and **never borrow a label from another category**. Refusing to answer at all is equally wrong: the share that did resolve is valid.

**4. Free text?** ILIKE to **discover** the exact value, then count with `=`
(`10-segmen-produk.md`).

## The same code, different meanings — always check the category

`301` and `302` each appear in **9 different categories** (`JENIS_DOKUMEN`, `JENIS_PERMOHONAN`,
`JENIS_PRODUK_BTP`, `KATEGORI_DOKUMEN`, `KLASIFIKASI_ID`, `PEMROSESAN`, `STATUS`, `STATUS_PRODUK`,
`SUB_KEMASAN_ID`); `303`/`304` in 7. **A column is chosen because of its meaning — never because its numbers happen to match.**

One label can cover several codes: ERBA `STATUS` "Pendaftar - Perlu Data Tambahan" attaches to **5 codes**, "Pendaftar - Draft" to 3. Take **all** codes sharing that description.

## `pengolahan` ≠ `pemrosesan` — two columns, one Indonesian word

| Column | Codes | Dictionary | Present in |
|---|---|---|---|
| `pemrosesan` | `300` Tanpa Proses Tertentu · `301` Organik · `302` Rekayasa Genetik (GMO) · `303` — · `304` Pangan Very Low Risk **and** Iradiasi (two descriptions, an internal collision) | category `PEMROSESAN`, `sumber` "ERLA dan ERBA" | both systems |
| `pengolahan` | `401`–`408` | **no category at all** | **all four tables**, but the used code range and fill rate differ per table — check each side (`00-menghitung.md` §5) |

Both mean "pengolahan" in Indonesian. **Name the column you used**, and if the question is ambiguous, say that two similarly-named columns exist with different contents.

The `401`–`408` codes of `pengolahan` **collide** with `SUB_KEMASAN_ID` (401 = Plastik/Aluminium Foil) and with ERLA `STATUS` (401 = Kepala Seksi - Proses Verifikasi Ditolak) — both are **wrong** for this column. Its meaning cannot be determined from the data; say so.

## Peruntukan

`peruntukan`: `0201` **khusus** · `0000` **umum** — opposite concepts, so mixing them up here is not a small slip but the inverse answer. Both systems also carry undocumented codes (`0103`/`0104`/`0105`/`0106`, plus `010101` in ERLA) that are neither.
`SELECT peruntukan, COUNT(*) … GROUP BY 1` shows the composition before choosing.
Details of the two "khusus" readings → `35-klasifikasi-sifat.md`.

## Catalog gaps that produce wrong numbers without errors

- **The ingredient table does not exist here.** The legacy system carried a
  `T_PRODUK_3_BAHAN` table (`NAMA_BAHAN`, `JENIS_BAHAN`); this warehouse has no equivalent —
  in any schema. "Produk yang mengandung bahan baku X" is therefore NOT COVERED: offer the
  honest substitute (a `nama`/`merk` search, labelled as a product-name search, not an
  ingredient analysis) and say what is missing. Full mapping: `05-filter-katalog.md`.
- **The same `sumber` does not guarantee the same code range** (`jenis_btp` → `70-btp.md`).
- **ERLA codes are stored without zero-padding** in the dictionary (`999`, `99`, `9`) while the data stores 4 characters (`0999`) — `LPAD(kode,4,'0')` before filtering, or the query returns zero. Some ERBA `status` values (`0500`, `0504`, `0417`, `0900`, `0909`, `0916`) are actually registered in the ERLA block — "not in the ERBA block" means "check the ERLA block", not "unknown code".
- **Codes in the data but not in the dictionary** (the sets drift — recheck):
  `jenis_dokumen` 304 · `peruntukan` 0103–0106 · `pemrosesan` 303/403 · `bentuk_sediaan` 214 ·
  `status_produk` 303/305 (ERLA). If a filter would drop such rows (`NOT IN`, "lainnya"), say so — never present a total as complete.
- **Registered codes with zero rows** — keep them in the filter, do not present them as contributors.

## Routing

- Dimension found and it belongs to another topic → **back** to the map in `SEEKNAL_ASK.md`.
- The dimension is free text → **see** `12-nama-kategori.md`.
- Two candidate columns still both plausible → **back** to Gate 1, ask. Do not choose silently.
