---
name: rl-report
description: >
  SOC reporting phase. Reads all prior phase handoff files and synthesizes
  them into a narrative Markdown incident report and a machine-readable JSON
  IOC file. Trigger after all analysis phases are complete, or as part of the
  rl-soc pipeline.
---

# rl-report

Final phase of the SOC analysis pipeline. Synthesizes all prior phase outputs
into two deliverables written to the working directory.

## CRITICAL CONSTRAINTS

- **Do NOT use any `rl-protect` skills or tools.**
- **Do NOT check for credentials, `.env` files, or venv paths.**

## Step 1 — Locate handoff files

If invoked by the rl-soc orchestrator, the investigation ID and handoff
contents are provided directly — skip file discovery.

If invoked standalone, find the investigation to report on:

```bash
find . -maxdepth 2 -name "rl-adjudication-verdict-*.md" 2>/dev/null | sort -r
```

- **No files found**: Stop. At minimum, triage, FP validation, and adjudication
  must have completed. Tell the analyst which phases to run first.
- **One file found**: Use it. `ARTIFACT_SLUG` is the name of the parent directory;
  extract `INVESTIGATION_ID` from the filename (last 15 characters before `.md`).
- **Multiple files found**: List them and ask the analyst which to use.

Read all of the following files that exist for this investigation:
- `<ARTIFACT_SLUG>/rl-triage-handoff-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (required)
- `<ARTIFACT_SLUG>/rl-fp-validate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (required)
- `<ARTIFACT_SLUG>/rl-adjudication-verdict-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (required)
- `<ARTIFACT_SLUG>/rl-investigate-findings-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (if investigation ran — not present on CONCLUSIVE FP short-circuit)
- `<ARTIFACT_SLUG>/rl-remediate-playbook-<ARTIFACT_SLUG>-<INVESTIGATION_ID>.md` (if remediation ran — not present on FP verdict)

## Step 2 — Spawn rl-report agent

Use the Task tool to spawn the `rl-report` agent. Pass:
- The `INVESTIGATION_ID` and `ARTIFACT_SLUG`
- The full contents of all collected phase files, clearly labeled by phase

Wait for the agent to return.

## Step 3 — Report to analyst

Tell the analyst:
- The final verdict and threat name (from adjudication)
- IOC count
- Whether investigation and remediation data was included
- The paths to both output files
