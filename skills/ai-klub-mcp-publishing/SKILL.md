---
name: ai-klub-mcp-publishing
description: Use when an agent connected to AI Klub through MCP needs to research, draft, and publish a source-grounded blog post using only the delegated AI Klub MCP write path.
version: 1.1.0
author: AI Klub
license: MIT
metadata:
  hermes:
    tags: [ai-klub, mcp, oauth, publishing, editorial]
    related_skills: []
---

# AI Klub MCP Publishing

Publish AI Klub articles through the authenticated user's MCP connection. The MCP connector is the **only permitted write path** for blog content.

## When to Use

Use when the user asks to publish, draft for publishing, research for publishing, or curate an AI Klub blog post through a connected Claude/agent.

Do not use for:

- direct Supabase SQL, REST, dashboard, or service-role writes;
- publishing under a different person's identity;
- silently converting a rumor or community demo into a confirmed release;
- homepage curation unless the user explicitly asks for it.

## Security Boundary

The connected user's OAuth token determines the effective AI Klub identity. Never request, store, inspect, or reuse OAuth tokens, Supabase keys, cookies, or credentials.

Use only the AI Klub MCP tools for blog reads and writes:

```text
get_my_permissions
get_publishing_guidelines
list_posts
create_post
set_homepage_selection  # only on explicit user request
```

Never substitute `create_post` with direct database access. Do not call arbitrary Supabase endpoints.

### Permission Model

1. Call `get_my_permissions` before any publication attempt.
2. A user must have role `creator` or `admin` to publish.
3. If the role is insufficient, stop and report the role required; do not try alternate credentials or write paths.
4. `create_post` publishes immediately. It is not a draft tool.
5. `set_homepage_selection` is separate and requires explicit user intent plus homepage permission.

**Done:** the agent knows which delegated account and role it is acting under before creating content.

## Publication Tight Loop

### 1. Confirm publication intent

Treat an explicit request such as “publish this” or “put this on the blog” as permission to publish immediately. If the user asks only to research, summarize, or draft, do not call `create_post`.

**Done:** publish-now versus draft-only is unambiguous.

### 2. Load the editorial contract

Call, in this order:

1. `get_my_permissions`
2. `get_publishing_guidelines`
3. `list_posts` with a useful limit

Use the returned guidelines as the source of truth for language, Markdown, category, hashtags, covers, and publication semantics.

**Done:** role is eligible, constraints are loaded, and relevant recent posts are available for duplicate comparison.

### 3. Research before writing

Use the agent's read-only web, browser, repository, or document tools to research. Preserve a source for every material claim.

Source hierarchy:

1. vendor/product documentation, model cards, release notes, repositories, or official announcements;
2. the vendor's official social account;
3. reputable secondary sources only for context, clearly attributed;
4. community posts only as leads or explicitly labelled unverified signals.

For an X/social lead, find the original organisation's official source before treating a claim as confirmed. If the official source is unavailable or blocked, keep only claims visible in the official post and state the limitation.

**Done:** every planned factual claim has a direct source, or is omitted/labeled as uncertain.

### 4. Classify the item

Use independent metadata dimensions:

| Situation | Category | Rule |
|---|---|---|
| Confirmed release, research, or product update | `AI novinky` or `AI nástroje` | Cite primary sources. |
| Practical tool or service | `AI nástroje` | Explain use and caveats. |
| Community demo, rumor, leak, or missing primary confirmation | `leaks` | Put uncertainty in title and opening paragraph. |

Use 1–8 lowercase practical hashtags without `#`, for example `models`, `coding`, `agentic`, `security`, `research`, `automation`, or `local-ai`.

Do not use hashtags as a replacement for category, author, or subsection metadata. If the connector lacks a required metadata field, do not invent a workaround.

**Done:** category, uncertainty level, and hashtags match the evidence.

### 5. Prepare the article

Write Czech CommonMark unless the user asks for another language.

Minimum structure:

```markdown
Krátký úvod s hlavním ověřeným zjištěním.

## Co je potvrzené

## Praktický dopad / proč je to důležité

## Výhrady nebo limity

## Zdroje

- [Primární zdroj](https://example.com/)
```

Rules:

- No raw HTML.
- Do not invent model specifications, pricing, availability, benchmarks, quotes, or source URLs.
- For unverified leads, say what the source shows and what remains unconfirmed.
- **A cover image is mandatory. Never publish without one.** Every published article must have a verified `cover_image_url`; omitting it makes the publication incomplete. Use a relevant first-party WebP hosted in AI Klub's `post-covers` bucket. Download external source images, visually inspect them, preserve their aspect ratio within a 1280×720 bounding box, encode lossy WebP at quality 78, keep the file under 1.5 MB, upload it to `post-covers/posts/<post-id>.webp`, and use only the resulting first-party public URL. If no suitable source image exists, create a relevant original editorial cover; do not omit the cover and do not use an unrelated placeholder. Verify HTTP 200, `image/webp`, nonzero dimensions, public access, visual relevance, and the first-party URL before publication.

**Done:** the article is concise, source-grounded, and ready to pass unchanged to `create_post`.

### 6. Publish through MCP only

Call `create_post` with:

```json
{
  "title": "...",
  "content": "CommonMark article...",
  "category": "AI novinky",
  "hashtags": ["models", "agentic"],
  "cover_image_url": "https://public.example/cover.png"
}
```

Do not call `create_post` until a verified cover exists. Never add author IDs, emails, role fields, status overrides, or database fields not exposed by the MCP tool; the connector binds authorship to the OAuth user and publishes according to its policy.

**Done:** `create_post` returns a successful created post record.

### 7. Read back and report

Call `list_posts` after publication. Confirm the created post appears once with the expected title, category, hashtags, published state, and delegated author identity.

If public web access is available, inspect the public blog rendering separately. This is read-only verification, not an alternate write path.

Report:

- title and category;
- whether the item was confirmed or explicitly marked as a leak;
- primary source links;
- any unresolved limitation;
- the MCP-created post ID, if returned.

**Done:** the user can identify the published item and audit the evidence without relying on an unverified completion claim.

## Common Pitfalls

1. **Writing directly to Supabase.** Never do this for delegated publishing. Use `create_post` only.
2. **Assuming OAuth scopes grant publishing.** OAuth identifies the user; AI Klub role/RLS determines whether publishing is allowed.
3. **Publishing a draft by accident.** `create_post` is immediate. Ask if the user did not explicitly request publication.
4. **Treating a social post as a release note.** Find a primary source or categorize the item as `leaks`.
5. **Publishing without a cover.** This is not allowed. Use a verified first-party WebP from a relevant official source, or create a relevant original editorial cover; never use an unrelated placeholder.
6. **Using an unverified community image as an official cover.** A community image may be used only when the article explicitly documents that same community signal and the cover is clearly relevant; otherwise use an original editorial cover or an official source asset.
7. **Skipping duplicate checks.** Always call `list_posts` before `create_post`.
8. **Curating the homepage implicitly.** Never call `set_homepage_selection` unless the user explicitly requests homepage placement.

## Verification Checklist

- [ ] `get_my_permissions` confirms `creator` or `admin`
- [ ] `get_publishing_guidelines` loaded
- [ ] `list_posts` checked for duplicates
- [ ] Every material claim has a primary source or an explicit uncertainty label
- [ ] A relevant first-party WebP cover exists, is public, and has been verified before publication
- [ ] Category and hashtags match the evidence
- [ ] Article uses CommonMark without raw HTML
- [ ] `create_post` was the only blog write operation
- [ ] `list_posts` read-back finds exactly one created post
- [ ] Homepage selection was untouched unless explicitly requested
