# Risk & Commitment

kategori risiko, menengah rendah/tinggi, MR, MT, komitmen, pemenuhan, dibatalkan, disetujui.

**Commitment is not applicable in this repo.** `t_produk_3_rilis_erla` has no `status_komitmen` and no `jenis_penolakan_komitmen` column, so every commitment question ("komitmen dibatalkan", "pemenuhan komitmen", "disetujui dengan catatan") is **out of scope here** — state that limit rather than answering. The rest of this page is about **risk**, which the ERLA table does carry.

## Risk — one column, three levels

Risk lives in `jenis_dokumen`. The dictionary catalogs `JENIS_DOKUMEN`:

| Code | Meaning |
|---|---|
| `000` | Belum Dikategorikan |
| `301` | Pangan Low Risk |
| `302` | Pangan High Risk |
| `303` | Pangan Medium Risk |

⚠️ **ERLA uses `jenis_dokumen`, not `kategori_dokumen`.** Read the risk category from `jenis_dokumen`; this repo carries no second risk column.

**The ERLA data also carries `304`, which no dictionary row describes** — report it as an undocumented class, do not borrow a label from anywhere.

**ERLA has only three levels — Low, Medium, High.** There is **no "Tinggi Notifikasi" in ERLA**; that class belongs to a schema this repo does not use.

### Narrative rules

- Questions touching the risk FAMILY report each class as its own labelled figure. Do not widen one requested class into its neighbour — "produk Medium" means Medium (`303`) only.
- **"Risiko Tinggi" in this repo = `jenis_dokumen = '302'` (Pangan High Risk)**, and nothing else. Do not fold `301` or `304` into it — that merge belongs to a schema this repo does not use.
- MR/MT are official BPOM shorthand for Menengah Rendah / Menengah Tinggi. Write them out in full at least once. In ERLA the three levels are Low (`301`), Medium (`303`), High (`302`); "Medium Risk" alone loses the Rendah/Tinggi distinction the shorthand carries.
- An unmapped class (`304`) is reported alongside the resolved share, never merged into a neighbour.

### Background — the old four-class processing deadlines (not a filter here)

In the earlier four-class risk scheme the classes also carried a processing deadline: **30 days** = Tinggi, **5 days** = Menengah Tinggi, **1 day** = Menengah Rendah, and **15 days** for a fourth class, "Tinggi Notifikasi" (still counted inside the high group). That last class **has no counterpart among the risk codes this repo uses**, so it cannot be separated here. Treat this as background explanation only — it is **not** a filter you can apply; the codes present in the data here are the ones in the table above.

## Routing

- **"Berdasarkan Kontrak" / makloon is NOT this page** — that is the `status_produk` column (`304`), a production-method filter. → **see** `60-asal-produksi.md` (panel binding).
- Process stages also mentioned → **see** `20-status-pipeline.md`
- jenis permohonan also mentioned → **see** `15-permohonan.md`
- "belum ditetapkan kategori risikonya" → **see** `90-kualitas-data.md` (that is `jenis_dokumen='000'`, and the status filter is dropped there).
- Period mentioned → **see** `80-waktu-periode.md`
