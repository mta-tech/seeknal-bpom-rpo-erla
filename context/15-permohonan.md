# Permohonan & jenis_permohonan

baru, perubahan mayor/minor, daftar ulang, notifikasi, disetujui, diterima.

`jenis_permohonan` is not universal — its presence is decided by the wording of the question, not by habit.

**`sumber` scope.** In `data_dictionary`, `jenis_permohonan`'s `sumber` lists both ERBA and ERLA; this repo queries the ERLA table only. The same code means the same thing wherever it appears, so do not build separate code sets.

| Code | Meaning | Group |
|---|---|---|
| `301` | Permohonan Baru | **baru** |
| `302` | Perubahan Mayor | perubahan |
| `303` | Perubahan Minor | perubahan |
| `304` | Daftar Ulang | periodic renewal — stands alone |
| `305` | Permohonan Baru Notifikasi | notification track — stands alone |

**"layanan" = `jenis_permohonan`.** Apply a service filter only when the question names the service; the table below maps the trigger words to the filter.

## Branch — pick it from the words in the question

| Intent | Trigger words | Filter |
|---|---|---|
| New registration | "baru", "permohonan baru" | **`301` + `305`** |
| New — notification only | "baru notifikasi" | `305` |
| Modification | "modifikasi", "perubahan", "variasi", **"p5"** | `302` + `303` |
| Major only | "mayor" | `302` |
| Minor only | "minor" | `303` |
| Renewal | "daftar ulang" | `304` |
| Generic | "berapa permohonan/pengajuan" | **no `jenis_permohonan` filter** |

**Persetujuan uses the SAME filter but a different entity.** A "persetujuan" question still selects applications by `jenis_permohonan`, but counts the issued product — `COUNT(DISTINCT nomor)` rather than `produk_id` (`00-menghitung.md` §1):

- "berapa persetujuan modifikasi" → `COUNT(DISTINCT nomor)` + `jenis_permohonan IN ('302','303')`;
- "persetujuan baru" → `COUNT(DISTINCT nomor)` + `jenis_permohonan IN ('301','305')`;
- "variasi dan baru" → `jenis_permohonan IN ('301','305','302','303')`;
- "mayor" alone → `jenis_permohonan = '302'`.

`304` Daftar Ulang never counts as "baru" — it renews an existing izin. `305` is the notification track, merged into "baru" only per the table above.

**`nomor_surat` is 1:1 with `produk_id`.** Unlike ERBA — where the two columns hold the same value — ERLA stores the **real decision letter** (a `PN.…` value), yet the cardinality is identical: distinct `produk_id` equals distinct `nomor_surat`, which equals the row count. So "berapa pengajuan / permohonan / surat keputusan" returns the **same figure** counted from either column. A question using the word `nomor` here means the application/surat number, never the NIE — the NIE lives in `nomor` too, so do not swap the two.

**"Terbit" is not a `jenis_permohonan` trigger.** "NIE yang terbit di {period}" counts every application type in that period; only the explicit word "baru" narrows. The reason: products whose active NIE arrived via a Perubahan still hold an active NIE — filtering to "baru" unasked drops them. **But when BOTH words appear — "permohonan baru yang terbit {periode}" — each word does its own job**: "baru" applies the `jenis_permohonan IN ('301','305')` filter, and "terbit" selects the date column (`tanggal`, issuance) with entity `nomor` + valid status. Do not let the pengajuan pairing (`produk_id`+`tanggal_aju`) swallow the "terbit" wording.

**The two volume readings are not interchangeable.** Raw-submission counts include every application regardless of outcome; approved counts only those that ended in issuance — different populations, not a slight shift. The default is the approved reading — **product level (`nomor`)**; "disetujui", "diterima", "izin edar" point there. Drop the status filter only when the question is genuinely about submission volume regardless of outcome.

**Perubahan/revisi** = `302` (mayor) + `303` (minor). When the question compares "baru vs mengubah", present `301`+`305` on one side and `302`+`303` on the other, each labelled, and mention `304` as the group that falls in neither — leaving it out unmentioned makes both sides look like they sum the whole population when they do not.

**Entity and date come from ONE row of the decision table (`00-menghitung.md` §1) — not from the word "permohonan" alone:**

| Wording | Entity · date · status |
|---|---|
| "diajukan" · "Tanggal Permohonan" · "pengajuan masuk" · "berapa yang masuk/mengajukan" · volume trend without the word izin | `produk_id` · `tanggal_aju` · **no status filter** |
| **"disetujui" · "persetujuan"** (panel reading) | **`nomor` · `tanggal` · valid tier** |
| "surat keputusan" / "berapa kali keputusan" | `produk_id` · `tanggal_bayar` · valid tier |
| "izin edar" · "terbit" · "produk terdaftar" | `nomor` · `tanggal` · valid tier |

"Disetujui/persetujuan" is read at **product level** (`nomor`): a product revised and re-approved in
the same period counts once, not once per revision (the `nomor` reading runs slightly below the
`produk_id` reading; the gap is same-month re-approvals). "Surat keputusan" is the opposite: the
question asks how many decisions were made, so count `produk_id`. If a question carries both
wordings, that is a Gate 1 clarification.

**"Diperbarui dengan nomor baru" / "sudah diubah"** = `status='9999'` — the NIE was REPLACED by a
newer version, not renewed. It is never `jenis_permohonan='304'` (Daftar Ulang is scheduled
renewal); offering "Daftar Ulang" as the clarification answer to this wording is a misread.

## The versioning story — one NIE per revision, new registrations add NIEs

Revisions never change the NIE — but a **new registration (`301`/`305`) can issue a second NIE
for the same product**, so "the product" is not always one NIE (`00-menghitung.md` §1 has the
tie-break). Every revision, perubahan, or daftar ulang creates a **new nomor pengajuan**
(`produk_id`), and each carries its own context in `jenis_permohonan`. This is why the entity
choice decides the answer, not a nuance:

- **Many NIEs have more than one application.** Example: an NIE carries several applications —
  each with its own `jenis_permohonan` (`301` Baru, then `302` Mayor revisions) — one NIE with
  many `produk_id`, each prefixed `EREG…`.
- **"Berapa surat keputusan / berapa kali pengajuan" → `COUNT(DISTINCT produk_id)`.** When the
  wording says keputusan, add the valid status set — an application that never
  issued produced no decision. **"Berapa produk terregistrasi" → `COUNT(DISTINCT nomor)`.**
  **"Persetujuan / disetujui" → `COUNT(DISTINCT nomor)` + `tanggal`** (product level —
  a same-period re-approval of a revised application counts once; see the decision table above).
  These readings differ by exactly the revision multiplicity above; swapping them silently
  multiplies or divides the answer.
- **Unissued applications never masquerade as NIEs**: their `nomor` equals `produk_id`
  (a value prefixed `EREG…`) or is empty. Present them as application numbers with their
  `jenis_permohonan` context, never as issued NIEs (`data_architecture.md`).
- The **history of a product** = the ordered set of its applications: new → mayor/minor
  revisions → daftar ulang. Questions like "konteksnya permohonannya apa" read each
  application's own `jenis_permohonan`, one row per `produk_id`.

## Two exceptions that take neither branch

- **Pipeline-stage questions** ("berapa nyangkut di Draft", "menunggu verifikasi"): their population is defined by the status codes themselves; layering a valid-NIE set on top erases it. → `20-status-pipeline.md`
- **Komitmen Case-B questions** ("berapa komitmen dibatalkan"): the population lives in `status_komitmen` and mostly never reaches an NIE. → `30-risiko-komitmen.md`

## When the question asks for a LIST / product detail (not just a count)

"produk apa saja", "sebutkan", "daftar", "contoh produk", "NIE-nya apa saja" → the answer shows product rows. For such rows, include the **Nomor Pengajuan (`produk_id`)** alongside the **NIE (`nomor`)**, merk, and name — the application number is the file's identity, the NIE is its izin edar, and together they tell the full story.

- If `nomor` = `produk_id` (a value prefixed `EREG…`), that product **has not been issued an NIE yet** — present it as an application number, never call it an NIE (`data_architecture.md`).
- One NIE can carry several application numbers (daftar ulang, perubahan, multi-pabrik). When counting **how many products**, use `COUNT(DISTINCT nomor)`; when counting **how many applications / registration activities**, use `COUNT(DISTINCT produk_id)` (`00-menghitung.md` §1).
- Identifiers may only come from `execute_sql` this turn — never invent numbers.

Pure count/trend questions ("berapa", "per tahun") need none of this list — answer concisely.

## Routing

- Pipeline stages / queues mentioned → **see** `20-status-pipeline.md`
- Komitmen / pemenuhan / dibatalkan mentioned → **see** `30-risiko-komitmen.md`
- Period, trend, per year/month mentioned → **see** `80-waktu-periode.md`
- entity · date · status come from ONE row of the decision table → `00-menghitung.md` §1
- "total permohonan" never adds `t_produk_*` with `t_btp_*` — one population per number (v6) → `00-menghitung.md`
