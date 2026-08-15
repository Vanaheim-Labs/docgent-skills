---
name: "docgent-doc-access"
description: "Read and write Docgent documents from a docs.docgent.io/<brand>/<slug> URL pasted into chat, using a per-brand agent token."
primaryEnv: "DOCGENT_AGENT_TOKEN"
metadata:
  openclaw:
    requires:
      config:
        - "skills.entries.docgent-doc-access.brands"
---

# Docgent Document Access

Use when a Docgent URL is pasted into a conversation (`https://docs.docgent.io/<brand>/<slug>`,
optionally with `?ref=<sha>`) and the task is to read, summarise, or propose an edit to that
document — not to browse Docgent generally, and not for documents from any other system.

## What this skill needs, and why it may not work yet

This skill calls Docgent's per-document API
(`GET/PUT https://docs.docgent.io/api/doc/<brand>/<slug>`) with a bearer token:

```
Authorization: Bearer <brand-scoped agent token>
```

**As of 2026-08-15, Docgent's API routes only accept a signed-in Studio session
cookie — there is no server-side bearer-token check yet.** That is separate,
not-yet-built work (adding `validAgentToken(req, brand)` alongside the existing
`auth()` check in `apps/studio/src/app/api/doc/[brand]/[slug]/route.ts`, and a
CLI command to mint a token per brand). Until that lands, a call from this skill
will get a 401 even with a token configured here. Report that plainly rather than
guessing at a workaround — do not attempt to sign in as a human or scrape the
authenticated Studio UI to route around it.

## Config shape (per-brand tokens, one skill entry)

One OpenClaw instance may need to reach several brands' documents, so tokens are
nested under a single skill entry rather than one entry per brand:

```json5
// ~/.openclaw/openclaw.json
{
  skills: {
    entries: {
      "docgent-doc-access": {
        enabled: true,
        brands: {
          inkl: { token: "<agent token for the inkl brand>" },
          vanaheim: { token: "<agent token for the vanaheim brand>" },
          northface: { token: "<agent token for the northface brand>" }
        }
      }
    }
  }
}
```

A brand with no entry under `brands` is simply not reachable by this skill —
fail closed, report "no token configured for brand '<id>'", do not fall back to
an unscoped or another brand's token.

## Procedure

1. **Extract brand + slug from the URL.** A Docgent doc URL is always
   `https://docs.docgent.io/<brand>/<slug>` (optionally `?ref=<commit-sha>` for
   a pinned historical version, or `?v=<sha>` on document pages — the API param
   is `ref`). Parse the first two path segments; do not guess brand/slug from
   message text if a URL is present — the URL is authoritative.
2. **Look up the token.** Read `skills.entries.docgent-doc-access.brands.<brand>.token`
   from `openclaw.json` (via the `gateway` config tool, `config.get`). If missing,
   stop and report which brand has no token configured — do not ask the user to
   paste a token into chat.
3. **Read the document:**
   `GET https://docs.docgent.io/api/doc/<brand>/<slug>[?ref=<sha>]` with the
   bearer token. Response: `{ brand, slug, content, sha, frontmatter }`. `content`
   is the raw markdown (Docgent's ~25-term vocabulary, not arbitrary HTML) — treat
   it as the document, not as something to re-render or reformat before showing
   the user.
4. **Proposing an edit — never free-write to HEAD.** Docgent's model is
   "AI rewrites are proposals, never direct commits" (see the product brief).
   Do not `PUT` a rewritten `content` straight back without the user reviewing a
   diff first. Show the proposed change, get confirmation, and only then call
   `PUT https://docs.docgent.io/api/doc/<brand>/<slug>` with
   `{ content, baseSha: <the sha from step 3>, message: "<what changed and why>" }`.
   Send the sha you read, not one you inferred — a stale `baseSha` returns 409,
   which means someone else changed the document since you read it; refetch and
   re-diff rather than retrying with a guessed sha.
5. **Report the result plainly.** A 401 means the bearer-token check isn't live
   yet (see above) or the token is wrong/revoked — do not retry with a different
   brand's token. A 404 means the slug doesn't exist under that brand. A 409 is a
   concurrent-edit conflict, not a bug — refetch and retry once with the fresh sha.

## Pitfalls

- Don't paste a raw Docgent URL's content into a Slack reply as if it were plain
  text you wrote — the vocabulary/frontmatter distinction (title, status,
  classification) is part of what makes the document meaningful; summarise or
  quote it, don't strip its structure silently.
- The brand segment in the URL and the key under `brands` in config must match
  exactly (Docgent brand ids are lowercase, e.g. `inkl`, `vanaheim`, `northface`)
  — do not title-case or otherwise transform it before the config lookup.
- `docs.docgent.io` serves every brand from one host with the brand in the path
  (not a subdomain per brand) — do not construct URLs like
  `https://inkl.docgent.io/<slug>`; that pattern was deprecated 2026-08-15.

## Verify

Before relying on a read, confirm the response actually has a `content` field
and a non-null `sha` — a caught error still returns HTTP 200 from some fetch
wrappers, so check the JSON shape, not just the status code.
