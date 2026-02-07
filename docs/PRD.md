# PRD: openclaw-1password

## Problem

OpenClaw agents need API keys and secrets (Anthropic, OpenAI, etc.) to function. Today, these credentials are stored as **plaintext** in configuration files:

- `openclaw.json` (e.g., `memorySearch.remote.apiKey`)
- `agents/main/agent/auth-profiles.json` (provider tokens)
- `.env` files or environment variables

This creates several risks:

1. **Leakage into logs**: Agent session logs (`.jsonl`) capture tool calls and responses. Secrets passed through config end up serialized into these logs in plaintext.
2. **No rotation**: Changing a key means editing config files manually across every machine.
3. **No audit trail**: No record of which agent accessed which secret, when.
4. **No access control**: Any process running as the user can read the config files.

## Solution

An OpenClaw plugin (`@openclaw/1password`) that integrates with **1Password** via the official JavaScript SDK. Agents retrieve secrets at runtime from a 1Password vault using a service account — fully headless, no desktop app or biometric prompts required.

## Target Users

- OpenClaw users who manage API keys for multiple providers
- Teams running OpenClaw agents on servers, CI/CD, or unattended machines
- Anyone who wants to stop storing plaintext secrets in config files

## Requirements

### Must Have (P0)

1. **Agent tool: `op_read_secret`** — Reads a single secret value from 1Password given vault, item, and field names. Returns the secret to the agent in memory (never written to disk).

2. **Agent tool: `op_list_items`** — Lists items in a vault so agents can discover available secrets.

3. **Service account auth** — Uses a 1Password service account token for headless operation. Token stored in a local file with `600` permissions.

4. **CLI commands** — `openclaw op-secrets test` (verify connectivity), `openclaw op-secrets read <item>` (read with redacted preview for debugging).

5. **Skill file** — `SKILL.md` that teaches agents what tools are available, what secrets exist, and the guardrails (never log secret values).

6. **npm-publishable** — Installable via `openclaw plugins install @openclaw/1password`.

### Should Have (P1)

7. **Configurable token path** — Token file location configurable via plugin config or `OP_SA_TOKEN_PATH` environment variable.

8. **Configurable default vault** — Default vault name set in plugin config, overridable per tool call.

9. **Lazy SDK client** — The 1Password SDK client is initialized on first use, not at plugin load time. Avoids blocking startup if the token is missing.

10. **Graceful degradation** — If the token file is missing or invalid, tools return clear error messages instead of crashing the gateway.

### Nice to Have (P2)

11. **Secret reference resolution** — A `resolveSecretRefs` utility that scans a config object for `op://` URIs and resolves them. This would enable `openclaw.json` to contain `"apiKey": "op://Vault/Item/field"` instead of plaintext keys. (Future: propose as OpenClaw core feature.)

12. **Multiple vault support** — Service account with access to multiple vaults.

13. **Write support** — `op_create_secret` tool for agents to store new credentials.

## Architecture

```
1Password Cloud
    ^
    |  HTTPS (SDK)
    |
@1password/sdk client  (singleton, lazy-init)
    ^
    |  reads token from file
    |
~/.openclaw/secrets/op-sa-token  (chmod 600)
    |
openclaw-1password plugin
    |
    +-- op_read_secret tool    (agent-facing)
    +-- op_list_items tool     (agent-facing)
    +-- openclaw op-secrets    (CLI for human debugging)
    +-- SKILL.md               (agent knowledge)
```

### 1Password Setup (User Responsibility)

1. 1Password **Business** or **Teams** plan (required for service accounts)
2. A **custom vault** (e.g., "Agent Secrets") — service accounts cannot access built-in Shared/Employee vaults
3. A **service account** with `read_items` access to that vault
4. Service account token saved to `~/.openclaw/secrets/op-sa-token`

### Plugin Lifecycle

1. **Load**: OpenClaw discovers plugin via `openclaw.plugin.json` manifest
2. **Register**: Plugin registers tools and CLI commands via `api.registerTool()` and `api.registerCli()`
3. **First use**: On first `op_read_secret` call, reads token from disk, creates SDK client
4. **Resolve**: Calls `client.secrets.resolve("op://<vault>/<item>/<field>")`
5. **Return**: Secret value returned to agent in tool response (in memory only)

## Non-Goals

- **Desktop app integration**: We explicitly avoid the `op` CLI and desktop app. The whole point is headless operation.
- **Token generation**: The plugin does not create service accounts. That's a one-time admin setup.
- **Secret rotation**: Out of scope. Users rotate secrets in 1Password; the plugin always reads the current value.
- **Caching**: Secrets are fetched fresh on every call. 1Password SDK handles connection pooling internally.

## Success Criteria

1. `openclaw plugins install @openclaw/1password` works
2. `openclaw op-secrets test` shows vault connectivity
3. Agent calls `op_read_secret` and receives the correct value
4. No secret values appear in session logs, config files, or CLI output
5. Plugin works after reboot with no human intervention
6. Plugin loads in <500ms, secret resolution in <2s

## Dependencies

- `@1password/sdk` (^0.3.x) — 1Password JavaScript SDK
- Node.js 18+ (SDK requirement)
- OpenClaw plugin SDK (`openclaw/plugin-sdk`)
