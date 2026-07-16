# ReversingLabs Plugins for Claude Code and Cowork

This repository contains plugins and skills that integrate **ReversingLabs**
malware analysis and threat hunting capabilities
into [Claude Code](https://claude.ai/code) and Claude Cowork.

Analysts can triage suspicious files, hashes, URLs, IPs, and domains; run
deep multi-agent investigations; receive prioritized remediation playbooks;
and generate structured incident reports — all without leaving their AI
coding assistant.

---

## Plugins

| Plugin | Purpose |
|---|---|
| [`rl-soc`](#rl-soc) | **Flagship.** Multi-agent SOC workflow via the `rl-spectra-intel` CLI |
| [`rl-soc-install`](#rl-soc-install) | Installs the bundled `rl-spectra-intel` CLI |
| [`rl-soc-connect`](#rl-soc-connect) | Configures `rl-spectra-intel` with ReversingLabs credentials |

---

## Quickstart

### Install

```
/plugin marketplace add https://github.com/reversinglabs/rl-spectra-ai-soc.git

/plugin install rl-soc-install
/plugin install rl-soc-connect
/plugin install rl-soc

/reload-plugins
```

### Setup

1. Run the `install` plugin to install the `rl-spectra-intel` cli tool:
```
/rl-soc-install
```
2. Launch a new terminal and a new Claude-code session
3. Run the `connect` plugin to set your ReversingLabs credentials:
```
/rl-soc-connect
```

### Update

```
/plugin delete rl-soc
/plugin delete rl-soc-install
/plugin delete rl-soc-connect

/plugin marketplace update rl-spectra-ai-soc

/plugin install rl-soc-install
/plugin install rl-soc-connect
/plugin install rl-soc

/reload-plugins
```

### Analyze

Use the `/rl-soc:analyze` command to begin the multi-phase multi-agent SOC workflow.

---

## rl-soc

The flagship plugin. Give it a file, hash, URL, IP, or domain and it runs a
full multi-phase SOC workflow using the `rl-spectra-intel` CLI, producing a
narrative incident report and a machine-readable IOC file.

### Multi-agent pipeline

The workflow is orchestrated by Claude's main session across six phases.
Triage and investigation each spawn multiple parallel sub-agents for speed.
FP validation gates investigation, and adjudication gates remediation — with
short-circuit paths when evidence is conclusive.

All ReversingLabs tool calls use the form:

```bash
rl-spectra-intel <tool_name> --args '<json_kwargs>'
```

Exit codes: `0` = success (JSON on stdout), `1` = API error (JSON on stderr), `2` = bad arguments. Agents check exit codes after every call and surface errors to the orchestrator rather than silently continuing.

Each phase is also independently invocable as a standalone skill — analysts can run just `rl-triage`, or re-run `rl-adjudicate` on existing handoff files without repeating prior phases.

#### Phase 1 — rl-triage

For file/hash artifacts, three parallel sub-agents run simultaneously. For network indicators (URL, IP, domain), only classify and sandbox run.

- **rl-triage-classify** — identifies the artifact type, submits files if needed, retrieves Spectra classification, behavioral indicators, MITRE ATT&CK TTPs, and detection rules.
- **rl-triage-sandbox** — checks for existing dynamic/auxiliary analyses and starts them if missing (60–90s timeout for speed). Does not wait for completion.
- **rl-triage-inspect** — calls `get_sample_overview` to determine if the sample is a `Text/*` type ≤ 722KB, then fetches the base64-encoded content via `get_sample_content` and semantically analyzes it across nine signals (code intent, obfuscation, network behavior, download-and-execute, persistence, credential access, evasion, file system operations, provenance). Produces a structured signal table and a CONCLUSIVE BENIGN / CONCLUSIVE MALICIOUS / INCONCLUSIVE verdict. Automatically skips for binary types, oversized files, ECONNRESET blocks by network security scanning, and network indicators.

  A **CONCLUSIVE BENIGN** verdict short-circuits the pipeline directly to reporting (or stops, for non-FULL scopes). A **CONCLUSIVE MALICIOUS** verdict skips FP validation and jumps directly to investigation.

Writes `rl-triage-handoff.md` and (if applicable) `rl-triage-inspect-findings.md`.

#### Phase 2 — rl-fp-validate

Evaluates whether the detection is likely a false positive before committing to a full investigation. Checks: AV vendor consensus, signer reputation, sample prevalence, threat name heuristics (generic.heur, packed.suspicious), detection sparsity, dual-use tool categories (PyInstaller, NSIS, Electron, AutoHotkey, etc.), admin/build context, and rule quality.

- **CONCLUSIVE FP** → skips investigation, adjudication, and remediation; goes directly to reporting.
- **POSSIBLE FP / LIKELY TP / INSUFFICIENT DATA** → proceeds to investigation.

Writes `rl-fp-validate-findings.md`.

#### Phase 3 — rl-investigate

Three parallel sub-agents run simultaneously:
- **rl-investigate-enrich** — bulk reputation lookups for all IOCs from triage.
- **rl-investigate-pivot** — advanced search pivots (threat name, signer, C2, imphash) and certificate analytics.
- **rl-investigate-hunt** — starts YARA retrohunt immediately, then polls in-flight sandbox results while the hunt runs.

Writes `rl-investigate-findings.md`.

#### Phase 4 — rl-adjudicate

Weighs FP validation findings against investigation evidence. Detects when FP critique was too aggressive (strong malicious evidence overrides it) or too weak (investigation found no corroboration). Produces a final verdict (CONFIRMED TP / CONFIRMED FP / UNCERTAIN), confidence level, and a full per-signal audit trail.

- **CONFIRMED FP** → skips remediation; goes to reporting.
- **CONFIRMED TP / UNCERTAIN** → proceeds to remediation.

Writes `rl-adjudication-verdict.md`.

#### Phase 5 — rl-remediate *(conditional)*

Produces a prioritized, human-executable containment and eradication playbook. Does not execute any action — instructions only. Actions ordered by urgency (P0 / P1 / P2) covering network blocks, endpoint isolation, identity controls, persistence cleanup, and detection engineering. Skipped entirely on a CONFIRMED FP verdict.

Writes `rl-remediate-playbook.md`.

#### Phase 6 — rl-report

Synthesizes all phase outputs into two files:

1. **`incident-<YYYYMMDD>-<id>-report.md`** — Narrative Markdown incident report including adjudication verdict and audit trail, executive summary, MITRE ATT&CK table, IOC summary, and (if TP) remediation playbook.

2. **`incident-<YYYYMMDD>-<id>-iocs.json`** — Schema-versioned JSON IOC file ready for ingestion by SOAR platforms and TIPs (e.g. MISP).

### Requirements

- `rl-spectra-intel` CLI installed and configured (see [`rl-soc-install`](#rl-soc-install) and [`rl-soc-connect`](#rl-soc-connect)).

### Allowing rl-spectra-intel without approval prompts

By default Claude Code will prompt for approval on every `rl-spectra-intel` call. To allow it automatically, add the following to `~/.claude/settings.json` (applies globally) or `.claude/settings.json` in your project (applies to that project only):

```json
{
  "permissions": {
    "allow": [
      "Bash(~/.local/bin/rl-spectra-intel *)",
      "Bash(rl-spectra-intel *)"
    ]
  }
}
```

### Usage

```
/rl-soc:analyze <indicator> [optional context]

/rl-soc:analyze a1b2c3d4e5f6...
/rl-soc:analyze https://suspicious.example.com Found in phishing email
/rl-soc:analyze 192.0.2.45 Beaconing from finance-laptop-07
/rl-soc:analyze /mnt/user-data/uploads/sample.exe Suspicious attachment
```

Or via natural language — the skill triggers automatically on requests like
"analyze this hash", "is this file malware?", or "investigate this domain".

### Customizing the agents

Each agent is a Markdown file in `agents/` with YAML frontmatter:

- **`model:`** — `haiku`, `sonnet`, or `opus`. Lightweight sub-agents (`rl-triage-sandbox`, `rl-investigate-enrich`) use `haiku`; classification, pivot, and hunt agents use `sonnet`; high-stakes reasoning agents (`rl-fp-validate`, `rl-adjudicate`, `rl-triage-inspect`) use `opus`.
- **`effort:`** — `low` / `medium` / `high` reasoning effort.
- **`maxTurns:`** — increase if an agent hits the limit on deep investigations.
- **`disallowedTools:`** — all agents except `rl-report` and `rl-adjudicate` are read-only (Write, Edit, MultiEdit blocked). `rl-adjudicate` also blocks Bash since it makes no API calls.

### Safety notes

- `rl-remediate` produces instructions only — the plugin never touches your EDR, firewall, or identity systems.
- Only `rl-report` writes incident files; phase handoff files (`.md`) are written by the orchestrator skill.
- A full workflow makes 30–60 Spectra API calls across all parallel sub-agents — check your account rate limits before running multiple pipelines simultaneously.
- `rl-adjudicate` makes no API calls — it is a pure reasoning step over prior phase outputs.

---

## rl-soc-install

Installs the bundled `rl-spectra-intel` Python package. Run this once before using `rl-soc`.

The skill:
1. Checks whether `rl-spectra-intel` is already installed in the venv — exits early if so.
2. Verifies Python 3.10 or later is available (required by the CLI).
3. Creates a dedicated virtual environment at `~/.rl-spectra-intel-venv`.
4. Installs `rl-spectra-intel` into the venv.
5. Verifies the installed binary is reachable in the venv.
6. Instructs the user to close Claude Code, open a new terminal, relaunch Claude Code, and then run `rl-soc-connect` to configure credentials.

---

## rl-soc-connect

Configures the `rl-spectra-intel` CLI with ReversingLabs credentials. Run
this after `rl-soc-install`, and after restarting your terminal.

The skill:
1. Prompts for `rl_username`, `rl_password`, and optionally `rl_host`.
2. Warns the user that credentials will be stored in plain text at `~/.rl-spectra-intel-venv/.env` (mode `600`) and requires explicit confirmation before proceeding.
3. Writes credentials to `~/.rl-spectra-intel-venv/.env`.
4. Creates a wrapper script at `~/.local/bin/rl-spectra-intel` that sources the credentials file before every invocation — so credentials are injected automatically without modifying the system environment.

---

## Repository structure

```
plugins/
├── rl-soc/                            # Flagship: CLI-based SOC workflow (rl-spectra-intel)
│   ├── agents/
│   │   ├── rl-triage-classify.md      # Triage: artifact classification, indicators, TTPs
│   │   ├── rl-triage-sandbox.md       # Triage: dynamic/auxiliary analysis orchestration
│   │   ├── rl-triage-inspect.md       # Triage: text-based content analysis (opus/high)
│   │   ├── rl-fp-validate.md          # FP validation: 8-signal scoring + recommendation
│   │   ├── rl-investigate-enrich.md   # Investigation: bulk IOC enrichment
│   │   ├── rl-investigate-pivot.md    # Investigation: advanced search pivots + cert analytics
│   │   ├── rl-investigate-hunt.md     # Investigation: YARA retrohunt + sandbox polling
│   │   ├── rl-adjudicate.md           # Adjudication: final TP/FP verdict + audit trail
│   │   ├── rl-remediate.md            # Remediation: prioritized containment playbook
│   │   └── rl-report.md               # Reporting: incident report + IOC file
│   ├── skills/
│   │   ├── rl-triage/                 # Standalone triage skill (includes content inspection)
│   │   ├── rl-fp-validate/            # Standalone FP validation skill
│   │   ├── rl-investigate/            # Standalone investigation skill
│   │   ├── rl-adjudicate/             # Standalone adjudication skill
│   │   ├── rl-remediate/              # Standalone remediation skill
│   │   ├── rl-report/                 # Standalone reporting skill
│   │   └── rl-soc/                    # Full pipeline orchestrator skill
│   ├── .claude-plugin/
│   └── commands/
├── rl-soc-install/                    # Installs rl-spectra-intel CLI
│   ├── .claude-plugin/
│   └── skills/rl-soc-install/
└── rl-soc-connect/                    # Configures rl-spectra-intel credentials
    ├── .claude-plugin/
    └── skills/rl-soc-connect/
.claude-plugin/
└── marketplace.json                   # Marketplace manifest listing all plugins
```