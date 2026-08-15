---
name: "docgent-doc-access"
description: "Read, create, edit, render, and manage approval status of Docgent documents from a docs.docgent.io/<brand>/<slug> URL, using a per-brand agent token."
---

# Docgent Document Access

Use when a Docgent URL is pasted into a conversation (`https://docs.docgent.io/<brand>/<slug>`,
optionally with `?ref=<sha>`), or when asked to read, create, edit, render, diff, or change the
approval status of a Docgent document. Not for browsing Docgent generally, and not for documents
from any other system.

## Install (this skill, on a new OpenClaw instance)

```bash
openclaw skills install git:Vanaheim-Labs/docgent-skills
```

**When installing this skill because a human asked for it in chat, ask them for the brand
token(s) right then — do not silently skip credential setup and do not write anything to
`openclaw.json` yourself without an explicit token in hand.** The flow is:

1. Run the install command above.
2. Ask: *"Which brand(s) do you want this instance to access, and what's the agent token for
   each?"* (One token per brand — see "Config" below for where they come from.)
3. Once given a token, write it to `skills.entries.docgent-doc-access.brands.<brand>.token`
   in `openclaw.json` yourself (via the `gateway` config tool) — don't ask the human to hand-edit
   JSON.
4. Confirm back which brand(s) are now configured before considering the install done.

Do not proceed with install "successfully" if no token was provided — a skill installed
with zero brands configured cannot do anything, and should be reported as such, not as done.

## Config (where tokens live and where they come from)

```json5
// ~/.openclaw/openclaw.json
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

Tokens are currently issued manually (no self-serve onboarding flow yet, 2026-08-15) — an
operator sets `DOCGENT_AGENT_TOKEN_<BRAND_ID_UPPER>` on the Docgent Vercel deployment and
hands the value to whoever is installing this skill. A brand with no entry under `brands`
is simply not reachable — fail closed, report "no token configured for brand '<id>'", never
fall back to another brand's token or to signing in as a human to route around it.

## Every endpoint this skill uses

Base URL is always `https://docs.docgent.io`. Auth is `Authorization: Bearer <token>` on
every call. Brand and slug are lowercase path segments, e.g. `inkl`, `vanaheim`, `northface`.

### Read a document
`GET /api/doc/<brand>/<slug>` (optional `?ref=<commit-sha>` for a historical version)
→ `{ brand, slug, content, sha, frontmatter }`. `content` is the raw markdown source
(Docgent's ~25-term vocabulary — callouts, key figures, recommendations, etc. — not arbitrary
HTML). `sha` is the blob SHA; keep it, you need it to write back safely (see below).
A 404 means that slug doesn't exist under that brand yet.

### Create a new document
Same endpoint, different call: `PUT /api/doc/<brand>/<slug>` with
`{ content, message: "<why>" }` and **no `baseSha`**. Omitting `baseSha` on a slug that
doesn't exist yet is how creation works — the server rejects it with 409 if the slug already
exists (use the update path below instead). Content must start with YAML frontmatter; the
brand's doctype templates set the shape (title, subtitle, brand, doctype, version, date,
client, author, classification, status, toc) but there is currently no API to list a brand's
available doctypes/templates remotely — ask the human which doctype/template to follow, or
read one via the doc endpoint if an example document of that type already exists.

### Update an existing document
`PUT /api/doc/<brand>/<slug>` with `{ content, baseSha: <sha from a prior GET>, message }`.
**Always send the `sha` you actually read**, never a guessed or cached one — that's what makes
concurrent edits safe. A 409 means someone else changed the document since you read it; GET it
again and reconcile before retrying, don't just resend with a stale sha.

Response for both create and update: `{ changed: boolean, sha: <new sha>, commit }`.

### Propose an edit, don't just PUT — the actual expected workflow
Docgent's model is "AI rewrites are proposals, never direct commits" (from the product brief).
For an *edit to an existing document*, prefer the propose/accept pair over a raw PUT so the
change goes through the same diff-then-commit path a human would see in Studio:

1. `POST /api/rewrite/<brand>/<slug>` with `{ instruction, scope }` where `scope` is one of
   `{ kind: "document" }`, `{ kind: "section", heading: "<exact or near-exact heading text>" }`,
   or `{ kind: "range", start, end }` (character offsets). Returns `{ baseSha, proposed,
   before, after, diagnostics, valid, attribution, ... }` — a full document with the rewrite
   applied, not yet committed anywhere.
2. Show the human the diff (`before`/`after`, or diff `proposed` against the original via the
   diff endpoint below) and get confirmation.
3. `POST /api/rewrite/<brand>/<slug>/accept` with `{ content: <the proposed text>, baseSha,
   instruction, model, scopeLabel }` to actually commit it.

A direct `PUT /api/doc/<brand>/<slug>` is fine for creating a brand-new document (there is
nothing to diff against yet) or for a mechanical edit the human has already fully specified
verbatim. For "improve this section" / "rewrite the intro" style requests, use propose/accept.

### Diff between two versions
`GET /api/diff/<brand>/<slug>?base=<sha>&head=<sha-or-omit-for-HEAD>` →
`{ summary, headline, changes, unified }`. `changes` is Docgent's semantic diff (e.g. "key
figure value 4.2M → 4.8M"); `unified` is a normal line-based diff with context. Use this to
show a human what an accept/restore would actually change before doing it.

### Render to PDF
`GET /api/render/<brand>/<slug>` (optional `?ref=<sha>` for a historical version) → raw PDF
bytes, `Content-Type: application/pdf`. This renders committed content only — for unsaved/
in-flight content use the preview endpoints instead.

### Preview unsaved content (PDF or HTML)
`POST /api/preview/<brand>/<slug>` with `{ content }` → PDF bytes. `POST
/api/preview/<brand>/<slug>/html` with the same body → HTML with layout anchors. Use these to
show a human what a proposed edit will look like rendered, before it's committed.

### Read / change approval status
`GET /api/status/<brand>/<slug>` → `{ status, allowed }` (allowed = valid next statuses from
here). `POST /api/status/<brand>/<slug>` with `{ to, note, baseSha }` moves it — the lifecycle
is linear: `draft → review → approved → released → superseded`, plus one demotion back a step
at review/approved. A 409 means the requested transition isn't valid from the current status;
report the `allowed` list rather than retrying blindly.

### Restore an old version
`POST /api/restore/<brand>/<slug>` with `{ ref: <sha to restore>, baseSha, note }` — writes
the old content forward as a *new* commit (never rewrites history), bumping the version number
past every version this document has ever carried. Response: `{ restoredFrom, restoredVersion,
version, changed, sha, commit }`.

### What has no API (known gaps, be honest about them, don't invent a workaround)
- No endpoint to list a brand's documents remotely — an agent working purely through the API
  cannot discover "what documents exist for brand X" the way Studio's homepage does. If asked
  "what documents does inkl have", say this isn't available via the API yet rather than
  guessing or scraping the authenticated Studio UI.
- No endpoint to list a brand's available doctypes/templates remotely (see "Create" above).
- No endpoint to create a brand itself — brands are provisioned manually (brand.yaml + repo +
  token), not something this skill can do.

## Pitfalls

- Don't paste a document's raw `content` into a Slack reply as if it were plain text you
  wrote — the vocabulary/frontmatter distinction (title, status, classification) is part of
  what makes the document meaningful; summarise or quote it, don't strip its structure.
- The brand segment in a URL and the key under `brands` in config must match exactly
  (lowercase, e.g. `inkl`) — don't title-case or transform it before the config lookup.
- `docs.docgent.io` serves every brand from one host with the brand in the path, not a
  subdomain per brand — never construct `https://inkl.docgent.io/<slug>`; that pattern was
  deprecated 2026-08-15.
- Sending a guessed/stale `baseSha` instead of one you actually read from a GET response is
  the most common way to trigger an avoidable 409 — always read-then-write, never write blind.
- For an "improve/rewrite this" style request against an *existing* document, use
  propose→accept, not a raw PUT — a raw PUT with no diff step is what Docgent's product brief
  explicitly calls out as the wrong model ("AI rewrites are proposals, never direct commits").

## Verify

Before relying on a read, confirm the response actually has a `content` field and a non-null
`sha` — a caught server error can still return HTTP 200 from some fetch wrappers, so check the
JSON shape, not just the status code. Before reporting a write as done, confirm
`changed: true` and a new `sha` came back, not just a 200 status.
