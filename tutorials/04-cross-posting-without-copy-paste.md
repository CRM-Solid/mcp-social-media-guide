# Cross Post to Multiple Social Networks Without Copy and Paste

To cross post to multiple social networks properly, you write one source note, ask your assistant for a variant per platform, schedule the variants as a single batch, then edit any individual variant that needs it. What you do not do is paste identical text into five boxes. This tutorial walks the whole loop over the Model Context Protocol against a CRM Solid MCP server, taking one idea to LinkedIn, X, Instagram, Threads and TikTok.

The identical dump is the default because it is easy, and it is also why cross posted content underperforms native content on every platform at once. A 900 character LinkedIn post truncated into X is not a shorter post, it is a broken one.

## What changes per platform, and what breaks first

| Platform | Practical text ceiling | Media | What breaks first |
|---|---|---|---|
| LinkedIn | About 3,000 characters | Optional | Roughly the first 200 characters show before "see more". Everything after that is opt in |
| X | 280 characters on a standard account, more on paid tiers | Optional | Anything longer needs a deliberate thread, not a truncation |
| Instagram | About 2,200 character caption | Required | Links in the caption are not clickable, so a link-led post dies |
| Threads | About 500 characters | Optional | Long text gets chained into replies and loses the ending |
| TikTok | Caption only, video required | Required, video | A text-only idea cannot exist here at all |

Treat those numbers as current practice, not as contract. Platforms change limits without notice, and the authoritative answer is whatever error the platform returns at publish time. The last section covers reading those errors.

## What you need

| Requirement | Detail |
|---|---|
| MCP client | Claude Desktop, Claude Code, Cursor, ChatGPT, or any MCP client |
| Server | `@crmsolid/mcp-server` over stdio |
| Scopes | `posts:read` and `posts:write` |
| Accounts | The five platforms connected, each with its own account id |

```jsonc
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "posts"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

Start with [01: Connect Your First Social MCP Server](./01-connect-your-first-social-mcp-server.md) if the server is not running yet. MCP tool output is camelCase; the v1 REST API underneath is PascalCase. Everything below is camelCase.

Before anything else, get the account ids straight.

```json
{
  "name": "crm_list_social_accounts",
  "arguments": {}
}
```

```json
{
  "count": 5,
  "accounts": [
    { "id": 12, "platform": "linkedin", "displayName": "Mert Aydin", "username": "mert-aydin", "isActive": true, "timeZone": "Europe/Istanbul", "dailyPostLimit": 10 },
    { "id": 15, "platform": "x", "displayName": "Mert Aydin", "username": "mertaydin", "isActive": true, "timeZone": "Europe/Istanbul", "dailyPostLimit": 25 },
    { "id": 18, "platform": "instagram", "displayName": "Mert Aydin", "username": "mertaydin", "isActive": true, "timeZone": "Europe/Istanbul", "dailyPostLimit": 5 },
    { "id": 21, "platform": "threads", "displayName": "Mert Aydin", "username": "mertaydin", "isActive": true, "timeZone": "Europe/Istanbul", "dailyPostLimit": 10 },
    { "id": 24, "platform": "tiktok", "displayName": "Mert Aydin", "username": "mertaydin", "isActive": true, "timeZone": "Europe/Istanbul", "dailyPostLimit": 3 }
  ]
}
```

**Verify.** Five accounts, every one `isActive: true`, and note each `dailyPostLimit` before you plan a heavy day. An inactive account will accept a schedule and fail at publish, which is the worst place to find out. Account ids are integers, and you will pass them as integers in `accountIds`.

## Step 1: Write one source note, not one post

The source note is the raw material. It is longer than any variant and is never published. Write it yourself, in plain text, and paste it in.

```text
SOURCE NOTE

Claim: most support backlogs are a routing problem, not a staffing problem.

Evidence: we migrated 40 support inboxes this year. In 31 of them, median
first response time dropped by more than half after we changed routing rules
only, with no headcount change.

Detail worth keeping: the biggest single win was routing by the first inbound
channel rather than by topic. Topic routing needs someone to read the message
first, which is the delay you were trying to remove.

Audience: support leads who inherited an inbox they did not design.

Ask: reply with how you route today.

Do not say: "paradigm shift", "10x", anything about AI replacing agents.
```

**Verify.** The note contains a claim, a number you can defend, a specific mechanism, and a named audience. If any of the four is missing, the variants will be padding. A model can rewrite; it cannot supply evidence you did not have.

## Step 2: Ask for platform variants and review them as a table

One turn, five variants, no tool calls yet.

```text
From the source note, write variants for LinkedIn, X, Instagram, Threads and TikTok.

Rules per platform:
- LinkedIn: 900 to 1,400 characters. The first 200 characters must stand alone and
  earn the "see more" click. Three short paragraphs after that. End with the ask.
- X: under 250 characters. One claim, one number. No list, no line-break ladder,
  no closing question, no hashtags.
- Instagram: 400 to 700 character caption. Assume the reader saw an image first.
  No link in the caption. Two hashtags maximum.
- Threads: under 450 characters. Conversational, one idea, no formatting.
- TikTok: caption under 150 characters plus a 20 second script outline as bullets.

Output a table: platform, character count, variant text.
Do not schedule anything yet.
```

The character count column is the review tool. Read it first, before you read a word of copy. A variant outside its band was written to a different brief than the one you gave.

**Verify.** Five rows, five distinct openings. If three variants open with the same construction, reject the batch and ask for the X and Threads variants again in their own turn. Models drafting five things at once converge on one rhythm, and readers who follow you in two places will notice immediately.

## Step 3: Cross post to multiple social networks as one scheduled batch

Here is the thing that trips people up on the first run.

**`platforms` names the networks that receive identical text. It does not generate variants.** Two platform names in one array means the exact same characters go to both. So a batch is one `crm_schedule_social_post` call per distinct piece of copy, grouped only where the copy is genuinely the same.

One call is not one post, though. The server writes one post row per target account, so a two account call comes back with two ids in `postIds` and each row lives, fails and cancels on its own. In this example, the X variant is 240 characters and reads correctly on Threads too, so those two share a call. LinkedIn, Instagram and TikTok each get their own.

```text
Schedule the batch for Wednesday 26 August, Europe/Istanbul local times:
- LinkedIn variant at 09:00, account 12
- X variant at 12:30, account 15, and use the same text for Threads, account 21
- Instagram variant at 18:00, account 18, mediaUrls as I paste below
- TikTok variant at 18:00, account 24, mediaUrls as I paste below

Convert each local time to a UTC instant and print both before each call.
Then call crm_schedule_social_post once per distinct copy.
```

```json
{
  "name": "crm_schedule_social_post",
  "arguments": {
    "content": "Most support backlogs are a routing problem, not a staffing problem. We moved 40 inboxes this year. In 31 of them median first response time more than halved with no new headcount.",
    "platforms": ["x", "threads"],
    "accountIds": [15, 21],
    "scheduledAt": "2026-08-26T09:30:00Z",
    "timeZone": "Europe/Istanbul"
  }
}
```

```json
{
  "count": 2,
  "postIds": [994, 995],
  "platforms": ["x", "threads"],
  "scheduledAt": "2026-08-26T09:30:00Z",
  "status": "pending",
  "skipped": null,
  "message": "Scheduled on 2 account(s) for 2026-08-26 09:30 UTC."
}
```

12:30 in Istanbul is 09:30 UTC. A `scheduledAt` carrying a trailing `Z` or an offset names the instant outright, and `timeZone` is ignored for it. A bare wall clock is converted from `timeZone` instead, and read as UTC if you left `timeZone` out, which is the version that publishes at the wrong hour. Say which clock you mean one way or the other. [03: AI Content Calendar](./03-ai-content-calendar.md) has the full worked example of getting that wrong.

**`scheduledAt` is required unless you pass `publishNow: true`, and there is no draft state.** A batch call that forgets the time is rejected with "scheduledAt is required unless publishNow is true", so the failure mode is an error, never a surprise publish. If you want a review gap, schedule the batch at a parking time you can move later rather than hoping an omission is safe.

**Verify.** List the batch back.

```json
{
  "name": "crm_list_social_posts",
  "arguments": { "status": "pending", "fromDate": "2026-08-26", "toDate": "2026-08-27" }
}
```

Five rows from four calls, because the shared X and Threads copy became one row per account. Note the argument names are `fromDate` and `toDate`, and that the list takes `limit` (1 to 100, default 25) with no cursor to follow. Check every `scheduledAt` is in the future and every row's platform matches the account it was written for. A LinkedIn account id passed alongside an Instagram platform is accepted at schedule time and fails at publish.

## Step 4: Edit one variant without touching the others

`crm_update_social_post` takes `postId` and any of `content`, `scheduledAt`, `mediaUrls` and `timeZone`. It is annotated idempotent, so sending the same change twice changes nothing. It edits a `pending` post and nothing else: a published, failed or cancelled row answers "Post 993 not found, or it is no longer editable (only pending posts can be edited)".

Say the LinkedIn opening is weak. Fix that one row only.

```json
{
  "name": "crm_update_social_post",
  "arguments": {
    "postId": 993,
    "content": "We changed routing rules in 40 support inboxes this year and touched nobody's headcount. In 31 of them, median first response time more than halved.\n\nThe rule that did the work was routing by inbound channel, not by topic..."
  }
}
```

The other four rows are untouched, including both halves of the X and Threads pair.

Now the harder edit. You decide Threads deserves its own copy after all. `platforms` is not editable, and it does not need to be: the Threads row is already its own post, so cancel it and schedule a replacement.

```json
{
  "name": "crm_cancel_social_post",
  "arguments": { "postId": 995 }
}
```

```json
{
  "name": "crm_schedule_social_post",
  "arguments": {
    "content": "Routing, not staffing. 40 inboxes, 31 of them cut median first response time by more than half, same headcount. Curious how you route today.",
    "platforms": ["threads"],
    "accountIds": [21],
    "scheduledAt": "2026-08-26T09:35:00Z",
    "timeZone": "Europe/Istanbul"
  }
}
```

**Verify.** Read both posts back with `crm_get_social_post`. Post 995 should carry `"status": "cancelled"`, and the new row should be `pending` on the Threads account alone. If 995 is still `pending`, the cancel did not land and Threads will publish twice. Check this before you walk away; it is the one edit in this workflow with a visible public failure. Cancelling an already cancelled post succeeds and says so, so a second attempt is safe.

## The quality bar: what LinkedIn earns that X punishes

The two are close to opposites, and holding the line between them is most of the work.

A good **LinkedIn** variant earns the expand click, then rewards it. The first two lines carry a complete claim with a number in it, because that is all most readers ever see. The middle has structure: three short paragraphs, or a list of three, with real line breaks. It names the reader by role. It closes with a question a practitioner can actually answer in one sentence, which is what produces comments rather than reactions.

A good **X** variant must not do any of that. No "Here are 3 things we learned". No line-break ladder. No closing question inviting engagement. No hashtag stack. One claim, one number, one clause of proof, readable in a scroll without expanding. If the idea genuinely needs more room, write a thread on purpose, with each post standing alone. A truncated LinkedIn post is not a thread.

The short version: LinkedIn rewards structure, X reads structure as performance. If you can paste a variant from one into the other and it still looks native, at least one of them is wrong.

Instagram sits apart again: the caption is a companion to the image, not a replacement for it, and a caption that opens by explaining the image has already lost. Threads tolerates the X variant more often than not, which is why grouping them is usually safe and occasionally lazy. TikTok is a script with a caption attached, and the caption is the least important thing in it.

## Media: what `mediaUrls` accepts and where it stops

`mediaUrls` is an array of publicly reachable HTTPS URLs. The CRM fetches them server-side at publish time, which produces four practical rules.

1. **The URL must resolve without authentication.** Open it in a private browser window before you paste it. A file behind a login works on your machine and fails at publish.
2. **Signed and expiring URLs are a trap.** A link valid for one hour attached to a post scheduled for Wednesday will fail on Wednesday, not today. Host the asset somewhere stable.
3. **Instagram and TikTok require media, and TikTok requires video.** An Instagram post with `"mediaUrls": []` is accepted by the API and rejected by the platform. This is the single most common cause of a failed row in a mixed batch.
4. **Size, duration and aspect ratio limits belong to the platform, not to the API.** They are enforced at publish. The API cannot tell you in advance that a 4 minute video is too long for a given surface.

The array is per call, so every row a call creates shares one media set. If X and Threads need different images, they need separate calls, exactly as in Step 4.

One rule for the assistant: it may propose alt text and it may suggest what the image should show. It must never invent a URL. Paste every URL yourself.

## When one platform rejects and the others succeed

Because a batch is several posts rather than one, a failure is scoped. Four rows go out and one lands in `failed`.

Find them:

```json
{
  "name": "crm_list_social_posts",
  "arguments": { "status": "failed", "limit": 10 }
}
```

```json
{
  "count": 1,
  "posts": [
    {
      "id": 996,
      "platform": "instagram",
      "externalAccountId": "acc_zx91",
      "content": "Routing is the cheapest fix in support...",
      "scheduledAt": "2026-08-26T15:00:00Z",
      "status": "failed",
      "publishedUrl": null,
      "errorMessage": "Instagram requires at least one media item",
      "publishedAt": null
    }
  ]
}
```

The `errorMessage` and the empty media set explain this one without any further reading. Call `crm_get_social_post` for the full record when the cause is less obvious. The usual four: missing or expired media, a disconnected or re-authorised account, over-length text after the platform counted a link differently to your character count, and a duplicate-content rejection when two variants ended up identical.

Then do the recovery in this order:

1. Read the row back with `crm_get_social_post` and take `errorMessage` at face value.
2. Fix the cause outside the post: host the media somewhere stable, reconnect the account, shorten the copy.
3. Schedule a fresh post for that one account and leave the failed record alone as a log entry. `crm_update_social_post` will not revive it, because only a `pending` post is editable.

**Do not re-run the whole batch.** This is the production mistake that follows every partial failure. The four successful rows already published; running the batch again duplicates all four to fix one. Fix the failed row on its own.

Nothing in the MCP surface deduplicates that for you. `crm_send_social_message` and `crm_schedule_social_post` take no idempotency key, so a blind retry is a second publish. Four things stand between you and a double post, and all four are worth knowing by name. The schedule tool is annotated as an open world write, so the client asks before it runs. A rejection comes back as an error rather than as a silent retry, so a failure stays a failure until you act on it. `crm_schedule_social_post` checks each target account's daily post limit before it writes anything, returning the blocked accounts in `skipped` rather than half committing a fan out. And you list what already exists with `crm_list_social_posts` before you re-run anything, which is the only one of the four that depends on you.

Two boundaries worth knowing. `crm_cancel_social_post` only helps before a post goes out; once published there is nothing to cancel. And the MCP surface exposes cancel but not delete: a published post is never removed upstream from here. If something needs to come down from the platform, take it down on the platform.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Same text appeared on all five platforms | One call with all five in `platforms` | One call per distinct copy, group only identical text |
| Instagram or TikTok row failed | `mediaUrls` empty | Both require media, TikTok requires video |
| Media worked in testing, failed at publish | Signed URL expired between schedule and publish | Host the asset at a stable public URL |
| Post published three hours off | A local time written with a `Z` suffix, or a bare local time sent with no `timeZone` | Convert to UTC, or write the offset, or pass `timeZone`, see [03](./03-ai-content-calendar.md) |
| Threads posted twice | The old Threads row was not cancelled before the replacement was scheduled | Cancel the original row with `crm_cancel_social_post`, then read it back |
| X variant reads like a LinkedIn post | Variants drafted in one turn | Redraft X and Threads in a separate turn with their own rules |
| Duplicate-content rejection | Two variants ended up identical after edits | Re-read the character count column, rewrite one |
| `crm_update_social_post` not in the list | Key lacks `posts:write`, or `--read-only` is set | Check the key at the API keys page, then the client config |

## Exercise

Take one post you have already published on a single platform and rebuild it as five variants using the loop above, scheduled a week apart from the original so the comparison is not polluted. After both weeks, call `crm_social_post_stats` with `days` set to 14 to confirm all five actually went out, then open each platform's own analytics for the performance side. Post stats count publishing outcomes, not reach, so the comparison you care about comes from the networks themselves.

Then do the part that teaches you something: work out whether the winning variant won on copy, on timing, or on media. Write the answer into the style file from [03: AI Content Calendar](./03-ai-content-calendar.md) as one line of platform rule. Five of those lines and your variants stop needing much editing at all.

## Where to go next

- [05: Escalation and Human Handoff](./05-escalation-and-human-handoff.md) covers what happens when a cross posted idea generates replies faster than you can answer them.
- [06: Social Media Reporting With AI](./06-social-media-reporting-with-ai.md) turns `crm_social_post_stats` into a monthly comparison you can act on.
- [07: MCP Security Checklist](./07-mcp-security-checklist.md) covers scope minimisation, `--read-only`, and key rotation.
- [02: AI DM Triage](./02-ai-dm-triage-workflow.md) for the inbox side of the same server.
- Full tool reference and argument lists: [docs.crmsolid.com/integrations/mcp/](https://docs.crmsolid.com/integrations/mcp/)
- Package: [@crmsolid/mcp-server on npm](https://www.npmjs.com/package/@crmsolid/mcp-server) and [source on GitHub](https://github.com/CRM-Solid/crmsolid-mcp)
- Back to [the guide index](../README.md)
