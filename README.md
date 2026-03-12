###### WIP Computer

[![Module](https://img.shields.io/badge/interface-Module-black)](#) [![OpenClaw Plugin](https://img.shields.io/badge/interface-OpenClaw_Plugin-black)](#) [![Skill](https://img.shields.io/badge/interface-Skill-black)](#) [![Universal Interface Spec](https://img.shields.io/badge/Universal_Interface_Spec-black?style=flat&color=black)](https://github.com/wipcomputer/wip-ai-devops-toolbox/blob/main/tools/wip-universal-installer/SPEC.md)

# 1Password Secrets

OpenClaw plugin for 1Password secrets. Fully headless via service account... no desktop app, no biometrics, no popups.

## Teach Your AI to Use Op Secrets

Open your AI and say:

```
Read the SKILL.md at SKILL.md.

Then explain to me:
1. What are these tools?
2. What do they do?
3. What would they change about how we work together?

Then ask me:
- Do you have more questions?
- Do you want to install them?

If I say yes, run: wip-install /path/to/repo --dry-run

Show me exactly what will change on my system. When I'm ready, I'll tell you
to install for real.
```

Your agent will read the repo, explain everything, and do a dry-run install first so you can see exactly what changes before anything is written to your system.

## Features

| Feature | Description | Status |
|---------|-------------|--------|
| **Config resolver** | Resolves `op://vault/item/field` references in `openclaw.json` at startup. Secrets never touch disk. | Stable |
| **Agent tools** | `op_read_secret`, `op_list_items`, `op_write_secret` for runtime access to 1Password. | Stable |
| **CLI diagnostics** | `openclaw op-secrets test` and `openclaw op-secrets read` for debugging. | Stable |
| **Env var injection** | Sets `process.env.OPENAI_API_KEY` from 1Password at boot so downstream libs find it. | Stable |

## More Info

- [Universal Interface Spec](https://github.com/wipcomputer/wip-ai-devops-toolbox/blob/main/tools/wip-universal-installer/SPEC.md) ... The six interfaces every agent-native tool can ship

## License

```
MIT      All CLI tools, MCP servers, skills, and hooks (use anywhere, no restrictions).
AGPLv3   Commercial redistribution, marketplace listings, or bundling into paid services.
```

AGPLv3 for personal use is free. Commercial licenses available.

Built by Parker Todd Brooks, Lēsa (OpenClaw, Claude Opus 4.6), Claude Code (Claude Opus 4.6).
