---
name: rl-soc
description: >
  Full SOC threat-analysis pipeline. Give it a file, hash, URL, IP, or domain
  and it runs all phases in sequence: triage → FP validation → investigation →
  adjudication → remediation (if TP) → reporting. Triggers on "analyze this
  hash", "investigate this URL", "is this malware", "run threat analysis on",
  "I got a suspicious file", or any submission of an indicator for end-to-end
  SOC workflow.
---

# rl-soc

Full multi-phase SOC threat-analysis pipeline backed by ReversingLabs Spectra
Intelligence. Runs all phases in sequence, with parallel sub-agents within
triage and investigation for speed.

All Spectra Intelligence calls are made via the `rl-spectra-intel` CLI using the
`Bash` tool — no MCP server required.

## CRITICAL CONSTRAINTS

- **Do NOT perform any manual analysis of files.** Never run `xxd`, `strings`,
  `file`, `hexdump`, `cat`, or any tool to inspect file contents. Pass file
  paths as plain strings to sub-agents. All analysis goes through Spectra.
- **Use the `rl-spectra-intel` CLI via `Bash` for all Spectra calls.**
  Do NOT use MCP tools. Do NOT call the Spectra API directly.
  Pattern: `rl-spectra-intel <tool_name> --args '<json>'` — single-line Bash,
  exit codes: 0 success / 1 API error / 2 bad args.
- **Use Spectra Intelligence tools exclusively.** Do NOT query VirusTotal,
  URLhaus, AbuseIPDB, Shodan, AlienVault OTX, MalwareBazaar, or any other
  external platform.
- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, config files, or the venv.**

## You are the orchestrator

You manage the pipeline. Run phases in order using the Task tool to spawn
sub-agents. Write handoff files after each phase. Route conditionally based
on FP validation and adjudication results.

---

## Pre-Step — Determine pipeline scope

Before doing anything else, determine how much of the pipeline the analyst wants.

**Infer from the request:**

| If the request says… | Scope |
|---|---|
| "analyze", "full analysis", "is this malware", "threat analysis", "run the pipeline", "SOC workflow" | **FULL** — all 6 phases |
| "triage only", "classify", "quick check", "just triage" | **TRIAGE_ONLY** — phase 1 only |
| "investigate only", "triage and investigate", "investigate but skip report" | **INVESTIGATE** — phases 1–3 |
| "is this a FP", "is this a TP", "evaluate this detection", "verdict only", "just tell me if it's malicious", "FP/TP check" | **VERDICT_ONLY** — phases 1–4, no remediation or report |
| Unclear — none of the above clearly match | **Ask** |

**If unclear**, ask before starting:
> "Should I run the full pipeline including remediation playbook and incident report?
> Those phases take longer and consume significantly more tokens. Or would you prefer:
> - **Full pipeline** — all 6 phases including remediation and incident report
> - **Verdict only** — triage + investigation + FP/TP verdict (no remediation or report)
> - **Triage only** — quick classification and sandbox kickoff only"

Store the scope as `PIPELINE_SCOPE` before proceeding.

---

## Step 0 — Generate investigation ID and artifact slug

Before starting any phase, generate a unique investigation ID:

```bash
date +%Y%m%d-%H%M%S
```

Store this as `INVESTIGATION_ID`.

Also derive `ARTIFACT_SLUG` from the artifact to make all handoff files self-describing:

| Artifact type | ARTIFACT_SLUG |
|---|---|
| SHA256 hash | First 12 characters of the hash (e.g. `a3f8e91b2c04`) |
| SHA1 or MD5 hash | First 12 characters of the hash |
| File upload | `upload` for now; **replace with first 12 chars of SHA256 after Phase 1 returns** |
| URL | Hostname with `.` replaced by `-`, truncated to 20 chars (e.g. `evil-example-com`) |
| IP address | Dots replaced with hyphens (e.g. `192-168-1-1`) |
| Domain | Dots replaced with hyphens, truncated to 20 chars (e.g. `malware-example-com`) |

All handoff files are written to a subdirectory named `<ARTIFACT_SLUG>/` in the
current working directory. For hash and network artifacts, create the directory
immediately after deriving the slug:

```bash
mkdir -p <ARTIFACT_SLUG>
```

For file uploads, create the directory after Phase 1 returns the SHA256 and
ARTIFACT_SLUG is finalized (see Phase 1).

Files follow the pattern `<ARTIFACT_SLUG>/<name>-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

### Pre-lookup for hash artifacts

For **hash** artifacts (SHA256, SHA1, MD5), call `get_sample_overview` now —
before spawning any sub-agents — to determine the file type and size:

```bash
rl-spectra-intel get_sample_overview --args '{"hash_value": "<hash>"}'
```

Store the result as `SAMPLE_OVERVIEW`. Extract:
- `SAMPLE_FILE_TYPE` — the Spectra file type (e.g. `Text/PowerShell`, `PE/Exe`)
- `SAMPLE_SIZE` — file size in bytes

If the call fails or the sample is unknown, store `SAMPLE_FILE_TYPE = unknown`
and `SAMPLE_SIZE = unknown`. This is non-fatal — triage will proceed and
rl-triage-inspect will handle the unknown case itself.

Skip this pre-lookup for **file uploads** (SHA256 not yet known) and **network
indicators** (no file to inspect).

Tell the analyst:
> "Starting investigation `<INVESTIGATION_ID>` for `<artifact>`."

---

## Phase 1 — Triage + Content Inspection

### Hash artifacts

Spawn ALL THREE sub-agents simultaneously in a single Task turn:

**rl-triage-classify** — artifact identification, classification, indicators, TTPs,
detection rules. Pass the full `SAMPLE_OVERVIEW` JSON from the pre-lookup so the
agent does not repeat the `get_sample_overview` call.

**rl-triage-sandbox** — dynamic/auxiliary analysis status check and kickoff.

**rl-triage-inspect** — pass the SHA256, `SAMPLE_FILE_TYPE`, and `SAMPLE_SIZE`
(from the pre-lookup above). The agent will use these directly and skip
`get_sample_overview`. If `SAMPLE_FILE_TYPE` is not `Text/*` or `SAMPLE_SIZE`
exceeds 722KB, the agent will skip immediately without making any API calls.

### File upload artifacts

Spawn only **rl-triage-classify** and **rl-triage-sandbox** simultaneously.
After rl-triage-classify returns the SHA256, file type, and file size, check
whether rl-triage-inspect is applicable:
- If file type starts with `Text/` AND size ≤ 739,328 bytes → spawn
  **rl-triage-inspect**, passing the SHA256, file type, and size.
- Otherwise → skip rl-triage-inspect.

Wait for rl-triage-inspect to complete before proceeding.

Update `ARTIFACT_SLUG` using the first 12 characters of the SHA256, then create
the directory:

```bash
mkdir -p <ARTIFACT_SLUG>
```

### Network indicators (URL, IP, domain)

Spawn only **rl-triage-classify** and **rl-triage-sandbox** — do not spawn
rl-triage-inspect (no file content to analyze).

---

After all applicable sub-agents return, synthesize their outputs.

Write the triage handoff to `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

If rl-triage-inspect ran and produced output, write
`<ARTIFACT_SLUG>/rl-triage-inspect-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

**Check the rl-triage-inspect verdict (if it ran):**

- **CONCLUSIVE BENIGN** → Skip Phases 2, 3, and 4. Skip Phase 5.
  Tell the analyst:
  > "Triage complete. Content inspection: CONCLUSIVE BENIGN — [key signals].
  > Skipping FP validation, investigation, and remediation. Generating report."
  Go directly to Phase 6 (reporting).

- **CONCLUSIVE MALICIOUS** → Skip Phase 2 only. Go directly to Phase 3.
  Tell the analyst:
  > "Triage complete. Content inspection: CONCLUSIVE MALICIOUS — [key signals].
  > Skipping FP validation. Proceeding to investigation."

- **INCONCLUSIVE**, **SKIPPED**, or **did not run** → Continue to Phase 2.
  Give the analyst a status update:
  > "Triage complete — [verdict]. Proceeding to FP validation."

---

## Phase 2 — FP Validation
*(Skip if rl-triage-inspect returned CONCLUSIVE BENIGN or CONCLUSIVE MALICIOUS)*

Spawn: **rl-fp-validate** — pass full `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` contents.
Also pass `<ARTIFACT_SLUG>/rl-triage-inspect-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
if it exists — content analysis signals are relevant FP evidence.

After it returns, write `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

**Check the recommendation:**

- **CONCLUSIVE FP** → Skip Phases 3 and 4. Skip Phase 5. Go directly to Phase 6.
  Tell the analyst:
  > "FP validation returned CONCLUSIVE FP — [key signals]. Skipping investigation
  > and remediation. Generating report."

- **POSSIBLE FP / LIKELY TP / INSUFFICIENT DATA** → Continue to Phase 3.
  Tell the analyst the FP confidence level and proceed.

---

## Phase 3 — Investigation
*(Skip if Phase 1.5 returned CONCLUSIVE BENIGN)*
*(Skip if Phase 2 returned CONCLUSIVE FP)*

Spawn ALL THREE sub-agents simultaneously in a single Task turn:

**rl-investigate-enrich** — bulk IOC enrichment.
**rl-investigate-pivot** — advanced search pivots + certificate analytics.
**rl-investigate-hunt** — YARA retrohunt + sandbox polling.

After all three return, synthesize and write
`<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

Give the analyst a status update:
> "Investigation complete. Starting adjudication."

---

## Phase 4 — Adjudication
*(Skip if Phase 2 returned CONCLUSIVE FP)*

Spawn: **rl-adjudicate** — pass triage handoff + FP findings + investigation findings.

After it returns, write `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

**Check the routing decision:**

- **CONFIRMED FP** → Skip Phase 5. Go to Phase 6.
  Tell the analyst:
  > "Adjudication: CONFIRMED FP — [rationale]. Skipping remediation."

- **CONFIRMED TP** or **UNCERTAIN** → Continue to Phase 5.
  Tell the analyst the verdict and confidence.

---

## Phase 5 — Remediation
*(Only runs if `PIPELINE_SCOPE` is `FULL` AND adjudication verdict is CONFIRMED TP or UNCERTAIN)*
*(Skip entirely if `PIPELINE_SCOPE` is `VERDICT_ONLY`, `INVESTIGATE`, or `TRIAGE_ONLY`)*

Spawn: **rl-remediate** — pass triage handoff + investigation findings + adjudication verdict.

After it returns, write `<ARTIFACT_SLUG>/rl-remediate-playbook-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

---

## Phase 6 — Reporting
*(Always runs when the pipeline short-circuited via CONCLUSIVE BENIGN or CONCLUSIVE FP)*
*(Otherwise only runs if `PIPELINE_SCOPE` is `FULL`; skip if `VERDICT_ONLY`, `INVESTIGATE`, or `TRIAGE_ONLY`)*

Spawn: **rl-report** — pass all available phase outputs (triage, content
inspection if ran, FP validation if ran, investigation if ran, adjudication,
remediation if ran).

After it returns, confirm both output files exist.

---

## Abort if blocked

If any phase fails (API error, agent returns nothing useful, file write fails),
surface the error to the analyst and ask how to proceed. Do not silently skip
a phase with missing data.

---

## Final summary

After all phases complete, tell the analyst:
- Final verdict and threat name (from adjudication, if adjudication ran)
- FP critique assessment (from adjudication, if adjudication ran)
- IOC count (if investigation ran)
- Which phases ran and which were skipped (including any skipped due to `PIPELINE_SCOPE`)
- Investigation ID (`<INVESTIGATION_ID>`)
- If Phase 6 ran: paths to both output files (`<ARTIFACT_SLUG>/incident-<INVESTIGATION_ID>-<ARTIFACT_SLUG>-report.md` and `<ARTIFACT_SLUG>/incident-<INVESTIGATION_ID>-<ARTIFACT_SLUG>-iocs.json`)
- If Phase 6 did not run: remind the analyst they can run `rl-report` or `rl-remediate` standalone if they want those outputs later
- Any UNCERTAIN verdict open questions for human review
- Token usage summary table collected from all agent `## Usage Estimate` sections:

  | Agent | Model | Input tokens | Output tokens |
  |---|---|---|---|
  | rl-triage-classify | sonnet/high | N | N |
  | rl-triage-sandbox | haiku/low | N | N |
  | rl-triage-inspect *(if ran)* | opus/high | N | N |
  | ... | | | |
  | **Total** | | **N** | **N** |

  Note: full breakdown is also embedded in the incident report.
