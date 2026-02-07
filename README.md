# @openclaw/1password

OpenClaw plugin for reading secrets from 1Password. Uses the official 1Password JavaScript SDK with service accounts for fully headless operation — no desktop app, no biometrics, no popups.

## Why

OpenClaw agents need API keys to talk to providers (Anthropic, OpenAI, etc.). By default, these keys sit in plaintext in config files, which leak into session logs and are impossible to rotate cleanly. This plugin lets agents pull secrets from 1Password at runtime instead.

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

Both tools are registered as **optional** and must be allowlisted in agent config to use.

## CLI Commands

```bash
# Verify 1Password connectivity
openclaw op-secrets test

# Read a secret (shows redacted preview, safe for terminals)
openclaw op-secrets read "OpenAI API"
openclaw op-secrets read "Anthropic Auth Token" --vault "Agent Secrets"
openclaw op-secrets read "DB Password" --field "password"
```

## Configuration

Plugin config in `openclaw.json` under `plugins.entries.1password.config`:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `defaultVault` | string | `"Agent Secrets"` | Default vault for tool calls |
| `tokenPath` | string | `~/.openclaw/secrets/op-sa-token` | Path to service account token file |

The token path can also be set via the `OP_SA_TOKEN_PATH` environment variable.

## How It Works

1. On first secret request, the plugin reads the service account token from disk
2. It creates a 1Password SDK client (cached for the session lifetime)
3. Secrets are resolved via `client.secrets.resolve("op://Vault/Item/field")`
4. The SDK talks directly to 1Password's servers over HTTPS
5. Secret values are returned in memory only — never written to disk or logs

The service account token does not expire (unless you set `--expires-in`). It survives reboots. No desktop app needed.

## Security

- Service account tokens grant **read-only** access to specific vaults
- The token file is `chmod 600` — only the owning user can read it
- Secret values are never logged, cached, or written to disk by the plugin
- The SKILL.md file instructs agents to never echo or log returned secrets
- If a secret is exposed, rotate it in 1Password — the plugin always reads the current value

## Troubleshooting

**"Token file not found"** — Make sure `~/.openclaw/secrets/op-sa-token` exists and contains your `ops_...` token.

**"Vault not found"** — Service accounts can only access custom vaults, not built-in ones (Shared, Employee, Private). Create a custom vault and grant access.

**"403 Forbidden" when creating service account** — Enable "Developer" / "Secrets Automation" in your 1Password admin console under Settings.

**"Granting a service account access to the Team/Shared vault is not supported"** — Use a custom vault name, not the built-in "Shared" vault.

## License

MIT
