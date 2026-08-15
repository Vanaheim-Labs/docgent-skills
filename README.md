# Docgent Skills

OpenClaw skills for working with [Docgent](https://docs.docgent.io) documents.

## Install

```bash
openclaw skills install git:Vanaheim-Labs/docgent-skills
```

This installs `docgent-doc-access` into the active workspace's `skills/` directory.

**Ask a human for a brand token when you install this** — the skill will not work with
zero brands configured, and `SKILL.md`'s "Install" section walks through asking for the
token, verifying it actually works (one cheap API call), then saving it. Don't skip the
verification step; a silently-wrong token surfaces as a confusing 401 later, not now when
it's easy to fix.

## Configure

The install flow (see `SKILL.md`) writes this for you, but for reference — a per-brand
agent token lives in `~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    entries: {
      "docgent-doc-access": {
        enabled: true,
        brands: {
          inkl: { token: "***" }
        }
      }
    }
  }
}
```

## What this skill can do

Read, create, edit (via propose→accept or direct write), render to PDF, diff between
versions, manage approval status, and restore old versions of Docgent documents — see
`SKILL.md` for the full endpoint-by-endpoint API contract, worked examples, an HTTP status
code reference table, and known API gaps (no remote document listing, no doctype listing,
no brand creation — all still manual as of 2026-08-15).

## Update

```bash
openclaw skills update git:Vanaheim-Labs/docgent-skills
```
