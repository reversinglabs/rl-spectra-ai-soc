---
name: rl-soc-connect
description: >
  Configures the rl-spectra-intel CLI with ReversingLabs credentials. Trigger
  only when the analyst explicitly asks to set up or update credentials, provide
  an API username and password, or configure a custom host. Never invoke this
  skill automatically in response to CLI errors — authentication failures in
  analysis agents must be reported to the analyst, not silently fixed.
---

# rl-soc-connect

Collects ReversingLabs credentials from the user, saves them to
`~/.rl-spectra-intel-venv/.env`, and creates a wrapper script at
`~/.local/bin/rl-spectra-intel` that sources the credentials file before
every invocation — so credentials are injected automatically without
modifying the system environment.

## Step 1 — Check venv is installed

```bash
~/.rl-spectra-intel-venv/bin/rl-spectra-intel --version
```

If exit code is non-zero, stop and tell the user to run the `rl-soc-install`
skill first.

## Step 2 — Collect credentials

Ask the user for the following, one at a time:

1. **rl_username** (required) — their ReversingLabs Spectra Intelligence username
2. **rl_password** (required) — their ReversingLabs Spectra Intelligence password
3. **rl_host** (optional) — custom host name; leave blank to use the default

Do not proceed until both required values are provided.

## Step 3 — Warn about plain-text storage

Before writing anything, tell the user:

> ⚠️ **Credentials will be saved in plain text** to `~/.rl-spectra-intel-venv/.env`.
> Anyone with read access to your home directory can view them.
>
> Do you want to continue? (yes / no)

Wait for the user's response. If they say anything other than an unambiguous
confirmation (yes, y, ok, sure, etc.), **stop** — do not proceed.

## Step 4 — Write the credentials file

**Follow these exact commands. Do NOT use any other approach.**

Write credentials to `~/.rl-spectra-intel-venv/.env`. Build the content based on
whether `rl_host` was provided.

**Without custom host:**
```bash
printf 'REVERSINGLABS_USERNAME=%s\nREVERSINGLABS_PASSWORD=%s\nMCP_OAUTH_ENABLED=False\n' '<username>' '<password>' > ~/.rl-spectra-intel-venv/.env
```

**With custom host:**
```bash
printf 'REVERSINGLABS_USERNAME=%s\nREVERSINGLABS_PASSWORD=%s\nREVERSINGLABS_HOST=%s\nMCP_OAUTH_ENABLED=False\n' '<username>' '<password>' '<host>' > ~/.rl-spectra-intel-venv/.env
```

Then restrict permissions so only the current user can read it:

```bash
chmod 600 ~/.rl-spectra-intel-venv/.env
```

## Step 5 — Create the wrapper script

**Do NOT use `/usr/local/bin`, do NOT create a symlink, do NOT use `sudo`.
Follow these exact three commands and no others.**

```bash
mkdir -p ~/.local/bin
```

```bash
printf '#!/bin/sh\nset -a\n. ~/.rl-spectra-intel-venv/.env\nset +a\nexec ~/.rl-spectra-intel-venv/bin/rl-spectra-intel "$@"\n' > ~/.local/bin/rl-spectra-intel
```

```bash
chmod +x ~/.local/bin/rl-spectra-intel
```

## Step 6 — Verify PATH and wrapper

```bash
rl-spectra-intel --version
```

- If exit code **0**: the wrapper is working and credentials are injected. Proceed to Step 7.
- If exit code non-zero: `~/.local/bin` is not on PATH. Run these two commands automatically — do NOT ask the user to run them:

  ```bash
  grep -qxF 'export PATH="$HOME/.local/bin:$PATH"' ~/.zshrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
  ```

  ```bash
  ~/.local/bin/rl-spectra-intel --version
  ```

  Use the full path `~/.local/bin/rl-spectra-intel` for this verification because `source ~/.zshrc` does not affect the current shell session. If this exits 0, proceed to Step 7 and tell the user: `~/.local/bin` has been added to `~/.zshrc`. To use `rl-spectra-intel` without the full path, close Claude Code, open a new terminal, and relaunch it — Claude Code inherits PATH from the shell that started it, so the change does not take effect until it is restarted. If it exits non-zero, report the error.

## Step 7 — Report outcome

Tell the user:
- Configuration is complete
- Credentials are stored in `~/.rl-spectra-intel-venv/.env` (readable only by the current user)
- The wrapper at `~/.local/bin/rl-spectra-intel` injects credentials automatically on every call
- They can now use the `rl-soc` plugin to run threat analysis
