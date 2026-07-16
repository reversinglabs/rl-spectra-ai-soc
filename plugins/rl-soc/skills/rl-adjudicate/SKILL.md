---
name: rl-adjudicate
description: >
  SOC adjudication phase. Reads FP validation findings and investigation
  findings, spawns the rl-adjudicate agent to weigh all evidence and produce
  a final CONFIRMED TP / CONFIRMED FP / UNCERTAIN verdict with confidence and
  a full per-signal audit trail. Gates transition to remediation. Writes
  rl-adjudication-verdict.md. Trigger after investigation is complete, or as
  part of the rl-soc pipeline.
---

# rl-adjudicate

Fourth phase of the SOC analysis pipeline. Renders a final verdict by weighing
FP validation evidence against investigation findings. Determines whether the
pipeline proceeds to remediation or closes as a false positive.

## CRITICAL CONSTRAINTS

- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, or venv paths.**

## Step 1 — Locate handoff files

If invoked by the rl-soc orchestrator, the investigation ID and handoff
contents are provided directly — skip file discovery.

If invoked standalone, find the investigation to continue:

```bash
find . -maxdepth 2 -name "rl-investigate-findings-*.md" 2>/dev/null | sort -r
```

- **No files found**: Stop and tell the analyst to run `rl-investigate` first.
- **One file found**: Use it. `ARTIFACT_SLUG` is the name of the parent directory;
  extract `INVESTIGATION_ID` from the filename (last 15 characters before `.md`).
- **Multiple files found**: List them and ask the analyst which to use.

Read the following files (all required):
- `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (for subject context)

If any required file is missing, stop and tell the analyst which phase to run first.

## Step 2 — Spawn rl-adjudicate agent

Use the Task tool to spawn the `rl-adjudicate` agent. Pass the full contents of:
- `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

Wait for the agent to return.

## Step 3 — Write rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md

Write the agent's output to `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

## Step 4 — Route based on verdict

Read the **Final Verdict** and **Routing Decision** from the output:

- **CONFIRMED TP** or **UNCERTAIN**: Proceed to `rl-remediate`. Tell the analyst
  the verdict, confidence, and that the remediation playbook will be generated.

- **CONFIRMED FP**: Tell the analyst the verdict and key audit trail signals.
  Skip `rl-remediate` and proceed directly to `rl-report` (or inform the
  orchestrator to skip remediation).

## Step 5 — Report to analyst

Tell the analyst:
- The final verdict and confidence level
- The FP critique assessment (appropriate / too aggressive / too weak)
- 2–3 key signals from the audit trail that drove the decision
- That `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` has been written
- The next step based on the routing decision
