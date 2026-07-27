---
name: ai-klub-mcp-publishing
description: Use when an agent connected to AI Klub through MCP needs to research and publish source-grounded posts, including an explicit per-post Prioritní signály decision.
version: 1.2.0
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

Use when the user asks to publish, draft for publishing, research for publishing, or change whether an existing AI Klub article appears in **Prioritní signály**.

Do not use for:

- direct Supabase SQL, REST, dashboard, or service-role writes;
- publishing under a different person's identity;
- silently converting a rumor or community demo into a confirmed release;
- changing an existing post's homepage toggle without an explicit user request.

## Security Boundary

The connected user's OAuth token determines the effective AI Klub identity. Never request, store, inspect, or reuse OAuth tokens, Supabase keys, cookies, or credentials.

Use only the AI Klub MCP tools for blog reads and writes:

```text
get_my_permissions
get_publishing_guidelines
list_posts
create_post
set_post_homepage_featured  # existing post, only on explicit user request
```

Never substitute `create_post` with direct database access. Never substitute `set_post_homepage_featured` with direct database access. Do not call arbitrary Supabase endpoints.

### Permission Model

1. Call `get_my_permissions` before any publication or homepage-toggle attempt.
2. A user must have role `creator` or `admin` to publish.
3. `homepage_featured: true` and `set_post_homepage_featured` additionally require `effective_can_publish_homepage: true`.
4. If the role or homepage permission is insufficient, stop and report the required permission; do not try alternate credentials or write paths.
5. `create_post` publishes immediately. It is not a draft tool.

**Done:** the agent knows which delegated account and permissions it is acting under before creating or featuring content.

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

**Done:** category, uncertainty level, and hashtags match the evidence.

### 5. Prepare the article and cover

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
- **A cover image is mandatory. Never publish without one.** Use a relevant first-party WebP in AI Klub's `post-covers` bucket; preserve source ratio within 1280×720, encode lossy WebP at quality 78, keep it under 1.5 MB, and verify public HTTP 200 plus `image/webp` before publishing.

**Done:** the article is concise, source-grounded, and has a verified relevant first-party cover.

### 6. Always ask about privacy and Prioritní signály

**Before every `create_post` call, vždy výslovně zeptej se uživatele na obě nezávislé volby a vyčkej na odpověď:**

1. „Má být tento příspěvek soukromý? Soukromé příspěvky se před přihlášením zobrazují jen jako uzamčený náhled.“
2. „Má se tento příspěvek zobrazit v Prioritních signálech na domovské stránce?“

Do not infer either answer from the subject, source, prior posts, or a generic request to publish. Send the two explicit boolean answers as `is_private` and `homepage_featured`.

`homepage_featured: true` places a published post into the Prioritní signály carousel. Enabled posts sort automatically by `created_at` with the newest first; only the newest five are shown. `homepage_featured: false` means the post cannot appear in that carousel, but it may remain in lower homepage feeds.

Call `create_post` with:

```json
{
  "title": "...",
  "content": "CommonMark article...",
  "category": "AI novinky",
  "hashtags": ["models", "agentic"],
  "cover_image_url": "https://<project>.supabase.co/storage/v1/object/public/post-covers/posts/example.webp",
  "is_private": false,
  "homepage_featured": false
}
```

Do not add author IDs, emails, role fields, status overrides, or database fields not exposed by the MCP tool; the connector binds authorship to the OAuth user and publishes according to its policy.

**Done:** `create_post` returns a successful created post record.

### 7. Change an existing post's homepage toggle

Only when the user explicitly asks to change an existing article's appearance in Prioritní signály:

1. Call `get_my_permissions` and require `effective_can_publish_homepage: true`.
2. Use `list_posts` to resolve and verify the intended post ID.
3. Call `set_post_homepage_featured` exactly once with `post_id` and the requested boolean.
4. Read back with `list_posts` and confirm its `homepage_featured` value.

Never curate a manual sequence. The carousel order is automatic by publication date.

### 8. Read back and report

Call `list_posts` after publication or an existing-post toggle change. Confirm the expected title, category, hashtags, published state, delegated author identity, and `homepage_featured` value.

Report the created post ID when returned and whether it is private and/or in Prioritní signály.

## Common Pitfalls

1. **Writing directly to Supabase.** Never do this for delegated publishing. Use MCP tools only.
2. **Assuming OAuth scopes grant publishing.** OAuth identifies the user; AI Klub role/RLS determines whether publishing is allowed.
3. **Publishing a draft by accident.** `create_post` is immediate.
4. **Skipping either explicit question.** Privacy and homepage placement are separate choices; always ask both.
5. **Publishing without a cover.** This is not allowed. Use a verified relevant first-party WebP.
6. **Treating a social post as a release note.** Find a primary source or categorize the item as `leaks`.
7. **Skipping duplicate checks.** Always call `list_posts` before `create_post`.
8. **Trying to control carousel order manually.** The newest enabled posts sort automatically; use the per-post toggle only.

## Verification Checklist

- [ ] `get_my_permissions` confirms `creator` or `admin`
- [ ] `get_publishing_guidelines` loaded
- [ ] `list_posts` checked for duplicates
- [ ] Every material claim has a primary source or an explicit uncertainty label
- [ ] A relevant first-party WebP cover exists, is public, and has been verified before publication
- [ ] User explicitly answered both privacy and Prioritní signály questions
- [ ] `create_post` receives explicit `is_private` and `homepage_featured` booleans
- [ ] `list_posts` read-back finds exactly one created post with the requested toggle state
