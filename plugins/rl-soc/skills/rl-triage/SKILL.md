---
name: rl-triage
description: >
  SOC triage phase. Takes a suspicious file, hash, URL, IP, or domain and
  produces an initial classification using ReversingLabs Spectra Intelligence.
  For file and hash artifacts, spawns three parallel sub-agents: classification,
  sandbox orchestration, and content inspection (Text/* types). For network
  indicators, spawns two: classification and sandbox only. Writes
  rl-triage-handoff.md and (if applicable) rl-triage-inspect-findings.md.
  Trigger when an analyst wants to triage an artifact, or as the first phase
  of the full rl-soc pipeline.
---

# rl-triage

First phase of the SOC analysis pipeline. Produces an initial classification
of the artifact, starts any missing sandbox analyses, and (for text-based file
types) semantically analyzes the raw file content. Writes results to
`rl-triage-handoff.md` and (if applicable) `rl-triage-inspect-findings.md`
for downstream phases.

## CRITICAL CONSTRAINTS

- **Do NOT perform any manual file analysis.** Never run `xxd`, `strings`,
  `file`, `hexdump`, or any local tool on an artifact. Pass file paths as
  strings to sub-agents — all analysis goes through Spectra Intelligence.
- **Do NOT query any external threat intelligence service.**
- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, or venv paths.**

## Step 1 — Get the artifact and generate investigation ID

If invoked standalone (not by the rl-soc orchestrator), ask the analyst for:
- The artifact: file path, hash (SHA256/SHA1/MD5), URL, IP, or domain
- Optional context: source, ticket ID, urgency, or other relevant information

Then generate a unique investigation ID:
```bash
date +%Y%m%d-%H%M%S
```

Also derive `ARTIFACT_SLUG`:

| Artifact type | ARTIFACT_SLUG |
|---|---|
| SHA256 hash | First 12 characters of the hash |
| SHA1 or MD5 hash | First 12 characters of the hash |
| File upload | `upload` now; replace with first 12 chars of SHA256 after sub-agents return |
| URL | Hostname with `.` → `-`, truncated to 20 chars |
| IP address | Dots replaced with hyphens |
| Domain | Dots replaced with hyphens, truncated to 20 chars |

Tell the analyst: "Starting investigation `<INVESTIGATION_ID>`."

If invoked by the rl-soc orchestrator, the artifact, context, investigation ID,
and ARTIFACT_SLUG are provided directly — do not generate new ones.

## Step 2 — Run parallel sub-agents

### Hash artifacts (SHA256, SHA1, MD5)

Spawn ALL THREE sub-agents simultaneously in a single Task turn:

**rl-triage-classify**: Pass the hash and any analyst context.

**rl-triage-sandbox**: Pass the hash.

**rl-triage-inspect**: Pass the hash. The agent determines file type and size
itself via `get_sample_overview` and skips automatically if the type is not
`Text/*`, the file exceeds 722KB, or the sample is unknown.

Wait for all three to return before proceeding.

### File upload artifacts

Spawn only **rl-triage-classify** and **rl-triage-sandbox** simultaneously.

After rl-triage-classify returns with the SHA256, file type, and file size,
decide whether to spawn rl-triage-inspect:
- If file type starts with `Text/` AND size ≤ 739,328 bytes → spawn
  **rl-triage-inspect**, passing the SHA256.
- Otherwise → skip rl-triage-inspect.

Wait for rl-triage-inspect to complete (if spawned) before proceeding.

### Network indicators (URL, IP, domain)

Spawn only **rl-triage-classify** and **rl-triage-sandbox** — do not spawn
rl-triage-inspect (no file content to analyze).

## Step 3 — Synthesize results

Combine the outputs from both sub-agents into a unified triage handoff document.
Use the classification results from `rl-triage-classify` and the sandbox status
from `rl-triage-sandbox`.

## Step 4 — Write rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md

If the artifact was a file upload, update `ARTIFACT_SLUG` now using the first
12 characters of the SHA256 returned by `rl-triage-classify`.

Create the output directory and write the handoff file:

```bash
mkdir -p <ARTIFACT_SLUG>
```

Write the synthesized results to `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

The file must contain:

```markdown
# Triage Handoff

## Subject
- **Type**: file | hash | url | ip | domain
- **Value**: <SHA256 or indicator value>
- **Original input**: <what the analyst provided>
- **Analyst context**: <if any was provided>

## Verdict
- **Classification**: CONFIRMED MALICIOUS | SUSPICIOUS | LIKELY BENIGN | NEEDS INVESTIGATION
- **Threat name** (if any): <e.g. Win32.Trojan.Emotet>
- **Confidence**: HIGH | MEDIUM | LOW
- **Rationale**: 1–3 sentences

## Spectra Data
- Sample overview: <key fields>
- Detection rules matched: <list>
- Behavioral indicators: <key items>
- MITRE TTPs: <list of technique IDs>
- File metadata: <type, size, signer if signed>
- AV vendor breakdown: <count flagging / total, lone-wolf flags noted>

## IOCs Identified
### File hashes
- <SHA256, SHA1, MD5 of subject and any dropped files>
### Network indicators
- <IPs, domains, URLs>
### Other
- <Email addresses, certificate thumbprints, mutexes, named pipes>

## Sandbox Status
- Dynamic analysis: COMPLETE | IN_PROGRESS | STARTED_NOW | NOT_APPLICABLE
- Dynamic analysis ID: <id or null>
- Auxiliary analysis: COMPLETE | IN_PROGRESS | STARTED_NOW | NOT_APPLICABLE | SKIPPED
- Auxiliary analysis ID: <id or null>

## In-flight Analyses (for investigation to poll)
- Dynamic: <id or "none">
- Auxiliary: <id or "none">

## Open Questions
- <any ambiguity for downstream phases>
```

## Step 5 — Handle content inspection findings

If rl-triage-inspect ran and produced output (i.e. did not immediately skip):

Write the agent's output to
`<ARTIFACT_SLUG>/rl-triage-inspect-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

Check the **Overall Verdict**:

- **CONCLUSIVE BENIGN** — The content is clearly benign. Note that FP
  validation, investigation, adjudication, and remediation can be skipped —
  proceed directly to reporting if running the full pipeline.

- **CONCLUSIVE MALICIOUS** — The content contains clear hostile code. Note
  that FP validation can be skipped; investigation should run next.

- **INCONCLUSIVE** or **SKIPPED** — Proceed normally — FP validation runs next.

## Step 6 — Report to analyst

Tell the analyst:
- The triage verdict and threat name (if any)
- The content inspection verdict and key signals (if inspect ran and was not INCONCLUSIVE)
- Whether sandbox analyses are in-flight
- The investigation ID: `<INVESTIGATION_ID>`
- The artifact slug: `<ARTIFACT_SLUG>`
- Files written:
  - `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (always)
  - `<ARTIFACT_SLUG>/rl-triage-inspect-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (if inspect ran and produced output)
- The next step based on the content inspection verdict, or run `rl-fp-validate` if INCONCLUSIVE or not applicable (or the rl-soc orchestrator will continue automatically)
