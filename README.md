# claude-mcp-plugin-installer

A Claude Code skill that installs remote MCP servers as skills-directory
plugins under `~/.claude/skills/`, instead of via `claude mcp add`.

## The problem

`claude mcp add` writes user-scope MCP servers into `~/.claude.json` — the same
file that holds your OAuth session, per-project trust state, and caches. That
file is machine state, not configuration: it cannot be meaningfully
version-controlled, diffed, or selectively enabled.

## The approach

A folder under `~/.claude/skills/` that contains `.claude-plugin/plugin.json`
loads as a plugin named `<name>@skills-dir` — no marketplace, no install step.
The plugin declares its MCP server in a plain `.mcp.json` that can be
committed, reviewed, and toggled with `claude plugin enable/disable`.

One plugin per server. That granularity means you can disable a token-heavy
server like GitHub MCP on days you are not using it, without touching anything
else.

Credentials never go into the committed files: on macOS they live in the
encrypted Keychain via the plugin's `userConfig` dialog; on Linux and Windows
the skill wires up a helper script that reads your existing secret store. See
[docs/credential-storage.md](docs/credential-storage.md).

## Install

Copy the skill into your personal skills directory and restart Claude Code:

```sh
cp -R skills/mcp-plugin-installer ~/.claude/skills/
```

## Prerequisite

For the skill to write into `~/.claude/skills/` from a session running in some
other project directory, that path must be inside the permission boundary. Add
it to `~/.claude/settings.json`:

```json
{
  "permissions": {
    "additionalDirectories": ["~/.claude/skills"]
  }
}
```

Without this the skill will hit permission prompts (or denials) when
scaffolding the plugin.

## Usage

Say, for example:

> install the GitHub MCP server

The skill classifies the server (remote vs stdio) and your platform, asks two
questions — install location and credential handling — looks up the real
endpoint and header names in the vendor's documentation, scaffolds the plugin,
and validates it. You then restart (or `/reload-plugins`) — skills-dir plugins
enable automatically — and run `/plugin configure <name>-mcp@skills-dir` to
paste the token into the credential dialog. The skill never touches the token
itself.

## Credential handling

| Platform | Remote (HTTP) | Local (stdio) |
| --- | --- | --- |
| macOS | **A** — `userConfig` → header, token in Keychain | **C** — `userConfig` → `env`, token in Keychain |
| Linux / Windows | **B** — `headersHelper` script reads your secret store | **D** — wrapper script reads your secret store |

If none of these can work, the skill stops and asks before writing anything
sensitive to disk — it never inlines a secret silently. Details and threat
model: [docs/credential-storage.md](docs/credential-storage.md).

## Examples

Ready-made plugins in [`examples/`](examples/) — copy one into
`~/.claude/skills/` and enable it:

- [`github-mcp`](examples/github-mcp/) — GitHub remote MCP, Variant A
- [`context7-remote`](examples/context7-remote/) — Context7 remote MCP, Variant A
- [`context7-stdio`](examples/context7-stdio/) — Context7 local stdio MCP,
  Variant C (API key via `CONTEXT7_API_KEY` in `env`)

The examples use `userConfig`, so they are macOS-first; on Linux or Windows the
token would land in a plaintext `~/.claude/.credentials.json` — prefer letting
the skill generate a Variant B/D layout instead.

## Limitations

- Both remote HTTP and local stdio servers are supported, but stdio servers
  bring their own runtime dependency (Node, Python, a container) that this
  skill does not install or manage.
- Variant D wrapper scripts are POSIX shell; Windows outside WSL needs a
  hand-written `.ps1`/`.cmd` equivalent.
- The token entry step is manual by design and will not be automated.
- New plugins are only discovered on the next session (or after
  `/reload-plugins`).
- Variant A and Variant B cannot coexist under the same plugin name, so
  cross-OS dotfiles should standardise on one — in practice Variant B, which
  works everywhere.
- Project-scope installs (`.claude/skills/` inside a project) load only after
  the workspace is trusted, and must be scaffolded by hand — `claude plugin
  init` only targets `~/.claude/skills/`.

## Verify your setup

```sh
claude plugin validate ~/.claude/skills/<name>-mcp   # add --strict in CI
claude plugin details <name>-mcp@skills-dir           # tool count, projected token cost
claude --debug                                        # watch the server connect
```

Then `/mcp` inside a session shows the live connection status.

## License

[MIT](LICENSE)
