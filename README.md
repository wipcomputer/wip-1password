# @openclaw/1password

OpenClaw plugin for 1Password secrets. Uses the official JavaScript SDK with service accounts for fully headless operation — no desktop app, no biometrics, no popups.

## What It Does

1. **Resolves `op://` references in `openclaw.json` at startup** — Replace plaintext API keys with `op://Agent Secrets/Item/field` references. The plugin resolves them to real values in memory when OpenClaw boots. The plaintext key never touches disk.

2. **Gives agents tools to read and write secrets on demand** — Agents call `op_read_secret` to pull a key from 1Password at runtime, `op_list_items` to discover what's available, or `op_write_secret` to store new credentials.

3. **CLI for diagnostics** — `openclaw op-secrets test` verifies 1Password connectivity. `openclaw op-secrets read` shows a redacted preview of any secret.

## Limitations

- **`auth-profiles.json` cannot use `op://` refs.** This file is loaded by a separate OpenClaw subsystem that the plugin can't hook into. Auth provider tokens (Anthropic, OpenAI OAuth) must remain as real values in that file. The resolver service only handles `openclaw.json` (the main config passed to plugins via `ctx.config`).

- **`op_write_secret` requires `write_items` permission** on the service account. The default setup uses `read_items` only. See [Write Support](#write-support) for setup.

- **Secrets are resolved in memory only.** If OpenClaw reloads config from disk mid-session, `op://` refs need to be resolved again. The service runs at startup, so this is fine for normal operation.

## Prerequisites

- **1Password Business or Teams plan** (required for service accounts)
- **Node.js 18+**
- **OpenClaw** 2026.1+

## Quick Start

### 1. Install the plugin

```bash
openclaw plugins install @openclaw/1password
```

### 2. Set up 1Password

Create a custom vault and service account (service accounts can't access built-in vaults):

```bash
# Create a vault for agent secrets
op vault create "Agent Secrets"

# Add your API keys
op item create --category "API Credential" --title "OpenAI API" \
  --vault "Agent Secrets" 'api key=YOUR_KEY_HERE'

# Create a read-only service account
op service-account create "OpenClaw Agent" \
  --expires-in 0 \
  --vault "Agent Secrets:read_items"
```

### 3. Save the service account token

```bash
mkdir -p ~/.openclaw/secrets
# Save the ops_... token from step 2 into this file:
echo "ops_..." > ~/.openclaw/secrets/op-sa-token
chmod 600 ~/.openclaw/secrets/op-sa-token
```

### 4. Enable the plugin

Add to `~/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "1password": {
        "enabled": true,
        "config": {
          "defaultVault": "Agent Secrets"
        }
      }
    }
  }
}
```

### 5. Test

```bash
openclaw op-secrets test
```

### 6. Replace plaintext keys with `op://` references

In `openclaw.json`, replace any plaintext API keys with `op://` URIs:

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "remote": {
          "apiKey": "op://Agent Secrets/OpenAI API/api key"
        }
      }
    }
  }
}
```

The plugin resolves these at startup. The real key never appears on disk.

**Note:** This only works for `openclaw.json`. Auth profile tokens in `auth-profiles.json` cannot use `op://` refs — see [Limitations](#limitations).

## Agent Tools

### `op_read_secret`

Reads a secret value from 1Password. Agents call this tool when they need an API key or credential.

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `item` | Yes | — | Item title (e.g. "OpenAI API") |
| `vault` | No | Config default | Vault name |
| `field` | No | `"api key"` | Field name |

### `op_list_items`

Lists all items in a vault so agents can discover available secrets.

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `vault` | No | Config default | Vault name |

### `op_write_secret`

Creates or updates a secret in 1Password. Requires `write_items` permission on the service account.

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `item` | Yes | — | Item title (e.g. "New API Key") |
| `value` | Yes | — | The secret value to store |
| `vault` | No | Config default | Vault name |
| `field` | No | `"api key"` | Field name |

All tools are registered as **optional** and must be allowlisted in agent config to use.

## Config Secret Resolution

The plugin registers a startup service that walks `openclaw.json` and resolves any string matching `op://vault/item/field` to its real value via the 1Password SDK. Resolution happens in memory — the file on disk keeps the `op://` reference.

The `resolveSecretRefs` utility is also exported for programmatic use:

```typescript
import { resolveSecretRefs } from "@openclaw/1password";

const resolved = await resolveSecretRefs({
  apiKey: "op://Agent Secrets/OpenAI API/api key",
  other: "not a secret, passed through",
});
// resolved.apiKey === "sk-proj-..."
// resolved.other === "not a secret, passed through"
```

## CLI Commands

```bash
# Verify 1Password connectivity
openclaw op-secrets test

# Read a secret (shows redacted preview, safe for terminals)
openclaw op-secrets read "OpenAI API"
openclaw op-secrets read "Anthropic Auth Token" --vault "Agent Secrets"
openclaw op-secrets read "DB Password" --field "password"

# Dry-run: show which op:// refs would be resolved in a JSON file
openclaw op-secrets resolve ~/.openclaw/openclaw.json
```

## Write Support

To enable write operations (`op_write_secret`), the service account needs `write_items` permission:

```bash
op service-account create "OpenClaw Agent" \
  --expires-in 0 \
  --vault "Agent Secrets:read_items,write_items"
```

Then update the token at `~/.openclaw/secrets/op-sa-token`.

## Configuration

Plugin config in `openclaw.json` under `plugins.entries.1password.config`:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `defaultVault` | string | `"Agent Secrets"` | Default vault for tool calls |
| `tokenPath` | string | `~/.openclaw/secrets/op-sa-token` | Path to service account token file |

The token path can also be set via the `OP_SA_TOKEN_PATH` environment variable.

## How It Works

1. On plugin load, the resolver service walks `openclaw.json` and resolves `op://` references in memory
2. On first secret request, the plugin reads the service account token from disk
3. It creates a 1Password SDK client (cached for the session lifetime)
4. Secrets are resolved via `client.secrets.resolve("op://Vault/Item/field")`
5. The SDK talks directly to 1Password's servers over HTTPS
6. Secret values exist in memory only — never written to disk or logs

The service account token does not expire (unless you set `--expires-in`). It survives reboots. No desktop app needed.

## Security

- Service account tokens grant access only to specific vaults you configure
- The token file is `chmod 600` — only the owning user can read it
- Secret values are never logged, cached, or written to disk by the plugin
- The SKILL.md file instructs agents to never echo or log returned secrets
- If a secret is exposed, rotate it in 1Password — the plugin always reads the current value
- `op://` references in config files are safe to commit to version control

## Troubleshooting

**"Token file not found"** — Make sure `~/.openclaw/secrets/op-sa-token` exists and contains your `ops_...` token.

**"Vault not found"** — Service accounts can only access custom vaults, not built-in ones (Shared, Employee, Private). Create a custom vault and grant access.

**"403 Forbidden" when creating service account** — Enable "Developer" / "Secrets Automation" in your 1Password admin console under Settings.

**"Granting a service account access to the Team/Shared vault is not supported"** — Use a custom vault name, not the built-in "Shared" vault.

**Write operations fail** — The service account needs `write_items` permission. See [Write Support](#write-support).

## License

MIT
