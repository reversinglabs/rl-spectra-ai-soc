---
name: rl-investigate
description: >
  SOC investigation phase. Reads rl-triage-handoff.md and performs deep
  enrichment by spawning three parallel sub-agents: IOC enrichment, sample
  pivots, and YARA hunting with sandbox polling. Writes
  rl-investigate-findings.md. Trigger when triage and FP validation are
  complete and a full investigation is needed, or as part of the rl-soc
  pipeline.
---

# rl-investigate

Third phase of the SOC analysis pipeline. Performs deep enrichment of all
triage IOCs using three parallel sub-agents and writes findings to
`rl-investigate-findings.md`.

## CRITICAL CONSTRAINTS

- **Do NOT perform any manual file analysis.**
- **Do NOT query any external threat intelligence service.**
- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, or venv paths.**

## Step 1 — Locate handoff files

If invoked by the rl-soc orchestrator, the investigation ID and handoff
contents are provided directly — skip file discovery.

If invoked standalone, find the triage handoff file:

```bash
find . -maxdepth 2 -name "rl-triage-handoff-*.md" 2>/dev/null | sort -r
```

- **No files found**: Stop and tell the analyst to run `rl-triage` first.
- **One file found**: Use it. `ARTIFACT_SLUG` is the name of the parent directory;
  extract `INVESTIGATION_ID` from the filename (last 15 characters before `.md`).
- **Multiple files found**: List them and ask the analyst which investigation
  to continue (show artifact slug, ID, and modification time for each).

Read `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`. Also read
`<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` if it exists —
pass it to the sub-agents as additional context.

## Step 2 — Run parallel sub-agents

Use the Task tool to spawn ALL THREE sub-agents simultaneously in a single turn:

**rl-investigate-enrich**: Pass all IOCs from `rl-triage-handoff.md`. This agent
performs bulk reputation lookups for all file hashes and network indicators.

**rl-investigate-pivot**: Pass triage classification data (threat name, signer,
C2 infrastructure, imphash if available). This agent performs advanced search
pivots and certificate analytics.

**rl-investigate-hunt**: Pass the subject SHA256 and any in-flight analysis IDs
from `rl-triage-handoff.md`. This agent runs a YARA retrohunt and polls sandbox
results.

Wait for all three sub-agents to return before proceeding.

## Step 3 — Synthesize results

Combine all three sub-agent outputs. Identify:
- The highest-confidence malicious IOCs across all enrichment
- The most significant related sample clusters from pivoting
- New behavioral indicators from sandbox completion
- YARA hunt cluster summary
- Full consolidated MITRE ATT&CK mapping (combine triage TTPs with new findings)

**NEUTRALITY REQUIREMENT**: The findings document is a factual record. Do NOT draw
TP/FP conclusions, characterize evidence as "decisive," or pre-adjudicate the verdict.
Present what each sub-agent found — classifications, counts, ratios, patterns — and
let the adjudicator weigh their significance. The words "false positive," "true positive,"
"overrides," "decisively," "confirms," and "establishes" must NOT appear in any preamble
or summary section. State facts: "X samples on this cert are MALICIOUS" — not "cert abuse
evidence decisively overrides the FP hypothesis."

## Step 4 — Write rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md

Write the synthesized findings to `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.
The file must contain:

```markdown
# Investigation Findings

## Subject (from triage)
- <subject summary>

## IOC Enrichment Results
<paste enrich sub-agent output>

## Pivot and Certificate Results
<paste pivot sub-agent output>

## Hunt and Sandbox Results
<paste hunt sub-agent output>

## Consolidated MITRE ATT&CK Mapping
| Tactic | Technique ID | Technique Name | Evidence |
|---|---|---|---|

## Campaign / Actor Attribution
- <synthesis from pivot + cert + hunt data, or "Insufficient evidence">

## Key Findings for Adjudication
- <3–5 bullet points summarizing the most significant data points found>
- State facts and Spectra classifications only — do NOT conclude TP or FP here
- Example: "Certificate 41386D has 0 MALICIOUS and 2 SUSPICIOUS samples out of 4,758 total"
- Example: "ClipBanker (SHA1: 0967394...) classified SUSPICIOUS (3/29 AV engines) on same cert"
- The adjudicator weighs these facts — this section must not prejudge the outcome
```

## Step 5 — Report to analyst

Tell the analyst:
- Summary of enrichment (IOC count, malicious count)
- Whether sandbox data came back and any key behavioral findings
- YARA hunt match count
- Attribution findings (if any)
- That `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` has been written
- The next step: run `rl-adjudicate`
