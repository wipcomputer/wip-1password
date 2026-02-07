# Handoff: Remaining Tasks for Lesa

This document picks up where the initial build left off. The plugin is built, tested (read path), committed, and the service account is live. Three things remain.

---

## Task 1: Test `op_write_secret` Against Real Vault

The write tool was implemented but never tested live.

### Steps

1. Start with a test write to the existing "Agent Secrets" vault on `wipcomputer.1password.com`:

```bash
# Via CLI (if available), or call the tool directly from an agent session:
openclaw op-secrets test  # verify connectivity first
```

2. Test creating a new item via the agent tool `op_write_secret`:
   - `item`: "Test Secret"
   - `value`: "test-value-12345"
   - `vault`: "Agent Secrets" (default)
   - `field`: "api key" (default)

3. Verify it exists:
```bash
openclaw op-secrets read "Test Secret"
# Expected: Agent Secrets/Test Secret/api key: test-v...[25 chars]...12345
```

4. Test updating the same item:
   - Call `op_write_secret` again with `item`: "Test Secret", `value`: "updated-value-67890"
   - Read again to verify the value changed

5. Clean up — delete "Test Secret" from the vault via 1Password UI or `op item delete`.

### If it fails

The `op_write_secret` tool uses `client.items.create()` and `client.items.put()` from the `@1password/sdk`. The service account currently has `read_items` permission. **Write operations require `write_items` permission.**

To fix:
```bash
# You may need to recreate the service account with write access:
op service-account create "OpenClaw Agent" \
  --expires-in 0 \
  --vault "Agent Secrets:read_items,write_items" \
  --account wipcomputer.1password.com
```

Then update the token at `~/.openclaw/secrets/op-sa-token`.

---

## Task 2: Replace Plaintext Keys with `op://` References

Two files still contain plaintext secrets:

### File 1: `~/.openclaw/openclaw.json`

**Line 44** contains the OpenAI API key in plaintext:
```json
"memorySearch": {
  "provider": "openai",
  "remote": {
    "apiKey": "sk-proj-hTYw...EEwA"
  }
}
```

Replace with:
```json
"memorySearch": {
  "provider": "openai",
  "remote": {
    "apiKey": "op://Agent Secrets/OpenAI API/api key"
  }
}
```

### File 2: `~/.openclaw/agents/main/agent/auth-profiles.json`

Contains three plaintext credentials:

1. **Anthropic auth token** (line 15):
   ```json
   "anthropic:manual": {
     "type": "token",
     "provider": "anthropic",
     "token": "sk-ant-oat01-JYN5...0AAA"
   }
   ```
   Replace `token` value with: `"op://Agent Secrets/Anthropic Auth Token/api key"`

2. **OpenAI OAuth access token** (line 7): A JWT starting with `eyJhbG...`
   - This is an OAuth token that refreshes automatically — it's not a static secret stored in 1Password
   - **Leave this as-is** unless you want to store it in 1Password and manage rotation manually

3. **OpenAI refresh token** (line 8): `rt_7MBCu...`
   - Same — OAuth managed, leave as-is

### After replacing

The `op://` URIs won't resolve automatically yet. OpenClaw's config loader doesn't know about them. That's Task 3.

---

## Task 3: Hook `resolveSecretRefs` into Config Loading

This is the integration step. The plugin exports a `resolveSecretRefs()` function that walks any JS object and resolves `op://vault/item/field` strings to their actual values.

### Approach A: Plugin service (recommended)

Register a service in the plugin that resolves secret refs at startup:

In `src/index.ts`, add a service registration inside the `register()` function:

```typescript
api.registerService({
  id: "op-secret-resolver",
  async start(ctx) {
    // Resolve any op:// references in the current config
    const config = ctx.config;

    // Walk specific config paths that might contain secrets
    if (config.agents?.defaults?.memorySearch?.remote?.apiKey) {
      const ref = config.agents.defaults.memorySearch.remote.apiKey;
      if (typeof ref === "string" && ref.startsWith("op://")) {
        const resolved = await resolveSecret(
          ...parseOpRef(ref)  // you'll need to extract vault/item/field from the URI
        );
        // Patch the config in memory
        config.agents.defaults.memorySearch.remote.apiKey = resolved;
      }
    }

    ctx.logger.info("Secret references resolved");
  },
});
```

### Approach B: Generic resolver (more ambitious)

Use the exported `resolveSecretRefs()` to resolve the entire config:

```typescript
api.registerService({
  id: "op-secret-resolver",
  async start(ctx) {
    const resolved = await resolveSecretRefs(ctx.config);
    Object.assign(ctx.config, resolved);
    ctx.logger.info("All op:// references resolved");
  },
});
```

**Caveat**: This mutates the config object in memory. It works for the current process but doesn't persist. If OpenClaw reloads config from disk, the `op://` URIs would need to be resolved again. The service runs at startup, so this should be fine for most cases.

### Approach C: Propose to OpenClaw core (long-term)

The cleanest solution would be for OpenClaw to support `op://` (or a generic `secret://`) protocol natively in its config loader. The plugin would register as a "secret provider" and the core would call it during config resolution. This is a feature request / PR to OpenClaw, not something the plugin can do alone.

---

## Reference

### Service account details
- **Account**: wipcomputer.1password.com
- **Service account name**: "OpenClaw Agent"
- **Token location**: `~/.openclaw/secrets/op-sa-token` (chmod 600)
- **Vault**: "Agent Secrets" (custom vault, ID: `ywmget6o6aki6wh2e4slzvrr5a`)
- **Permissions**: `read_items` (may need `write_items` for Task 1)

### Secrets in vault
| Item | Field | What it is |
|------|-------|------------|
| OpenAI API | api key | OpenAI project API key (`sk-proj-...`) |
| Anthropic Auth Token | api key | Anthropic auth token (`sk-ant-oat01-...`) |

### Plugin location
- **Repo**: `/Users/lesa/Documents/Projects/Claude Code/openclaw-1password/`
- **Live extension**: `~/.openclaw/extensions/op-secrets/` (older version, can be replaced with repo build)
- **Source**: `src/index.ts`
- **Built output**: `dist/index.js` (run `npm run build` after changes)

### Key function
```typescript
import { resolveSecretRefs } from "@openclaw/1password";

// Resolves any "op://vault/item/field" strings in an object tree
const resolved = await resolveSecretRefs({
  apiKey: "op://Agent Secrets/OpenAI API/api key",
  other: "not a secret, passed through",
});
// resolved.apiKey === "sk-proj-hTYw...EEwA"
// resolved.other === "not a secret, passed through"
```
