# seeknal-bpom-rpo: v6 Context & Skill — Process Compliance + Engine Pakem

**Document type:** Implementation note
**Date:** 2026-09-24
**Status:** Applied to variant folders `docs/context_recap/september/v6-sep-w1` (both repos). **Not yet retested** — retest deliberately deferred; expectations below are the retest contract.
**Scope (changed files, identical edits in both repos unless noted):**
`v6-sep-w1/SEEKNAL_ASK.md` · `v6-sep-w1/context/00-menghitung.md` · `context/10-segmen-produk.md` · `context/11-kode-segmen.md` · `context/15-permohonan.md` · `context/70-btp.md` · `context/35-klasifikasi-sifat.md` (ERBA only) · `v6-sep-w1/skills/bpom-forecaster/SKILL.md` · `v6-sep-w1/skills/detect-anomaly/SKILL.md` · `v6-sep-w1/README.md`
**Baseline:** `v5-sep-w1` (full copy) + audit findings in `docs/audit_context/2026-09-23-v4-sep-w1-domain-split/09-RERUN-DAN-V5.md` and `10-DB-DAN-KLASIFIKASI-ULANG-240926.md`

---

## 1. Purpose

v5 fixed truth-of-numbers (stale records, tolerance, cross-schema fixtures). v6 targets **process compliance**: what the agent must do around the numbers (one population per figure, cross-system namespace honesty, CSV availability branches, engine-only forecast/anomaly) — without ever citing absolute figures in context text (style rule since v4, re-verified: zero real numbers in the new context passages).

## 2. Why This Was Needed (evidence)

- **Cross-table sum** — `UAT-E02` (ERLA): agent answered 422.085 by adding produk (412.591) + BTP. No context rule said a "total" must never add the two populations.
- **Wrong-system namespace** — `UAT-E13` (ERLA): codes `0809/0905` are ERBA's `jenis_pangan` namespace; in ERLA they return 0. The agent's name-based fallback (`nama_kategori ILIKE '%Sosis Daging%'` = 2.453) was the right method, but no context page taught it or sanctioned it.
- **CSV contradiction** — variant configs set `upload_to_s3: enabled: false`, while SEEKNAL_ASK mandated CSV export unconditionally in three places (skill table, Gate 0 closing pair, Gate 5 CSV bullet). Compliance measured at 32 % (batch runs) and 0 % (worker sessions where the tool was not even registered). The agent improvised the behaviour ("tool tidak terdaftar") with no taught rule.
- **Forecast/anomaly engine violations** — `UAT-E46` (both repos): forecast-per-category answered **without calling `run_forecast`** (ERBA also ran `execute_python`); the fixture rewarded it. `UAT-E30`: `detect_anomaly` ran but `execute_python` ran alongside — statistics possibly computed outside the engine.
- **Unlabelled bulk bucket** — `klasifikasi_id = '3'` ("Deputi 3 (Pangan)", dictionary-labelled) holds the single largest classification cluster in ERBA; `35-klasifikasi-sifat.md` never mentioned it, inviting mis-folding into 301/302.

## 3. Changes and Expectations

### 3.1 One population per number — `00-menghitung.md` (+ pointers in `15-permohonan.md`, `70-btp.md`)

**Change:** new paragraph after the v5 §3 exception: `t_produk_*` and `t_btp_*` are different populations; a "total" never adds them — report each side labelled, or ask. `15`/`70` gained one routing line each pointing back to `00-menghitung.md`.

**Expectation:** a "berapa total permohonan/BTP" question either returns the asked table's figure alone, or two labelled figures — never one summed number. On retest, `UAT-E02`-style answers must not contain a produk+BTP sum as the headline.

### 3.2 Cross-system namespace — `10-segmen-produk.md`, `11-kode-segmen.md`

**Change:** new block: `jenis_pangan` codes are per-system; a foreign panel code returning zero is a **wrong-system zero**, not a missing segment; state the mismatch; when the user names the product, resolve via `nama_kategori ILIKE` and acknowledge the name-based route.

**Expectation:** `UAT-E13` (ERLA) answers either explain the namespace zero or take the name-based route **and say so**. Silent `IN ('0809','0905') → 0` presented as "segmen tidak ada" is a fail.

### 3.3 CSV two-branch rule — `SEEKNAL_ASK.md` (Gate 0 + Gate 5 CSV bullet)

**Change:** closing-pair and CSV bullets now carry an explicit v6 branch: if `upload_to_s3` is **not registered** in the session → the chart alone closes the answer; state ONCE that the CSV export is unavailable; never retry, never invent a file. When the tool **is** registered, the old obligation stands unchanged (this is the trigger-on-enable path — enabling the tool in config re-activates the mandatory closing pair with zero context edits).

**Expectation:** worker-mode sessions (tool absent) stop silently skipping CSV and start stating the limitation once; batch sessions with the tool registered keep/return to closing-pair compliance. Measured CSV compliance becomes meaningful only relative to tool availability (harness will snapshot availability — pending, see §5).

### 3.4 Engine pakem for forecast/anomaly — `SEEKNAL_ASK.md` Gate 3 + both trigger skills

**Change:** Gate 3 forecast paragraph and both `SKILL.md` RUN sections now state: `execute_python` is **never** a forecast or anomaly instrument — projections and flagged-period statistics come from `run_forecast` / `detect_anomaly` output only; engine refusal is presented as-is (no replacement estimate).

**Expectation:** every forecast/anomaly answer's tool trace contains `run_forecast` / `detect_anomaly` and contains **no** `execute_python`. On retest, `UAT-E46` must not pass without the engine; `UAT-E30` must not mix a python computation into the pipeline. (Fixture-level `assert_tools`/`assert_tools_forbid` to enforce this mechanically are part of the pending v6 harness work, §5.)

### 3.5 ERBA classification bucket `3` — `35-klasifikasi-sifat.md` (ERBA only)

**Change:** one bullet: the bulk `3` bucket is the dictionary-labelled class "Deputi 3 (Pangan)" — a sweep-level bucket, not a missing label; never fold it into 301/302 when tiered classes are asked. Minor classes 306 (Herbal), 308 (Rekayasa Genetika), 309 (Organik) exist the same way. (No counts cited, per style.)

**Expectation:** `UAT-KLASIFIKASI-1`-style answers stop drifting into the Deputi-3 bucket when tiered classes are requested.

### 3.6 README v6 — both variant folders

Five-point change summary in Indonesian, matching the v5 README format, so the variant is self-describing at checkout.

## 4. What Stayed the Same

- Everything from v5 (Case B commitment carve-out, diubah = 9999, MD/ML as origin markers, ERLA MT/MR clarify-first, Gate 5 anti-hallucination + consistency items, engine-only forecast contract from the July era).
- Domain separation: all new text is per-domain-neutral where possible and both variants received identical edits except §3.5 (ERBA-only). No "gabungan/UNION" wording reintroduced anywhere.
- Style: method-teaching prose, zero absolute figures in context passages (machine-checked: no real counts appear in the new text).

## 5. Pending v6 Work (documented, not yet built)

1. **Harness:** `assert_tools` / `assert_tools_forbid` / `assert_skill` / `assert_page_read` (extends the v5 `assert_sql` pattern); `--runs N` majority verdict; tool-availability snapshot recorded per run; two-score report (correctness + Gate compliance; CSV counted only when the tool is registered).
2. **Fixtures:** `assert_tools` + `assert_tools_forbid: [execute_python]` on `E45/E46/E47/E30`; redesign `E46` (engine Total series + labelled category momentum, or clarification); `assert_sql_forbid: [tanggal_aju]` on the rolling-window family; `MT-JP-MEI26-1` gets the MT-1/MT-2 treatment.
3. **Retest:** targeted rerun of the touched fixtures against `v6-sep-w1`, then full 393 sweeps for the v5 baseline and the v6 measurement (documents `11`/`12`).
