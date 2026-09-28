# Process Stages

draft, bayar, verifikasi, evaluator, direktur, ditolak, dicabut, antrian, nyangkut, bottleneck.

A stage's population is defined by its own status codes. Layering a valid-NIE set on top erases the population being asked about (`00-menghitung.md` §2).

| Stage | ERBA codes |
|---|---|
| Evaluator | `0301, 0308` |
| Verifikator 1 | `0402, 0403, 0405, 0406, 0407, 0417` |
| Verifikator 2 | `0500, 0502, 0504` (exactly these; `0501, 0503` never occur in ERBA) |
| Direktur | `0600, 0601, 0666` |
| Deputi / Kepala Badan | `0700` / `0800` |
| Draft | `0910, 0912` |
| Bayar (awaiting SPB/HPR) | `0903, 0907` |
| Data Tambahan | `0308, 0402, 0407` (petugas) **+** `0901, 0914, 0915, 0917, 0951` (pendaftar) — **always both groups** |
| Ditolak Sistem | `0908, 0911, 0918` |
| Ditolak lainnya (penerimaan/verifikasi) | `0902, 0905, 0913` |
| Terbit / Perubahan / Sudah Diubah | `0999` / `0906` / `9999` |
| Dibatalkan / Dicabut / Tidak Berlaku | `0000, 0009, 0099` |

**This list wins over dictionary lookups.** Dictionary descriptions repeat across codes — "Pendaftar - Perlu Data Tambahan" attaches to **5 codes at once**, "Pendaftar - Draft" to 3. Picking a single code from a repeated label silently loses the rest.

## User terms → buckets: what is obvious, and what must be asked

The stage names in the table are not the words people use. Map first; never guess from word similarity:

| User term | Bucket |
|---|---|
| "di direktur", "menunggu tanda tangan" | Direktur |
| "draft", "belum disubmit" | Draft |
| "menunggu bayar", "belum bayar" | Bayar |
| "minta data tambahan", "dikembalikan ke pendaftar" | Data Tambahan (**both groups**) |
| "izin edar **diubah** / **diperbarui** / **direvisi** / diganti versi baru" | `status='9999'` (Sudah Diubah — a VERSION, see below). Not a revocation, not a permohonan |
| **"menunggu verifikasi", "di verifikasi", "tahap verifikasi"** | **AMBIGUOUS — Gate 1, ask** |

**"Verifikasi" is two consecutive stages, not one.** Verifikator 1 and Verifikator 2 are separate queues staffed by different people, and their sizes differ a lot. Summing both and summing either one are two different answers — neither can claim to be the default. Ask which one is meant, or present **all three labelled** (Verifikator 1 · Verifikator 2 · combined) when clarification is impossible.

The general rule: when one user term maps to more than one bucket in the table above, that is Gate 1 — not a reason to pick the largest or the first.

## Which table

- **Product pipeline: `t_produk_3_erba` only** — `t_produk_3_rilis_erla` stores final states only.
- **BTP pipeline: BOTH** `t_btp_3_erba` AND `t_btp_3_erla` — the BTP ERLA table genuinely carries live states. Its code set is smaller: `t_btp_3_erba` lacks `0009`; `t_btp_3_erla` lacks `0099` and adds `0299`; of the Verifikator 2 trio only `0502` appears. Do not copy the stage list between tables unchecked.
- "Permohonan/produk" in a pipeline question is ambiguous about BTP → present **two labelled figures** (products-only and products+BTP, each with its source table), or state the scope used and why.

## Shape rules

- **Keep every stage code in the filter, including empty ones** (`0402`, `0406`, `0601`, `0666`, `0700`, `0905` currently have zero rows). Filtering them costs nothing and survives future fills — but the **answer** must not present empty codes as contributing stages; name the codes that actually carry rows.
- **`NOT IN` buckets absorb rows no stage claims**: rare codes `000X, 0417, 0900, 0909, 0916`, and the largest one — rows whose `status` is **four spaces** (`TRIM(status)=''`, which `status <> ''` misses). Mention this absorption when presenting a `NOT IN` total.
- "Currently at stage X" is a snapshot — include its as-of date.
- **"Sedang diproses (petugas)"** = Evaluator + Verifikator 1/2 + Direktur/Deputi/Kepala Badan + Data Tambahan. **"Belum selesai (total)"** = everything `NOT IN` a terminal state (Terbit/Perubahan/Sudah Diubah + Dibatalkan/Dicabut/Tidak Berlaku) — the wider reading, including the registrant-side Draft/Bayar queues. Lead with the first and **always** attach the second as ONE labelled figure from its own `NOT IN` query. They are not variants of each other: the second covers the whole registrant-side queue the first omits.
- **"Nyangkut / bottleneck / paling menumpuk"** = rank the STAGES with a single `GROUP BY` over buckets, then name the largest — never a hand-picked code.

## Dicabut is not kedaluwarsa — three codes that keep getting confused

| Code | Meaning | Class |
|---|---|---|
| `0000` | Dihapus | **TERMINATION** — an authority action |
| `0009` | Dicabut / Dibatalkan | **TERMINATION** — an authority action |
| `0099` | Tidak Berlaku | **VALIDITY** — expires on its own, not revoked |
| `9999` | Sudah Diubah | **VERSION** — superseded by a newer revision, not revoked |

"Dicabut / dibatalkan / diterminasi" = `0000` + `0009` **only**. The other codes answer different questions: `0099` answers "expired", `9999` answers "revised".

**Before sending a revocation query, re-read its code set.** If `0099` is in there and the question does not mention kedaluwarsa, remove it. This is the best-known and least-applied rule — knowing it is not enough; the code set must actually be checked in the SQL text before it executes.

## Products vs products+BTP scope — state it before counting

The table rules are in "Which table" above, but the failures do not come from ignorance — they come from the scope never being consciously decided. Before any pipeline query: write down the scope (products-only / products+BTP), then check the `FROM` tables match what you wrote. When the question does not say, products-only is the default — and that default **must be stated in the answer**, not left silent.

## Routing

- Need to map codes to labels / find an unlisted stage → **continue to** `21-kode-status.md`
- Risk or komitmen mentioned → **see** `30-risiko-komitmen.md`
- Period/trend mentioned → **see** `80-waktu-periode.md`
- Test accounts pile up in Draft — a mandatory exclusion, and here it can flip the conclusion (`00-menghitung.md` §3).
