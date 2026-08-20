# Credential storage

How Claude Code stores sensitive plugin `userConfig` values per OS, and why the
installer skill picks different variants on different platforms.

## Where sensitive values live

| OS | Storage | Protection |
| --- | --- | --- |
| macOS | Encrypted Keychain | OS-level encryption, per-app access control |
| Linux | `~/.claude/.credentials.json` | File mode 0600 — plaintext |
| Windows | `%USERPROFILE%\.claude\.credentials.json` | User-profile ACLs — plaintext (see note below) |

On Linux and Windows, `CLAUDE_CONFIG_DIR` relocates the credentials file.

> **Windows note (unverified).** The plugins reference says sensitive values go
> to the macOS Keychain or to `.credentials.json` where no supported keychain
> exists; some third-party writeups claim Windows Credential Manager is used
> instead. The official authentication page documents a file. We have not
> verified Windows behaviour ourselves. To check on your machine: enable a
> plugin with a sensitive `userConfig` field, then look for
> `%USERPROFILE%\.claude\.credentials.json`.

## Why Variants B and D exist

On macOS, a sensitive `userConfig` value ends up in the encrypted Keychain, so
letting Claude Code manage it (Variants A and C) is safe. On Linux and Windows
the same value lands in a plaintext file readable by any process running as
your user. Variants B (`headersHelper`) and D (wrapper script) keep the secret
in an external store you already trust — `gopass`, `pass`, `sops`, a hardware
token — and hand it to the server only at connection time.

## Why secrets go in `env`, never in `args`

A process's argument vector is world-readable: `ps -ef` shows it to every user
on the machine. Its environment is not — on Linux, `/proc/<pid>/environ` is
readable only by the process owner. So `--api-key YOUR_KEY` in a stdio server's
`args` leaks the key to everyone; the same key in `env` does not.

This is the single most common mistake in published MCP setup snippets, many of
which show `--api-key YOUR_KEY` on the command line. Most servers also read an
environment variable — use it.

## A failed variant escalates, it never degrades

If a chosen variant stops working partway through — the keychain write fails,
the secret-store command is missing, the server ignores the environment
variable — the skill must not silently fall back to writing the secret into a
file. That is Variant E, and it requires an explicit, informed confirmation
from the user first: what would be written, into which file, and who could read
it. On confirmation the file is gitignored in the same step and the plugin is
declared unshareable.

## Keychain budget

Keychain storage for sensitive `userConfig` values is shared with Claude Code's
own OAuth tokens and is limited to roughly 2 KB in total. Keep sensitive values
small: store a token, not a JSON blob.

## `apiKeyHelper` — the same idea for the Anthropic API key

Claude Code's own API key has an equivalent escape hatch: `apiKeyHelper` in
settings runs a script that returns the key, so it too can live in an external
store. The helper is re-invoked after 5 minutes or on an HTTP 401;
`CLAUDE_CODE_API_KEY_HELPER_TTL_MS` overrides the interval. A helper that takes
longer than 10 seconds triggers a warning — relevant if yours requires a
hardware-token touch.

## Threat model

The threat model here is same-user local exposure, not privilege escalation. A
root-level attacker reads your keychain-adjacent memory and your keystrokes;
nothing in this repo defends against that. What these variants reduce is the
blast radius of things that run as *you* without being you: infostealers,
malicious npm postinstall scripts, and hostile editor extensions, all of which
can trivially read a plaintext config file but cannot read the macOS Keychain
entry of another app or your hardware-backed secret store.
