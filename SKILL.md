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
3. **Verify each token before saving it, don't save-and-hope.** Call
   `GET https://docs.docgent.io/api/status/<brand>/__token-check__` with the token as a bearer
   header. A `404` means the token is valid and that brand is reachable (the slug just doesn't
   exist, which is expected — that's fine, it proves auth passed). A `401` means the token is
   wrong, expired, or for the wrong brand — say so plainly and ask for a corrected token rather
   than saving a token you know is bad.
4. Once verified, write it to `skills.entries.docgent-doc-access.brands.<brand>.token` in
   `openclaw.json` yourself (via the `gateway` config tool) — don't ask the human to hand-edit
   JSON.
5. **Patch AGENTS.md** — append the following block verbatim to the workspace AGENTS.md file
   (create the file if it doesn't exist). Replace `<brand>` with each brand you just configured,
   comma-separated if multiple (e.g. `inkl`, `vanaheim`):

   ```markdown
   ## Docgent — URL Routing Rule

   When you see any URL matching `docs.docgent.io<brand>/<slug>` (or a short form
   like `docgent.io/<brand>/<slug>`), **always invoke the `openclaw-skills:docgent-doc-access`
   skill before doing anything else** — never attempt an unauthenticated `web_fetch` of a
   Docgent URL. Unauthenticated fetches always redirect to the sign-in page. The skill handles
   authentication via the configured bearer token for that brand.

   This rule applies in every session, channel, and context (Slack threads, DMs, heartbeats).
   ```

   If AGENTS.md already has a `## Docgent` section from a previous install, replace it rather
   than appending a duplicate.

6. Confirm back which brand(s) are now configured **and verified working** before considering
   the install done. Do not report success for a brand whose token you saved but never checked.

Do not proceed with install "successfully" if no token was provided, or if the only token given
failed verification — a skill installed with zero *working* brands cannot do anything, and that
should be reported as a blocker, not glossed over as done.

**Example install report, once verified:**
> Installed `docgent-doc-access`. Verified and configured: `northface` ✅, `increm` ✅.
> AGENTS.md patched with URL routing rule ✅. Ready to use — paste a
> `docs.docgent.io/northface/...` or `docs.docgent.io/increm/...` URL any time.

## Config (where tokens live and where they come from)

Multiple brands are stored side by side under `brands` — add as many as needed:

```json5
// ~/.openclaw/openclaw.json
{
  skills: {
    entries: {
      "docgent-doc-access": {
        enabled: true,
        brands: {
          northface: { token: "***" },
          increm:    { token: "***" },
          vanaheim:  { token: "***" }
          // add further brands here as provisioned
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

## URL-driven brand inference (multi-brand workflow)

When a `docs.docgent.io/<brand>/<slug>` URL is pasted into a conversation, extract the brand
slug from the path and use the matching token automatically. Do **not** ask the user which
brand to use — the URL is the source of truth.

Steps:
1. Parse `<brand>` from the URL path (first segment after the host, lowercase).
2. Check `skills.entries.docgent-doc-access.brands.<brand>` exists and has a `token`.
3. If found, use that token for all subsequent API calls in this thread.
4. If not found, report "no token configured for brand '<brand>'" and stop — do not fall back
   to another brand's token.

**Multiple brands in one thread:** use whichever brand the most recently referenced URL
belongs to. Each API call uses only that brand's token — never mix tokens across brands.

**Thread with no document URL:** if a user asks to work on Docgent documents without pasting
a URL, ask which brand and document slug they want before proceeding.

## Every endpoint this skill uses

Base URL is always `https://docs.docgent.io`. Auth is `Authorization: Bearer <token>` on
every call. Brand and slug are lowercase path segments, e.g. `inkl`, `vanaheim`, `northface`,
`increm`.

### Read a document
`GET /api/doc/<brand>/<slug>` (optional `?ref=<commit-sha>` for a historical version)
→ `{ brand, slug, content, sha, frontmatter }`. `content` is the raw markdown source
(Docgent's ~25-term vocabulary — callouts, key figures, recommendations, etc. — not arbitrary
HTML). `sha` is the blob SHA; keep it, you need it to write back safely (see below).
A 404 means that slug doesn't exist under that brand yet.

**Worked example:**
```
GET /api/doc/northface/q3-strategy-memo
Authorization: Bearer <token>

200 OK
{
  "brand": "northface",
  "slug": "q3-strategy-memo",
  "content": "---\ntitle: \"Q3 Strategy\"\nstatus: draft\n---\n\n# Context\n...",
  "sha": "a1b2c3d...",
  "frontmatter": { "title": "Q3 Strategy", "status": "draft", "doctype": "Strategy Memo" }
}
```

### List a brand's documents
`GET /api/docs/<brand>` (optional `?status=<status>` and/or `?doctype=<doctype>` filters)
→ `{ brand, count, documents: [{ slug, title, doctype, status, classification, version, date,
sha }] }`. Metadata only — no `content` — use this to discover what exists, then
`GET /api/doc/<brand>/<slug>` for the body of whichever one you need.

### Create a new document
`PUT /api/doc/<brand>/<slug>` with `{ content, message: "<why>" }` and **no `baseSha`**.
Omitting `baseSha` on a slug that doesn't exist yet is how creation works — the server rejects
it with 409 if the slug already exists (use the update path below instead). Content must start
with YAML frontmatter.

### Update an existing document
`PUT /api/doc/<brand>/<slug>` with `{ content, baseSha: <sha from a prior GET>, message }`.
**Always send the `sha` you actually read**, never a guessed or cached one — that's what makes
concurrent edits safe. A 409 means someone else changed the document since you read it; GET it
again and reconcile before retrying, don't just resend with a stale sha.

Response for both create and update: `{ changed: boolean, sha: <new sha>, commit }`.

### Propose an edit, don't just PUT — the actual expected workflow
Docgent's model is "AI rewrites are proposals, never direct commits". For an *edit to an
existing document*, prefer the propose/accept pair over a raw PUT:

1. `POST /api/rewrite/<brand>/<slug>` with `{ instruction, scope }` where `scope` is one of
   `{ kind: "document" }`, `{ kind: "section", heading: "<exact or near-exact heading text>" }`,
   or `{ kind: "range", start, end }` (character offsets). Returns `{ baseSha, proposed,
   before, after, diagnostics, valid, attribution, ... }` — a full document with the rewrite
   applied, not yet committed anywhere.
2. Show the human the diff (`before`/`after`, or diff `proposed` against the original via the
   diff endpoint below) and get confirmation.
3. `POST /api/rewrite/<brand>/<slug>/accept` with `{ content: <the proposed text>, baseSha,
   instruction, model, scopeLabel }` to actually commit it.

A direct `PUT /api/doc/<brand>/<slug>` is fine for creating a brand-new document (nothing to
diff against yet) or for a mechanical edit the human has fully specified verbatim. For
"improve this" / "rewrite the intro" style requests against an existing document, use
propose/accept.

### Diff between two versions
`GET /api/diff/<brand>/<slug>?base=<sha>&head=<sha-or-omit-for-HEAD>` →
`{ summary, headline, changes, unified }`. `changes` is Docgent's semantic diff (e.g. "key
figure value 4.2M → 4.8M"); `unified` is a normal line-based diff with context.

### Render to PDF
`GET /api/render/<brand>/<slug>` (optional `?ref=<sha>` for a historical version) → raw PDF
bytes, `Content-Type: application/pdf`. Renders committed content only — for unsaved/in-flight
content use the preview endpoints instead.

### Preview unsaved content (PDF or HTML)
`POST /api/preview/<brand>/<slug>` with `{ content }` → PDF bytes. `POST
/api/preview/<brand>/<slug>/html` with the same body → HTML with layout anchors.

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
- No endpoint to list a brand's available doctypes/templates remotely — the `doctype` values
  in `GET /api/docs/<brand>` are a good proxy (they're whatever existing documents actually
  use), but there's no template/schema listing yet.
- No endpoint to create a brand itself — brands are provisioned manually (brand.yaml + repo +
  token), not something this skill can do.

## HTTP status codes, what each one actually means here

| Code | Meaning in this API | What to do |
|---|---|---|
| 200 | Success (read, render, preview, diff) | Use the response |
| 401 | No/invalid session and no valid bearer token for this brand | Check the token is set and correct for *this* brand — don't retry with a different brand's token |
| 404 | Slug doesn't exist under that brand | For a read: report it doesn't exist yet. For a status check during token verification: this is a *success* signal (auth passed, the probe slug just isn't real) |
| 409 | Concurrent write conflict, or invalid status transition, or restoring a no-op | Refetch current state and reconcile — never blindly retry the same write |
| 422 | Content failed vocabulary validation | Read `diagnostics` in the response and fix the specific block(s) named, don't retry unchanged |
| 502 | Render/preview pipeline failed | Not a content problem — report as an infrastructure issue, don't retry rewriting the document to work around it |

## Pitfalls

- Never fetch a `docs.docgent.io` URL with a generic `web_fetch`/browser tool — unauthenticated
  requests redirect straight to the sign-in page and return no real document content. Any URL
  matching `docs.docgent.io/<brand>/<slug>` is a trigger to use this skill's endpoints directly.
- Don't paste a document's raw `content` into a chat reply as if it were plain text you
  wrote — summarise or quote it, don't strip its structure.
- The brand segment in a URL and the key under `brands` in config must match exactly
  (lowercase) — don't title-case or transform it before the config lookup.
- `docs.docgent.io` serves every brand from one host with the brand in the path, not a
  subdomain per brand — never construct `https://<brand>.docgent.io/<slug>`; that pattern was
  deprecated 2026-08-15.
- Sending a guessed/stale `baseSha` instead of one you actually read from a GET response is
  the most common way to trigger an avoidable 409 — always read-then-write, never write blind.
- For an "improve/rewrite this" style request against an *existing* document, use
  propose→accept, not a raw PUT — a raw PUT skips the review step Docgent's model requires.
- Don't skip the token-verification step at install — a silently-wrong token surfaces as a
  confusing 401 on the first real document request, not at install time when it's easy to fix.
- **Do not infer that a brand is configured just because this skill is installed.** Always
  check config for actual brand entries before claiming any brand is reachable.

## Verify

Before relying on a read, confirm the response actually has a `content` field and a non-null
`sha` — a caught server error can still return HTTP 200 from some fetch wrappers, so check the
JSON shape, not just the status code. Before reporting a write as done, confirm
`changed: true` and a new `sha` came back, not just a 200 status.
