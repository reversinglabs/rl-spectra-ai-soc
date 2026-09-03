---
name: rl-soc-connect
description: >
  Configures a ReversingLabs SOC CLI with credentials. Prompts for which service
  to configure — Spectra Intelligence (username/password) or Spectra Analyze
  (appliance URL + API token) — then writes the matching config and a PATH
  wrapper. Trigger only when the analyst explicitly asks to set up or update
  credentials or configure a host/appliance. Never invoke this skill
  automatically in response to CLI errors — authentication failures in analysis
  agents must be reported to the analyst, not silently fixed.
---

# rl-soc-connect

Collects connection details from the user, saves them next to the installed
CLI, and creates a PATH wrapper so credentials/appliance settings are applied
automatically on every invocation. It also points the canonical `rl-soc-cli`
symlink at the configured endpoint's wrapper — this is the single command the
`rl-soc` pipeline calls, regardless of which service is active.

## Step 0 — Ask which service to configure

Ask the user:

> Which ReversingLabs service do you want to configure?
>
> 1. **Spectra Intelligence** — cloud threat intelligence
> 2. **Spectra Analyze** — on-prem appliance
> 3. **Both** — configure Spectra Intelligence and Spectra Analyze

Wait for an unambiguous answer, then follow the matching path:
- **Spectra Intelligence** → Path A
- **Spectra Analyze** → Path B
- **Both** → run Path A in full first, then run Path B in full. Complete one
  path end to end before starting the other, and report each outcome separately.
  Then run the **Both — choose the active endpoint** section to decide which one
  `rl-soc-cli` should point at.

---

# Path A — Spectra Intelligence

Writes the endpoint settings to `~/.rl-spectra-intel-venv/.env` and a wrapper at
`~/.local/bin/rl-spectra-intel` that sources the credentials before every call.

## A1 — Check the CLI is installed

```bash
~/.rl-spectra-intel-venv/bin/rl-spectra-intel --version
```

If exit code is non-zero, stop and tell the user to run the `rl-soc-install`
skill first (choosing Spectra Intelligence).

## A2 — Collect the endpoint host

Ask the user:

1. **rl_host** (optional) — custom host name; leave blank to use the default

**Do not ask for a username or password here.** In step A8 the analyst runs
`rl-spectra-intel --login` to collect username and password outside the shell

## A3 — Warn about plain-text storage

> ⚠️ **Your username and password will be saved in plain text** to
> `~/.rl-spectra-intel-venv/.env` when you run `--login`.
> Anyone with read access to your home directory can view them.
>
> Do you want to continue? (yes / no)

Wait for an unambiguous confirmation. Otherwise **stop**.

## A4 — Write the config file

This writes the endpoint settings but leaves the credentials empty — `--login`
fills them in at step A8.

**Without custom host:**
```bash
printf 'MCP_OAUTH_ENABLED=False\n' > ~/.rl-spectra-intel-venv/.env
```

**With custom host:**
```bash
printf 'REVERSINGLABS_HOST=%s\nMCP_OAUTH_ENABLED=False\n' '<host>' > ~/.rl-spectra-intel-venv/.env
```

```bash
chmod 600 ~/.rl-spectra-intel-venv/.env
```

## A5 — Create the wrapper script

The wrapper sources the `.env` before every call and bakes in `--env-file` so
`rl-spectra-intel --login` writes to the same file the wrapper reads
(`--env-file` is ignored on normal tool calls).

```bash
mkdir -p ~/.local/bin
```

```bash
printf '#!/bin/sh\nset -a\n. ~/.rl-spectra-intel-venv/.env\nset +a\nexec ~/.rl-spectra-intel-venv/bin/rl-spectra-intel --env-file ~/.rl-spectra-intel-venv/.env "$@"\n' > ~/.local/bin/rl-spectra-intel
```

```bash
chmod +x ~/.local/bin/rl-spectra-intel
```

## A6 — Point the canonical `rl-soc-cli` symlink at this wrapper

The `rl-soc` pipeline always calls a single canonical command, `rl-soc-cli`,
rather than the per-service wrapper name. Point it at the Spectra Intelligence
wrapper so this becomes the active endpoint:

```bash
ln -sfn ~/.local/bin/rl-spectra-intel ~/.local/bin/rl-soc-cli
```

`ln -sfn` atomically repoints the link if it already exists, so this is also how
the active endpoint is switched later. 

## A7 — Verify PATH and wrapper

```bash
rl-spectra-intel --version
```

If exit code non-zero, run the **Finalize PATH** section below with
`<cli>` = `rl-spectra-intel`, then continue.

## A8 — Log in with username and password (analyst runs this)

`--login` prompts for the analyst's Spectra Intelligence **username and
password** and saves them to the `.env` the wrapper already sources
(`~/.rl-spectra-intel-venv/.env`, chmod 600).

**`--login` requires a real interactive terminal.** Claude Code's command
execution (including the `!` prefix) is not a TTY, so you (the agent) **must
not** run it. Instead, tell the analyst:

> Open a normal terminal (not Claude Code) and run:
>
> ```
> rl-spectra-intel --login
> ```
>
> Enter your Spectra Intelligence username and password at the prompts (the
> password is hidden). On success your credentials are saved. Then come back here.

Wait for the analyst to confirm they have completed the login before continuing.

## A9 — Verify the connection

```bash
rl-spectra-intel get_sample_overview --args '{"hash_value": "44d88612fea8a8f36de82e1278abb02f"}'
```

- Exit **0** with JSON: credentials work. Report outcome.
- Exit **2** with `REVERSINGLABS_USERNAME and REVERSINGLABS_PASSWORD must be set`: the analyst has not completed A8 — send them back to run `rl-spectra-intel --login`.
- Exit **1** with HTTP 401/403: username or password rejected — the analyst should re-run `rl-spectra-intel --login` with correct credentials.

Report: configuration complete; credentials in `~/.rl-spectra-intel-venv/.env`
(readable only by the current user); the wrapper injects credentials on every
call; `rl-soc-cli` now points at Spectra Intelligence, so the `rl-soc` plugin
will run against it. 

---

# Path B — Spectra Analyze

Writes appliance settings to `~/.rl-spectra-analyze-venv/config.ini` and a
wrapper at `~/.local/bin/rl-spectra-analyze` that points the CLI at that
config on every call.

## B1 — Check the CLI is installed

```bash
~/.rl-spectra-analyze-venv/bin/rl-spectra-analyze --version
```

If exit code is non-zero, stop and tell the user to run the `rl-soc-install`
skill first (choosing Spectra Analyze).

## B2 — Collect appliance details

Ask the user, one at a time:

1. **base_url** (required) — the appliance URL, e.g. `https://analyze.example.com/`
2. **verify_ssl** (optional) — verify the appliance TLS certificate.

Do not proceed until `base_url` is provided.

Present `verify_ssl` **impartially** — state what each choice means and let the
analyst decide. Do **not** recommend, default to, or nudge toward either value;
they know their appliance's certificate setup and you do not. Describe the two
options only:
- `true` — the CLI verifies the appliance certificate against the system trust store.
- `false` — the CLI skips certificate verification.

**Do not ask for an API token or credentials here.** In step B8 the analyst runs
`rl-spectra-analyze --login`, which prompts for their Spectra Analyze
**username and password**, exchanges them for an API token via the appliance
Authentication API, and saves only the token. The username and password are used
solely to fetch the token and are never stored, and never pass through the chat,
shell history, or a command argument.

## B3 — Warn about plain-text storage

> ⚠️ **The API token will be saved in plain text** to `~/.rl-spectra-analyze-venv/config.ini`.
> Anyone with read access to your home directory can view it.
>
> Do you want to continue? (yes / no)

Wait for an unambiguous confirmation. Otherwise **stop**.

## B4 — Write the config file

Substitute the collected values. `<verify_ssl>` must be `true` or `false`
(lowercase). This writes the appliance host and enables legacy token auth but
leaves the token empty — `--login` fills it in at step B7.

```bash
printf '[analyze_host]\nbase_url = %s\nverify_ssl = %s\n\n[authentication]\nenable_legacy_auth = true\nenable_oauth_auth = false\n' '<base_url>' '<verify_ssl>' > ~/.rl-spectra-analyze-venv/config.ini
```

```bash
chmod 600 ~/.rl-spectra-analyze-venv/config.ini
```

## B5 — Create the wrapper script

```bash
mkdir -p ~/.local/bin
```

```bash
printf '#!/bin/sh\nexec ~/.rl-spectra-analyze-venv/bin/rl-spectra-analyze --config ~/.rl-spectra-analyze-venv/config.ini "$@"\n' > ~/.local/bin/rl-spectra-analyze
```

```bash
chmod +x ~/.local/bin/rl-spectra-analyze
```

## B6 — Point the canonical `rl-soc-cli` symlink at this wrapper

The `rl-soc` pipeline always calls a single canonical command, `rl-soc-cli`,
rather than the per-service wrapper name. Point it at the Spectra Analyze wrapper
so this becomes the active endpoint:

```bash
ln -sfn ~/.local/bin/rl-spectra-analyze ~/.local/bin/rl-soc-cli
```

`ln -sfn` atomically repoints the link if it already exists, so this is also how
the active endpoint is switched later. If you are running the **Both** path, the
active endpoint is decided once at the end (see the *Both — choose the active
endpoint* section); running this command here is still correct.

## B7 — Verify PATH and wrapper

```bash
rl-spectra-analyze --version
```

If exit code non-zero, run the **Finalize PATH** section below with
`<cli>` = `rl-spectra-analyze`, then continue.

## B8 — Log in with username and password (analyst runs this)

`--login` prompts for the analyst's Spectra Analyze **username and password**,
exchanges them for an API token via the appliance Authentication API, and saves
the token to the config the wrapper already points at
(`~/.rl-spectra-analyze-venv/config.ini`). The username and password are used
only to obtain the token and are never stored; only the token is written to
disk.

**`--login` requires a real interactive terminal.** Claude Code's command
execution (including the `!` prefix) is not a TTY, so you (the agent) **must
not** run it. Instead, tell the analyst:

> Open a normal terminal (not Claude Code) and run:
>
> ```
> rl-spectra-analyze --login
> ```
>
> Enter your Spectra Analyze username and password at the prompts (the password
> is hidden). On success an API token is fetched and saved. Then come back here.

Wait for the analyst to confirm they have completed the login before continuing.

If the analyst reports `--login requires an interactive terminal`, they ran it
inside Claude Code — have them run it in a separate terminal window instead.

## B9 — Verify the appliance connection

```bash
rl-spectra-analyze get_server_info --args '{}'
```

- Exit **0** with JSON: connection works. Report outcome.
- Exit **2** with `no appliance API token found`: the analyst has not completed B8 — send them back to run `rl-spectra-analyze --login`.
- Exit **1** with HTTP 401/403: token invalid or expired — the analyst should re-run `rl-spectra-analyze --login` with a fresh token.
- TLS error: the appliance certificate is not trusted — confirm whether `verify_ssl` should be `false`, or install a trusted certificate.

Report: configuration complete; appliance URL in
`~/.rl-spectra-analyze-venv/config.ini` and the API token saved there by
`--login` (file readable only by the current user); the wrapper applies the
config on every call; `rl-soc-cli` now points at Spectra Analyze, so the `rl-soc`
plugin will run against this appliance. They can also run Spectra Analyze tools
directly (e.g. `rl-soc-cli --list-tools`).

---

# Both — choose the active endpoint

Run this only when the analyst chose **Both** in Step 0, after Path A and Path B
have both completed. Because `rl-soc-cli` is a single symlink, only one endpoint
can be active at a time. Whichever path ran last already left `rl-soc-cli`
pointing at its wrapper — confirm the analyst's preference and set it explicitly.

Ask:

> Both endpoints are configured. Which should `rl-soc` use by default?
>
> 1. **Spectra Intelligence** — cloud threat intelligence
> 2. **Spectra Analyze** — on-prem appliance

Point the symlink at the chosen wrapper:

- **Spectra Intelligence:**
  ```bash
  ln -sfn ~/.local/bin/rl-spectra-intel ~/.local/bin/rl-soc-cli
  ```
- **Spectra Analyze:**
  ```bash
  ln -sfn ~/.local/bin/rl-spectra-analyze ~/.local/bin/rl-soc-cli
  ```

Confirm the active target:

```bash
readlink ~/.local/bin/rl-soc-cli
```

Tell the analyst which endpoint is now active and that they can switch at any
time by re-running this skill and choosing the other service.

---

# Finalize PATH (shared)

Run this only when a wrapper verification exits non-zero — it means
`~/.local/bin` is not on PATH. Run these commands automatically; do NOT ask the
user to run them.

```bash
case "$(basename "$SHELL")" in
  zsh)  SHELL_RC="$HOME/.zshrc" ;;
  bash) SHELL_RC="$HOME/.bashrc" ;;
  *)    SHELL_RC="$HOME/.profile" ;;
esac
grep -qxF 'export PATH="$HOME/.local/bin:$PATH"' "$SHELL_RC" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$SHELL_RC"
```

```bash
~/.local/bin/<cli> --version
```

Use the full path `~/.local/bin/<cli>` here because sourcing the rc file does not
affect the current shell session. If this exits 0, tell the user: `~/.local/bin`
has been added to their shell rc file. To use `<cli>` without the full path, close
Claude Code, open a new terminal, and relaunch it — Claude Code inherits PATH from
the shell that started it, so the change does not take effect until it is restarted.
If it exits non-zero, report the error.
