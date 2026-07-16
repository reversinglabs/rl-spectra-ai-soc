# /analyze command

Run the full SOC threat-analysis pipeline on a file, hash, URL, IP, or domain.

## Usage

```
/rl-soc:analyze <indicator> [context]
```

- `<indicator>` — a file path, SHA256/SHA1/MD5 hash, URL, IP, or domain
- `[context]` — optional free-text context (source, ticket ID, urgency, etc.)

## Pipeline scope

The skill infers how much of the pipeline to run from your request. You can
also state it explicitly:

| Scope | Phases | Use when… |
|---|---|---|
| **Full pipeline** *(default)* | 1–6 | You want remediation guidance and an incident report |
| **Verdict only** | 1–4 | You want a TP/FP verdict without remediation or report |
| **Investigate** | 1–3 | You want enrichment and pivots but not a verdict yet |
| **Triage only** | 1 | You want a quick classification and sandbox kickoff |

If the intent is ambiguous, the skill will ask before starting and inform you
of the token cost difference.

Remediation and reporting are the most token-intensive phases. Run **verdict only**
or **triage only** when you just need a quick answer, and use the standalone
`rl-remediate` and `rl-report` skills later if needed.

## Examples

```
/rl-soc:analyze a1b2c3d4e5f6...
/rl-soc:analyze https://suspicious.example.com Found in phishing email to finance team
/rl-soc:analyze 192.0.2.45 Beaconing detected from finance-laptop-07
/rl-soc:analyze /mnt/user-data/uploads/sample.exe Suspicious attachment from external sender

# Explicit scope
/rl-soc:analyze a1b2c3d4e5f6... verdict only
/rl-soc:analyze a1b2c3d4e5f6... is this a FP or TP?
/rl-soc:analyze a1b2c3d4e5f6... triage only, quick check
```

For full pipeline details, see the `rl-soc` skill. Individual phases can
also be run standalone: `rl-triage`, `rl-fp-validate`, `rl-investigate`,
`rl-adjudicate`, `rl-remediate`, `rl-report`.
