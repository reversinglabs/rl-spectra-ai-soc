---
name: rl-remediate
description: >
  SOC remediation phase. Reads triage, investigation, and adjudication
  findings and produces a prioritized containment and eradication playbook.
  Writes rl-remediate-playbook.md. Only runs when adjudication verdict is
  CONFIRMED TP or UNCERTAIN. Trigger after adjudication, or as part of the
  rl-soc pipeline.
---

# rl-remediate

Fifth phase of the SOC analysis pipeline (conditional). Produces a prioritized
containment and eradication playbook based on all prior phase findings.

**This phase only runs if the adjudication verdict is CONFIRMED TP or UNCERTAIN.**
If the verdict is CONFIRMED FP, skip this phase.

## CRITICAL CONSTRAINTS

- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, or venv paths.**

## Step 1 — Locate handoff files

If invoked by the rl-soc orchestrator, the investigation ID and handoff
contents are provided directly — skip file discovery.

If invoked standalone, find the investigation to continue:

```bash
find . -maxdepth 2 -name "rl-adjudication-verdict-*.md" 2>/dev/null | sort -r
```

- **No files found**: Stop and tell the analyst to run `rl-adjudicate` first.
- **One file found**: Use it. `ARTIFACT_SLUG` is the name of the parent directory;
  extract `INVESTIGATION_ID` from the filename (last 15 characters before `.md`).
- **Multiple files found**: List them and ask the analyst which to use.

Read the following files:
- `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (required)
- `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (required)
- `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (for IOCs and context)

Check the adjudication verdict. If it is **CONFIRMED FP**, stop and tell the analyst:
> Adjudication verdict is CONFIRMED FP — remediation is not required.
> Proceeding to reporting.

## Step 2 — Spawn rl-remediate agent

Use the Task tool to spawn the `rl-remediate` agent. Pass the full contents of:
- `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`
- `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`

Wait for the agent to return.

## Step 3 — Write rl-remediate-playbook-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md

Write the agent's output to `<ARTIFACT_SLUG>/rl-remediate-playbook-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md`.

## Step 4 — Report to analyst

Tell the analyst:
- Containment action count (P0 / P1 / P2 breakdown)
- Eradication action count
- That `<ARTIFACT_SLUG>/rl-remediate-playbook-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` has been written
- The next step: run `rl-report`
