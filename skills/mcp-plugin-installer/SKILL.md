---
name: mcp-plugin-installer
description: Use when the user wants to install, add, or set up an MCP server (e.g. "install the GitHub MCP server", "add Context7 MCP"). Creates a skills-directory plugin that bundles the server in .mcp.json, with credentials handled by the OS keychain on macOS or by an external secret store elsewhere — never in plaintext config. Do NOT use for authoring a new MCP server from scratch.
---

# MCP server installer (skills-dir plugin)

Install MCP servers — remote HTTP or local stdio — as `<name>@skills-dir`
plugins so configuration lives in version-controllable files under
`~/.claude/skills/`, not in `~/.claude.json`.

## Step 1 — Classify along two axes

**Transport.** Determined by the vendor's documentation (Step 3), not by asking:

- **Remote** — an HTTP/SSE/WS endpoint, authenticated with a request header
- **Local (stdio)** — a command Claude Code spawns, authenticated with an
  environment variable or a CLI flag

**Platform.** Run `uname -s` (Bash):

| Result | Platform | Keychain |
| --- | --- | --- |
| `Darwin` | macOS | available |
| `Linux` | Linux (incl. WSL) | none |
| `MINGW*` / `MSYS*` / `CYGWIN*` | Windows | none |

Combine into a default variant:

| | macOS | Linux / Windows |
| --- | --- | --- |
| **Remote** | **A** — `userConfig` → `headers` | **B** — `headersHelper` script |
| **Local (stdio)** | **C** — `userConfig` → `env` | **D** — wrapper script |

Rationale, to state briefly to the user: on macOS, sensitive `userConfig` values
go to the encrypted Keychain. On Linux and Windows there is no keychain
integration; values land in `~/.claude/.credentials.json` — mode 0600 on Linux,
ACL-protected on Windows, but plaintext either way and readable by any process
running as that user.

### Secrets and `args`

**Never put a secret in `args`.** A process's argument vector is world-readable
via `ps`, so `--api-key <secret>` exposes it to every user on the machine. The
environment does not have this problem.

When a vendor documents only an `--api-key` flag, check whether the server also
reads an environment variable — most do — and prefer that. If it genuinely
accepts the credential only as a flag, that is Variant E.

## Step 2 — Ask the user

Use AskUserQuestion. Ask both questions in a single call. Put the variant from
Step 1 FIRST in the options list, but always offer the alternatives — the user
may be syncing dotfiles across machines and want one consistent approach.

1. "Install location" → `~/.claude/skills` (all projects) / `.claude/skills` in
   this project (loads only here, gated on workspace trust)
2. "Credential handling" → [Step 1 default first] OS keychain via userConfig /
   External secret store via helper script / No authentication

Do not proceed until answered. Never guess.

## Step 3 — Look up the real configuration

NEVER write a server URL or header name from memory. WebFetch the vendor's
official MCP setup page and confirm: endpoint URL, transport type, exact auth
header name, and required token scopes. Cite the source URL to the user.

## Step 4 — Scaffold

For `~/.claude/skills`:

```
claude plugin init <name>-mcp --with mcp
```

Rewrite the generated files. Delete the sample `SKILL.md` unless the vendor
recommends a usage rule worth keeping as a skill.

For project scope, `claude plugin init` has no target-directory option — create
the files by hand instead: `.claude/skills/<name>-mcp/.claude-plugin/plugin.json`
and `.claude/skills/<name>-mcp/.mcp.json`, with the same content as below.

### Variant A — userConfig (macOS)

`.claude-plugin/plugin.json`:

```json
{
  "$schema": "https://www.schemastore.org/claude-code-plugin-manifest.json",
  "name": "<name>-mcp",
  "description": "<one line>",
  "version": "0.1.0",
  "userConfig": {
    "token": {
      "type": "string",
      "title": "<Vendor> token",
      "description": "<where to obtain it, which scopes>",
      "sensitive": true,
      "required": true
    }
  }
}
```

`.mcp.json`:

```json
{
  "mcpServers": {
    "<server-key>": {
      "type": "http",
      "url": "<verified url>",
      "headers": { "<verified header>": "${user_config.token}" }
    }
  }
}
```

The user does NOT pre-create a Keychain entry. Claude Code writes the value to
the Keychain when they fill in the enable-time dialog.

### Variant B — headersHelper (Linux / Windows)

Omit `userConfig` entirely. `.mcp.json`:

```json
{
  "mcpServers": {
    "<server-key>": {
      "type": "http",
      "url": "<verified url>",
      "headersHelper": "${CLAUDE_PLUGIN_ROOT}/bin/auth-header.sh"
    }
  }
}
```

Before writing `bin/auth-header.sh`, WebFetch
https://code.claude.com/docs/en/mcp and confirm the exact output format the
helper must produce. Do not guess it.

Constraints for the helper script:
- `${user_config.*}` is rejected in `headersHelper` — the script must read the
  secret itself.
- Read from the user's existing secret store; ask which command to call
  (e.g. `gopass show -o <path>`, `sops -d`, `pass`).
- Never write the secret to a file, a log, or stdout beyond the header itself.
- `chmod +x` the script.
- `headersHelper` applies to remote HTTP/SSE/WS servers only, not stdio.

Tell the user which secret path the script expects, so they can store it before
first use. Unlike Variant A, the secret must already exist.

### Variant C — userConfig via env (macOS, stdio)

`plugin.json` is identical to Variant A. `.mcp.json` passes the value through
the environment, never through `args`:

```json
{
  "mcpServers": {
    "<server-key>": {
      "command": "npx",
      "args": ["-y", "<package>"],
      "env": { "<VENDOR_ENV_VAR>": "${user_config.token}" }
    }
  }
}
```

Confirm the exact environment variable name in the vendor's documentation. If
the package needs pinning, pin it in `args` — versions are not secrets.

### Variant D — wrapper script (Linux / Windows, stdio)

There is no stdio equivalent of `headersHelper`. Instead, point `command` at a
script that fetches the secret and execs the real server:

```json
{
  "mcpServers": {
    "<server-key>": {
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/run-server.sh"
    }
  }
}
```

`bin/run-server.sh` (replace `VENDOR_ENV_VAR` — in both places — with the
variable name the server actually reads):

```sh
#!/bin/sh
# Fetch the credential from the user's secret store and hand it to the server
# through the environment, never through argv.
set -eu
VENDOR_ENV_VAR="$(gopass show -o <path>)"
export VENDOR_ENV_VAR
# exec keeps the process tree flat so Claude Code's stdio pipes stay attached.
exec npx -y <package>
```

Requirements:
- `exec` is mandatory. Without it the server runs as a child and stdio framing
  breaks.
- `chmod +x` the script.
- Ask the user which secret-store command to call (`gopass show -o`, `sops -d`,
  `pass`, …) rather than assuming one.
- Never echo the secret or write it to a file.
- On Windows outside WSL, write a `.ps1` or `.cmd` equivalent and confirm the
  user's shell first.

The secret must already exist in the store. Tell the user the exact path the
script expects.

### Variant E — the secret cannot be kept out of the file

Reach this only when A–D are all impossible: no keychain, no secret store, and
the server accepts the credential only inline.

**STOP. Do not write the secret.** Raise it with the user via AskUserQuestion,
stating plainly:

- exactly what would be written, and into which file
- who could read it: any process running as this user, plus anyone with access
  to the repository if that file is ever committed
- that this defeats the purpose of the plugin layout

Offer: (a) inline it and gitignore the file, (b) set up a secret store first and
resume, (c) abort.

Only on an explicit (a) may you write it. When you do, add the file to
`.gitignore` in the same step and tell the user the plugin is now unshareable.

**This escalation is not optional and not limited to Variant E.** If a chosen
variant turns out not to work partway through — the keychain write fails, the
secret-store command is missing, the server ignores the environment variable —
do not silently fall back to inlining. Escalate here.

## Step 5 — Validate

```
claude plugin validate <install-dir>/<name>-mcp
```

where `<install-dir>` is the location chosen in Step 2.

## Step 6 — Hand back to the user

Print these and STOP. Never enter, echo, or read the token yourself.

1. Restart the session (or run `/reload-plugins`). Skills-dir plugins enable
   automatically on discovery — no enable-time credential dialog appears. For a
   project-scope install, the workspace must be trusted before the plugin is
   scanned.
2. Provide the credential
   - Variant A/C: run `/plugin configure <name>-mcp@skills-dir` and paste the
     token into the dialog
   - Variant B/D: make sure the secret is in the store at the path above
3. Run `/mcp` to confirm the connection (if the server has not picked up the
   new credential, `/reload-plugins` first)

## Never do

- Write a token, PAT, or API key into any file without an explicit Variant E
  confirmation, or echo one into the shell.
- Put a secret in a stdio server's `args`. Use `env`.
- Silently fall back to inlining a secret when the chosen variant fails.
- Pass a secret via `claude plugin install --config` — it lands in shell history.
- Run `claude mcp add` — it writes to `~/.claude.json`, defeating the purpose.
- Add a server whose URL, environment variable name, or package name came from a
  blog post rather than vendor documentation.
