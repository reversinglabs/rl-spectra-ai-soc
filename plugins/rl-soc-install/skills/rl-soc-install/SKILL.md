---
name: rl-soc-install
description: >
  Installs the rl-spectra-intel CLI from the bundled wheel included with this
  plugin. Trigger when the user wants to set up rl-spectra-intel, when
  rl-spectra-intel is not found on PATH, or when rl-soc reports
  that the CLI is missing.
---

# rl-soc-install

Installs the `rl-spectra-intel` CLI into a dedicated Python virtual environment
at `~/.rl-spectra-intel-venv`. The wrapper script and credential injection are
handled by `rl-soc-connect` after installation.

## Step 1 — Check if already installed; upgrade if so

```bash
~/.rl-spectra-intel-venv/bin/rl-spectra-intel --version
```

- If exit code **0**: already installed. Search for a bundled wheel in the plugin cache:
  ```bash
  find ~/.claude/plugins/cache/rl-spectra-ai-soc ~/.claude/plugins/marketplaces/rl-spectra-ai-soc/plugins -name "rl_spectra_intel-*.whl" 2>/dev/null | head -1
  ```
  - If a wheel is found, upgrade from it:
    ```bash
    ~/.rl-spectra-intel-venv/bin/pip install --upgrade <wheel_path>
    ```
  - If no wheel is found: tell the user the bundled wheel could not be located in
    the plugin cache and stop.
  Then verify and report the resulting version:
  ```bash
  ~/.rl-spectra-intel-venv/bin/rl-spectra-intel --version
  ```
  Tell the user whether the package was upgraded or already at the latest version, and stop.
- If exit code non-zero or command not found: proceed to installation.

## Step 2 — Verify Python version

```bash
python3 --version
```

Parse the version from the output (e.g. `Python 3.10.2`). The CLI requires **Python 3.10 or later**.

- If the version is 3.10 or higher: proceed.
- If the version is lower than 3.10, or `python3` is not found: stop and tell the user:
  > `rl-spectra-intel` requires Python 3.10 or later. Found: `<version or "not found">`.
  > Please upgrade Python before continuing.

## Steps 3 and 4 — Create venv and install

These two steps MUST be executed in order using the EXACT commands below.
Do NOT use `pip`, `pip3`, `pipx`, `--user`, or `--break-system-packages`.
Do NOT skip Step 3. Do NOT ask the user which approach to use.

**Step 3 — create the venv:**

```bash
python3 -m venv ~/.rl-spectra-intel-venv
```

If exit code is non-zero: report the error and stop.

**Step 4 — locate the bundled wheel and install:**

Search for the bundled wheel in the plugin cache:

```bash
find ~/.claude/plugins/cache/rl-spectra-ai-soc ~/.claude/plugins/marketplaces/rl-spectra-ai-soc/plugins -name "rl_spectra_intel-*.whl" 2>/dev/null | head -1
```

- If a wheel is found, install from it:
  ```bash
  ~/.rl-spectra-intel-venv/bin/pip install <wheel_path>
  ```
- If no wheel is found: tell the user the bundled wheel could not be located in
  the plugin cache and stop. Do not attempt any other installation method.

If exit code is non-zero: show the full output to the user. The wheel file may
be missing or corrupt — the user should reinstall the plugin.

## Step 5 — Verify venv installation

```bash
~/.rl-spectra-intel-venv/bin/rl-spectra-intel --version
```

- If exit code **0**: report the installed version. Proceed to Step 6.
- If exit code non-zero: report the failure — the package may not have installed correctly.

## Step 6 — Report outcome

Tell the user:
- The installed version of `rl-spectra-intel`
- That it is installed in `~/.rl-spectra-intel-venv`
- That they must **close Claude Code, open a new terminal, and relaunch Claude Code** before configuring credentials
- That once they are in a new session, they can configure credentials by asking: **"Set up my ReversingLabs credentials"** (this will invoke the `rl-soc-connect` skill)
