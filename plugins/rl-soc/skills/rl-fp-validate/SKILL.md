---
name: rl-fp-validate
description: >
  False positive validation phase. Reads rl-triage-handoff.md and evaluates
  multiple FP signals to determine whether the detection is likely a false
  positive. A CONCLUSIVE FP recommendation short-circuits the pipeline,
  skipping investigation and remediation. Writes rl-fp-validate-findings.md.
  Trigger when triage is complete and FP validation is needed, or as part of
  the full rl-soc pipeline.
---

# rl-fp-validate

Second phase of the SOC analysis pipeline. Evaluates FP signals from triage
data and determines whether the pipeline should continue to investigation or
short-circuit as a false positive.

## CRITICAL CONSTRAINTS

- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, or venv paths.**

## Step 1 — Locate triage handoff

If invoked by the rl-soc orchestrator, the investigation ID and triage
handoff contents are provided directly — skip file discovery.

If invoked standalone, find the triage handoff file:

```bash
find . -maxdepth 2 -name "rl-triage-handoff-*.md" 2>/dev/null | sort -r
```

- **No files found**: Stop and tell the analyst to run `rl-triage` first.
- **One file found**: Use it. `ARTIFACT_SLUG` is the name of the parent directory;
  extract `INVESTIGATION_ID` from the filename (last 15 characters before `.md`).
- **Multiple files found**: List them and ask the analyst which investigation
  to continue (show the artifact slug, ID, and modification time for each).

Read the chosen `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

## Step 2 — Spawn rl-fp-validate agent

Use the Task tool to spawn the `rl-fp-validate` agent. Pass the full contents
of `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` as its input.

Wait for the agent to return.

## Step 3 — Write rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md

Write the agent's output to `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

## Step 4 — Check for short-circuit

Read the **Recommendation** field from the findings:

- **CONCLUSIVE FP**: Tell the analyst the finding is a false positive and explain
  the key signals. Tell them the pipeline will skip investigation, adjudication,
  and remediation. Proceed directly to `rl-report` (or inform the rl-soc
  orchestrator to short-circuit).

- **POSSIBLE FP / LIKELY TP / INSUFFICIENT DATA**: Tell the analyst the FP
  confidence level and key signals. Proceed to `rl-investigate` (or inform the
  orchestrator to continue normally).

## Step 5 — Report to analyst

Tell the analyst:
- The FP recommendation and overall FP confidence
- The top 2–3 signals driving the assessment
- That `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` has been written
- The next step based on the recommendation
