---
name: rl-soc-install
description: >
  Installs a ReversingLabs SOC CLI from a bundled wheel included with this
  plugin. Prompts for which service to set up — Spectra Intelligence (cloud) or
  Spectra Analyze (on-prem appliance) — then installs the matching CLI into a
  dedicated venv. Trigger when the user wants to set up the ReversingLabs SOC
  tooling, when the CLI is not found on PATH, or when rl-soc reports the CLI is
  missing.
---

# rl-soc-install

Installs a ReversingLabs SOC CLI into a dedicated Python virtual environment.
Credential/appliance configuration is handled by `rl-soc-connect` after
installation.

## Step 0 — Ask which service to configure

Ask the user:

> Which ReversingLabs service do you want to set up?
>
> 1. **Spectra Intelligence** — cloud threat intelligence (username + password auth)
> 2. **Spectra Analyze** — on-prem appliance (appliance URL + API token auth)
> 3. **Both** — set up Spectra Intelligence and Spectra Analyze

Wait for an unambiguous answer.

**If they choose Both:** run Steps 1–6 in full once for Spectra Intelligence,
then run Steps 1–6 again in full for Spectra Analyze. Do not interleave the two —
complete one service end to end before starting the other, and report each
outcome separately.

Once the service for the current pass is known, use the matching row from this
table for every `<placeholder>` in the steps below:

| Placeholder      | Spectra Intelligence            | Spectra Analyze                       |
|------------------|---------------------------------|---------------------------------------|
| `<venv>`         | `~/.rl-spectra-intel-venv`      | `~/.rl-spectra-analyze-venv`          |
| `<cli>`          | `rl-spectra-intel`              | `rl-spectra-analyze`              |
| `<wheel_glob>`   | `rl_spectra_intel-*.whl`        | `rl_spectra_analyze-*.whl`            |
| `<connect_ask>`  | "Set up my ReversingLabs credentials" | "Set up my Spectra Analyze appliance" |

## Step 1 — Check if already installed; upgrade if so

```bash
<venv>/bin/<cli> --version
```

- If exit code **0**: already installed. Search for the bundled wheel in the plugin cache:
  ```bash
  find ~/.claude/plugins/cache/rl-spectra-ai-soc ~/.claude/plugins/marketplaces/rl-spectra-ai-soc/plugins -name "<wheel_glob>" 2>/dev/null | head -1
  ```
  - If a wheel is found, upgrade from it:
    ```bash
    <venv>/bin/pip install --upgrade <wheel_path>
    ```
  - If no wheel is found: tell the user the bundled wheel could not be located in
    the plugin cache and stop.
  Then verify and report the resulting version:
  ```bash
  <venv>/bin/<cli> --version
  ```
  Tell the user whether the package was upgraded or already at the latest version, and stop.
- If exit code non-zero or command not found: proceed to installation.

## Step 2 — Verify Python version

```bash
python3 --version
```

Parse the version from the output (e.g. `Python 3.10.2`). The CLIs require **Python 3.10 or later**.

- If the version is 3.10 or higher: proceed.
- If the version is lower than 3.10, or `python3` is not found: stop and tell the user:
  > This CLI requires Python 3.10 or later. Found: `<version or "not found">`.
  > Please upgrade Python before continuing.

## Steps 3 and 4 — Create venv and install

These two steps MUST be executed in order using the EXACT commands below.
Do NOT use `pip`, `pip3`, `pipx`, `--user`, or `--break-system-packages`.
Do NOT skip Step 3. Do NOT ask the user which approach to use.

**Step 3 — create the venv:**

```bash
python3 -m venv <venv>
```

If exit code is non-zero: report the error and stop.

**Step 4 — locate the bundled wheel and install:**

Search for the bundled wheel in the plugin cache:

```bash
find ~/.claude/plugins/cache/rl-spectra-ai-soc ~/.claude/plugins/marketplaces/rl-spectra-ai-soc/plugins -name "<wheel_glob>" 2>/dev/null | head -1
```

- If a wheel is found, install from it:
  ```bash
  <venv>/bin/pip install <wheel_path>
  ```
- If no wheel is found: tell the user the bundled wheel could not be located in
  the plugin cache and stop. Do not attempt any other installation method.

If exit code is non-zero: show the full output to the user. The wheel file may
be missing or corrupt — the user should reinstall the plugin.

## Step 5 — Verify venv installation

```bash
<venv>/bin/<cli> --version
```

- If exit code **0**: report the installed version. Proceed to Step 6.
- If exit code non-zero: report the failure — the package may not have installed correctly.

## Step 6 — Report outcome

Tell the user:
- Which service was installed and the installed version of `<cli>`
- That it is installed in `<venv>`
- That they can configure it by asking: **"<connect_ask>"** (this will invoke the `rl-soc-connect` skill)
- That once configured, `rl-soc-connect` points the canonical `rl-soc-cli` command at this endpoint and the `rl-soc` plugin runs against it. If both Spectra Intelligence and Spectra Analyze are installed and configured, `rl-soc-cli` points at whichever was made active last in `rl-soc-connect`; to switch, re-run `rl-soc-connect` and choose the other service.
