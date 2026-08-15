# Docgent Skills

OpenClaw skills for working with [Docgent](https://docs.docgent.io) documents.

## Install

```bash
openclaw skills install git:Vanaheim-Labs/docgent-skills
```

This installs `docgent-doc-access` into the active workspace's `skills/` directory.

## Configure

Add a per-brand agent token to `~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    entries: {
      "docgent-doc-access": {
        enabled: true,
        brands: {
          inkl: { token: "<agent token for the inkl brand>" }
        }
      }
    }
  }
}
```

See `SKILL.md` for the full behaviour, API contract, and current known gap
(server-side bearer-token auth is not yet live on docs.docgent.io — see the
skill file's "What this skill needs, and why it may not work yet" section).

## Update

```bash
openclaw skills update git:Vanaheim-Labs/docgent-skills
```
