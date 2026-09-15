---
name: verdict
description: >
  Verdict-only SOC pipeline. Runs Phases 1–4: triage, FP validation,
  investigation, and adjudication. Produces a CONFIRMED TP / CONFIRMED FP /
  UNCERTAIN verdict with a full audit trail. No remediation playbook or
  incident report. Use when you want a verdict without the full output.
  Triggers on "is this a FP", "potential FP", "FP/TP check", "evaluate this
  detection", "verdict only", or similar.
user-invocable: true
---

# verdict

Verdict-only pipeline (VERDICT_ONLY scope). Runs Phases 1–4.

All Spectra calls are made through the canonical `rl-soc-cli` command using
the `Bash` tool.

## CRITICAL CONSTRAINTS

- **Do NOT perform any manual analysis of files.**
- **Do NOT create or repair the `rl-soc-cli` symlink, credentials, `.env`/config
  files, or the venv.** The single exception: if both wrappers are installed,
  Step 1 may repoint the symlink at the analyst's explicit choice.
- **Use `rl-soc-cli` via `Bash` only for the pre-lookup in Step 2.**
  All other Spectra calls are made by sub-agents. Pattern:
  `rl-soc-cli <tool_name> --args '<json>'` — exit codes: 0 success / 1 error / 2 bad args.
- **Use Spectra tools exclusively.** Do NOT query external platforms.
- **Do NOT use any `rl-protect` skills or tools.**
- **Pass `SPECTRA_SERVICE` and `AVAILABLE_TOOLS` to every sub-agent you spawn.**

---

## Step 1 — Resolve the active endpoint and tool surface

### 1. Confirm the canonical CLI exists

```bash
command -v rl-soc-cli
```

- **Not found** → stop. Tell the analyst to run `rl-soc-install` then `rl-soc-connect`.

### 2. Resolve which service it points at

```bash
BIN="$(dirname "$(command -v rl-soc-cli)")"; for w in rl-spectra-intel rl-spectra-analyze; do [ -x "$BIN/$w" ] && echo "installed: $w"; done; echo "current: $(readlink "$(command -v rl-soc-cli)")"
```

Map the symlink target to `SPECTRA_SERVICE`:

| Target basename | `SPECTRA_SERVICE` |
|---|---|
| `rl-spectra-intel` | `Spectra Intelligence` |
| `rl-spectra-analyze` | `Spectra Analyze` |
| anything else / not a symlink | `unknown` (continue) |

**Only one wrapper installed** → tell the analyst which endpoint this run uses and continue.

**Both wrappers installed** → ask which to use, defaulting to the current symlink target.
If they pick the other one, repoint:
```bash
ln -sfn "$BIN/rl-spectra-intel" "$BIN/rl-soc-cli"     # Spectra Intelligence
ln -sfn "$BIN/rl-spectra-analyze" "$BIN/rl-soc-cli"   # Spectra Analyze
```

### 3. Discover the tool surface

```bash
rl-soc-cli --list-tools
```

Collect tool-name tokens into `AVAILABLE_TOOLS`.

---

## Step 2 — Generate investigation ID and artifact slug

```bash
date +%Y%m%d-%H%M%S
```

Store as `INVESTIGATION_ID`. Set `PIPELINE_SCOPE = VERDICT_ONLY`.

Derive `ARTIFACT_SLUG`:

| Artifact type | ARTIFACT_SLUG |
|---|---|
| SHA256 hash | First 12 characters of the hash (e.g. `a3f8e91b2c04`) |
| SHA1 or MD5 hash | First 12 characters of the hash |
| File upload | `upload` for now; replace with first 12 chars of SHA256 after Phase 1 returns |
| URL | Hostname with `.` replaced by `-`, truncated to 20 chars (e.g. `evil-example-com`) |
| IP address | Dots replaced with hyphens (e.g. `192-168-1-1`) |
| Domain | Dots replaced with hyphens, truncated to 20 chars (e.g. `malware-example-com`) |

For hash and network artifacts, create the directory now:

```bash
mkdir -p <ARTIFACT_SLUG>
```

For file uploads, create the directory after Phase 1 returns the SHA256.

Files follow the pattern `<ARTIFACT_SLUG>/<name>-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

### Pre-lookup for hash artifacts

For **hash** artifacts (SHA256, SHA1, MD5) only:

```bash
rl-soc-cli get_sample_overview --args '{"hash_value": "<hash>"}'
```

Store as `SAMPLE_OVERVIEW`. Extract `SAMPLE_FILE_TYPE` and `SAMPLE_SIZE`.
If the call fails, store both as `unknown` — non-fatal.

Skip for **file uploads** and **network indicators**.

Tell the analyst:
> "Starting investigation `<INVESTIGATION_ID>` for `<artifact>` [scope: VERDICT_ONLY]."

---

## Phase 1 — Triage + Content Inspection
*(Always runs)*

### Hash artifacts

Tell the analyst: "Phase 1 — launching rl-triage-classify, rl-triage-sandbox, and rl-triage-inspect in parallel."

Spawn ALL THREE simultaneously in a single Task turn:

**rl-triage-classify** — artifact identification, classification, indicators, TTPs,
detection rules. Pass the full `SAMPLE_OVERVIEW` JSON so the agent skips its own
`get_sample_overview` call.

**rl-triage-sandbox** — dynamic/auxiliary analysis status check and kickoff.

**rl-triage-inspect** — pass SHA256, `SAMPLE_FILE_TYPE`, and `SAMPLE_SIZE`. The
agent skips internally if `SAMPLE_FILE_TYPE` is not `Text/*` or size exceeds 722KB.

### File upload artifacts

Tell the analyst: "Phase 1 — launching rl-triage-classify and rl-triage-sandbox in parallel."

Spawn **rl-triage-classify** and **rl-triage-sandbox** simultaneously.
After rl-triage-classify returns the SHA256, file type, and size:
- If file type starts with `Text/` AND size ≤ 739,328 bytes → spawn **rl-triage-inspect**.
- Otherwise → skip rl-triage-inspect.

Wait for rl-triage-inspect to complete. Finalize `ARTIFACT_SLUG` from the SHA256
and create the directory:

```bash
mkdir -p <ARTIFACT_SLUG>
```

### Network indicators (URL, IP, domain)

Tell the analyst: "Phase 1 — launching rl-triage-classify and rl-triage-sandbox in parallel."

Spawn **rl-triage-classify** and **rl-triage-sandbox** only — no file content to inspect.

---

After all applicable sub-agents return, synthesize their outputs.

Write: `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
If inspect ran: `<ARTIFACT_SLUG>/rl-triage-inspect-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

**Route on rl-triage-inspect verdict (if it ran):**

- **CONCLUSIVE BENIGN** → Skip Phases 2, 3, and 4.
  Tell the analyst:
  > "Triage complete. Content inspection: CONCLUSIVE BENIGN — [key signals].
  > Skipping FP validation, investigation, and adjudication."
  Write verdict summary and go to Final summary.

- **CONCLUSIVE MALICIOUS** → Skip Phases 2, 3, and 4.
  Tell the analyst:
  > "Triage complete. Content inspection: CONCLUSIVE MALICIOUS — [key signals].
  > Skipping FP validation, investigation, and adjudication."
  Write verdict summary and go to Final summary.

- **INCONCLUSIVE / SKIPPED / did not run** → go to Phase 2.
  Tell the analyst:
  > "Triage complete — [verdict]. Proceeding to FP validation."

---

## Phase 2 — FP Validation
*(Skip if rl-triage-inspect returned CONCLUSIVE BENIGN or CONCLUSIVE MALICIOUS)*

Tell the analyst: "Phase 2 — launching rl-fp-validate."

Spawn: **rl-fp-validate** — pass full triage handoff contents. Also pass inspect
findings if they exist — content analysis signals are relevant FP evidence.

Write: `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

**Route on recommendation:**

- **CONCLUSIVE FP** → Skip Phases 3 and 4.
  Tell the analyst:
  > "FP validation: CONCLUSIVE FP — [key signals]. Skipping investigation and adjudication."
  Write verdict summary and go to Final summary.

- **POSSIBLE FP / LIKELY TP / INSUFFICIENT DATA** → go to Phase 3.
  Tell the analyst the FP confidence level and proceed.

---

## Phase 3 — Investigation
*(Skip if Phase 1 returned CONCLUSIVE BENIGN)*
*(Skip if Phase 2 returned CONCLUSIVE FP)*

Tell the analyst: "Phase 3 — launching rl-investigate-enrich, rl-investigate-pivot, and rl-investigate-hunt in parallel."

Spawn ALL THREE simultaneously in a single Task turn:

**rl-investigate-enrich** — bulk IOC enrichment.
**rl-investigate-pivot** — advanced search pivots + certificate analytics.
**rl-investigate-hunt** — sandbox polling and YARA hunt.

After all three return, synthesize and write:
`<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

Tell the analyst: "Investigation complete. Starting adjudication."

---

## Phase 4 — Adjudication
*(Skip if Phase 1 returned CONCLUSIVE BENIGN)*
*(Skip if Phase 2 returned CONCLUSIVE FP)*

Tell the analyst: "Phase 4 — launching rl-adjudicate."

Spawn: **rl-adjudicate** — pass triage handoff + FP findings + investigation findings.

Write: `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

Tell the analyst the verdict and confidence. Write verdict summary and go to Final summary.

---

## Abort if blocked

If any phase fails (API error, agent returns nothing useful, file write fails),
surface the error to the analyst and ask how to proceed. Do not silently skip
a phase with missing data.

---

## Write verdict summary — always

Before the final summary, write:
`<ARTIFACT_SLUG>/rl-verdict-summary-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

This file is written at every pipeline exit point regardless of short-circuit path.
Populate it from whichever phase produced the verdict.

```markdown
# Verdict Summary

## Subject
- <artifact value and type>

## Investigation ID
`<INVESTIGATION_ID>`

## Pipeline Scope
`VERDICT_ONLY`

## Verdict
**<CONCLUSIVE BENIGN | CONCLUSIVE FP | CONFIRMED TP | CONFIRMED FP | UNCERTAIN | NO VERDICT RENDERED>**

## Reached via
<phase that produced the verdict — e.g. "Content inspection (rl-triage-inspect)" /
"FP validation (rl-fp-validate)" / "Adjudication (rl-adjudicate)" /
"N/A — pipeline stopped before verdict">

## Key Signals
- <1–3 bullets summarising the most important signals; sourced from the relevant phase output>

## Phase Outputs Written
- <list only files actually written, with their full filenames>
```

---

## Final summary

After all phases complete, tell the analyst:
- Final verdict and threat name (from adjudication, if ran)
- FP critique assessment (from adjudication, if ran)
- IOC count (if investigation ran)
- Which phases ran and which were skipped
- Investigation ID (`<INVESTIGATION_ID>`)
- Path to verdict summary: `<ARTIFACT_SLUG>/rl-verdict-summary-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- Remind the analyst they can run `/rl-soc:full` or the standalone `rl-report` / `rl-remediate` skills for the full output
- Any UNCERTAIN verdict open questions for human review
- Token usage summary collected from all agent `## Usage Estimate` sections:

  | Agent | Model | Input tokens | Output tokens |
  |---|---|---|---|
  | rl-triage-classify | sonnet/high | N | N |
  | rl-triage-sandbox | haiku/low | N | N |
  | rl-triage-inspect *(if ran)* | opus/high | N | N |
  | rl-fp-validate *(if ran)* | opus/medium | N | N |
  | rl-investigate-enrich *(if ran)* | haiku/high | N | N |
  | rl-investigate-pivot *(if ran)* | opus/high | N | N |
  | rl-investigate-hunt *(if ran)* | sonnet/high | N | N |
  | rl-adjudicate *(if ran)* | opus/medium | N | N |
  | **Total** | | **N** | **N** |
