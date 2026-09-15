---
name: full
description: >
  Full SOC threat-analysis pipeline. Give it a file, hash, URL, IP, or domain
  and it runs all phases in sequence: triage → FP validation → investigation →
  adjudication → remediation (if TP) → reporting. Triggers on "analyze this
  hash", "investigate this URL", "is this malware", "run threat analysis on",
  "I got a suspicious file", or any submission of an indicator for end-to-end
  SOC workflow.
user-invocable: true
---

# full

Full 6-phase SOC pipeline (FULL scope). For a verdict without the full
output, use `/rl-soc:verdict` (phases 1–4).

All Spectra calls are made through the canonical `rl-soc-cli` command using
the `Bash` tool. `rl-soc-cli` is a symlink managed by `rl-soc-connect` that
points at the active endpoint's wrapper.

## CRITICAL CONSTRAINTS

- **Do NOT perform any manual analysis of files.** Never run `xxd`, `strings`,
  `file`, `hexdump`, `cat`, or any tool to inspect file contents. Pass file
  paths to sub-agents as plain strings.
- **Do NOT create or repair the `rl-soc-cli` symlink, credentials, `.env`/config
  files, or the venv.** `rl-soc-connect` owns creation and all configuration.
  The single exception: if **both** endpoint wrappers are installed, Step 1
  may repoint the existing symlink between them at the analyst's explicit choice.
- **Use `rl-soc-cli` via `Bash` only for the pre-lookup in Step 2.**
  All other Spectra calls are made by sub-agents. Pattern:
  `rl-soc-cli <tool_name> --args '<json>'` — exit codes: 0 success / 1 error / 2 bad args.
- **Use Spectra tools exclusively.** Do NOT query external platforms.
- **Do NOT use any `rl-protect` skills or tools.**
- **Pass `SPECTRA_SERVICE` and `AVAILABLE_TOOLS` to every sub-agent you spawn.**
  Do not mix endpoints within a run.

---

## Step 1 — Resolve the active endpoint and tool surface

### 1. Confirm the canonical CLI exists

```bash
command -v rl-soc-cli
```

- **Not found** → stop. Tell the analyst to run the `rl-soc-install` skill and
  then `rl-soc-connect` before running the pipeline.

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

**Both wrappers installed** → ask the analyst which to use, defaulting to the current symlink target:
> "Both endpoints are installed. `rl-soc-cli` currently points at <current>. Which should this run use?"

If they pick the other one, repoint the symlink:
```bash
ln -sfn "$BIN/rl-spectra-intel" "$BIN/rl-soc-cli"     # Spectra Intelligence
ln -sfn "$BIN/rl-spectra-analyze" "$BIN/rl-soc-cli"   # Spectra Analyze
```
Confirm with `readlink` and set `SPECTRA_SERVICE` from the new target.

### 3. Discover the tool surface

```bash
rl-soc-cli --list-tools
```

Collect tool-name tokens into `AVAILABLE_TOOLS`. Stop and surface the error if
the probe exits non-zero or returns nothing parseable.

---

## Step 2 — Generate investigation ID and artifact slug

```bash
date +%Y%m%d-%H%M%S
```

Store as `INVESTIGATION_ID`. Set `PIPELINE_SCOPE = FULL`.

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

For **hash** artifacts (SHA256, SHA1, MD5) only, call `get_sample_overview`
before spawning any sub-agents — to determine file type and size for triage routing:

```bash
rl-soc-cli get_sample_overview --args '{"hash_value": "<hash>"}'
```

Store as `SAMPLE_OVERVIEW`. Extract:
- `SAMPLE_FILE_TYPE` — e.g. `Text/PowerShell`, `PE/Exe`
- `SAMPLE_SIZE` — file size in bytes

If the call fails or sample is unknown, store both as `unknown` — non-fatal;
triage will proceed and rl-triage-inspect will handle the unknown case itself.

Skip for **file uploads** (SHA256 not yet known) and **network indicators**.

Tell the analyst:
> "Starting investigation `<INVESTIGATION_ID>` for `<artifact>` [scope: FULL]."

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

Tell the analyst the inspect verdict (if it ran) and proceed:
> "Triage complete — [verdict]. Proceeding to FP validation."

Always proceed to Phase 2 regardless of inspect verdict.

---

## Phase 2 — FP Validation

Tell the analyst: "Phase 2 — launching rl-fp-validate."

Spawn: **rl-fp-validate** — pass full triage handoff contents. Also pass inspect
findings if they exist — content analysis signals are relevant FP evidence.

Write: `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

Tell the analyst the FP confidence level and proceed to Phase 3.

---

## Phase 3 — Investigation

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

Tell the analyst: "Phase 4 — launching rl-adjudicate."

Spawn: **rl-adjudicate** — pass triage handoff + FP findings + investigation findings.

Write: `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

**Route on routing decision:**

- **CONFIRMED FP** → Skip Phase 5.
  Tell the analyst:
  > "Adjudication: CONFIRMED FP — [rationale]. Skipping remediation."
  Go to Phase 6.

- **CONFIRMED TP** or **UNCERTAIN** → go to Phase 5.
  Tell the analyst the verdict and confidence.

---

## Phase 5 — Remediation

Tell the analyst: "Phase 5 — launching rl-remediate."

Spawn: **rl-remediate** — pass triage handoff + investigation findings + adjudication verdict.

Write: `<ARTIFACT_SLUG>/rl-remediate-playbook-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

---

## Phase 6 — Reporting

Tell the analyst: "Phase 6 — launching rl-report."

Spawn: **rl-report** — pass all available phase outputs (triage, content
inspection if ran, FP validation if ran, investigation if ran, adjudication,
remediation if ran).

After it returns, confirm both output files exist:
- `<ARTIFACT_SLUG>/incident-<INVESTIGATION_ID>-<ARTIFACT_SLUG>-report.md`
- `<ARTIFACT_SLUG>/incident-<INVESTIGATION_ID>-<ARTIFACT_SLUG>-iocs.json`

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
`FULL`

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
- Paths to incident report and IOC JSON
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
  | rl-remediate *(if ran)* | sonnet/medium | N | N |
  | rl-report *(if ran)* | sonnet/medium | N | N |
  | **Total** | | **N** | **N** |
