# AI Content Calendar: Plan and Schedule a Week of Social Posts With One Prompt

An AI content calendar is a weekly planning session where your assistant reads what actually happened in your business last week, proposes a themed set of posts, writes them in your voice, and schedules the ones you approve. This tutorial builds that session over the Model Context Protocol against a CRM Solid MCP server, which holds the connections to LinkedIn, X, Instagram, Threads, TikTok and seven other platforms.

The session takes about 25 minutes on a Monday morning and produces five to seven scheduled posts. Everything the assistant writes lands at a parking time first, far enough out that you review before anything moves, which is the safety property the whole workflow is built on.

## The rule that keeps a bad week from going out

Read this before any of the steps.

**`scheduledAt` is required unless you pass `publishNow: true`. Omitting both is rejected with "scheduledAt is required unless publishNow is true". A post goes out immediately only when you pass `publishNow: true` explicitly.**

There is no draft state and no accidental publish. If the assistant forgets to say when, it gets an error, not a surprise post. This means you can let it call `crm_schedule_social_post` freely during a planning session, as long as you give it one parking time to use, and review afterwards. That is exactly how the session below is structured: generate everything at a parking time, review as a table, then move the survivors onto their real slots and cancel the rest.

## What you need before the first AI content calendar session

| Requirement | Detail |
|---|---|
| MCP client | Claude Desktop, Claude Code, Cursor, ChatGPT, or any MCP client |
| Server | `@crmsolid/mcp-server` over stdio |
| Scopes on the key | `posts:read`, `posts:write`, plus `social:read` and `analytics:read` for the input step |
| Connected accounts | At least two, so the plan has somewhere to go |
| A style file | Covered in the last section. Write it before your second session, not your first |

```jsonc
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "posts,social,analytics"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

If this is your first connection, start with [01: Connect Your First Social MCP Server](./01-connect-your-first-social-mcp-server.md). One naming note: MCP tool output is camelCase, while the v1 REST API underneath is PascalCase. Everything below is MCP output.

## Step 1: Gather what actually happened last week

A plan built from nothing reads like it was built from nothing. Start with three reads.

```text
Before we plan anything, gather inputs:
1. Call crm_social_post_stats with days 7.
2. Call crm_list_social_posts with status "published", fromDate 2026-08-17 and toDate 2026-08-24.
3. Call crm_social_inbox_summary.
Report each result plainly. Do not propose posts yet.
```

```json
{
  "name": "crm_social_post_stats",
  "arguments": { "days": 7 }
}
```

```json
{
  "days": 7,
  "from": "2026-08-17T09:12:00Z",
  "to": "2026-08-24T09:12:00Z",
  "total": 14,
  "published": 9,
  "pending": 4,
  "processing": 0,
  "failed": 1,
  "cancelled": 0,
  "lastPublishedAt": "2026-08-23T06:00:00Z",
  "platforms": [
    { "platform": "linkedin", "total": 6, "published": 4, "pending": 2, "failed": 0, "cancelled": 0 },
    { "platform": "x", "total": 5, "published": 3, "pending": 1, "failed": 1, "cancelled": 0 },
    { "platform": "instagram", "total": 3, "published": 2, "pending": 1, "failed": 0, "cancelled": 0 }
  ]
}
```

Note what is and is not in there. `crm_social_post_stats` counts publishing outcomes: what went out, what is queued, what failed. It carries no impressions, no reach and no engagement, so it answers "did the calendar hold" and not "did it work". Reach lives in each platform's own analytics. [06: Social Media Reporting With AI](./06-social-media-reporting-with-ai.md) covers where the line sits.

Then add the input no API has: what your team shipped. Paste it in as plain text.

```text
Shipped last week:
- CSV import for contacts, with column mapping
- Fixed the WhatsApp attachment bug that broke image sends
- Turkish translation for the reporting screens

Recurring questions in the inbox this week:
- can I import from a spreadsheet
- does it work in Turkish
```

**Verify.** You have four things in front of you: platform publishing counts, last week's published posts, inbox pressure, and shipped work. If the shipped list is empty, stop and go get it. A content calendar generated purely from what you published last week will produce posts about posting, which is the most reliable way to bore an audience.

## Step 2: Point the assistant at your style file

The assistant reads the style file before it writes anything. In Claude Code, name the path. In Claude Desktop, keep it in the project. In Cursor, keep it as a rule file.

```text
Read content-style.md before drafting. Every rule in it overrides your defaults.
Confirm in one line that you have read it and name the three banned words it lists.
```

**Verify.** The assistant names the three banned words correctly. If it paraphrases, it did not read the file, and the drafts in Step 4 will come back in generic voice. This one line check costs a second and saves a rewrite.

## Step 3: Generate the themed plan, not the posts

Separate planning from writing. A model asked to do both at once will write first and rationalise the theme afterwards.

```text
Using the inputs above, propose a plan for the week of 24 to 28 August.
Five posts, one theme for the week, one angle per post.

Constraints:
- At least two posts must come from the shipped list.
- At least one must answer a recurring inbox question.
- No post may repeat an angle used in last week's published posts.
- Weight the platforms by where the inbox summary shows the most conversations.
  Post stats carry no engagement, so do not weight on numbers we do not have.

Output a table: day, platform, angle, why this beats the alternative.
Do not write post copy yet.
```

Clients that support MCP prompts can use the packaged `weekly-content-plan` prompt, which collects the same inputs itself.

The plan that comes back should be arguable. If you cannot disagree with a row, the row is too vague to be a plan.

**Verify.** Five rows, one theme, no angle duplicated from last week. Check the "why this beats the alternative" column honestly: if every entry says something like "high engagement topic", the model is filling a column rather than making a case. Push back once, in the same turn, before moving on.

## Step 4: Turn the approved plan into pending posts

Now the writing. Notice the one argument you supply yourself.

```text
Write the five posts from rows 1 to 5 following content-style.md.
For each one, call crm_schedule_social_post with content, platforms, accountIds,
and scheduledAt exactly 2026-12-31T09:00:00Z. That is the parking slot.
Do NOT pass publishNow. Do NOT invent a real posting time yet.
Report the returned postIds and status for each.
```

```json
{
  "name": "crm_schedule_social_post",
  "arguments": {
    "content": "Three things we learned migrating 40 support inboxes.",
    "platforms": ["linkedin"],
    "accountIds": [12],
    "scheduledAt": "2026-12-31T09:00:00Z"
  }
}
```

```json
{
  "count": 1,
  "postIds": [993],
  "platforms": ["linkedin"],
  "scheduledAt": "2026-12-31T09:00:00Z",
  "status": "pending",
  "skipped": null,
  "message": "Scheduled on 1 account(s) for 2026-12-31 09:00 UTC."
}
```

`status` is `pending` and the time is months away. That is the rule working. `crm_schedule_social_post` is annotated openWorld because it can reach a platform, but with a parking time and no `publishNow` flag it will not, and a post that arrives with no time at all is rejected before anything is written. One row is created per target account, which is why `postIds` is an array.

**Verify.** Ask for the list back, filtered.

```json
{
  "name": "crm_list_social_posts",
  "arguments": { "status": "pending", "limit": 10 }
}
```

Five posts, every one still parked on 31 December. If any came back on a date this week, the assistant supplied a time you did not ask for. Cancel that one with `crm_cancel_social_post` and repeat the instruction with the word "not" in capitals, which sounds crude and works.

## Step 5: Review the pending posts as one table

Never review posts one at a time. You lose the sense of the week.

```text
Show all five pending posts as a table: id, day, platform, first 90 characters of
content, character count, and any claim in the post that needs a source.
Flag anything that repeats a phrase from another row.
```

The character count column is the one that catches problems. A 900 character post aimed at X is not a near miss, it is a different post. The repeated phrase check matters more than it sounds: five posts drafted in one turn tend to share an opening construction, and readers notice that faster than they notice the content.

**Verify.** Read the table, then read one full post at random against the style file. Approve or reject each row explicitly. "Looks good" is not an approval you can audit later.

## Step 6: Move the approved posts onto their real times

Moving is a separate call on purpose. `crm_update_social_post` is annotated idempotent, so a repeated call with the same time changes nothing. It edits only a `pending` post: anything published, failed or cancelled comes back as "Post 993 not found, or it is no longer editable (only pending posts can be edited)".

```text
Move posts 1, 2, 4 and 5 onto real times. Cancel 3, I want to rewrite it.
Times, all Europe/Istanbul local: 1 on Tuesday 09:00, 2 on Wednesday 09:00,
4 on Thursday 17:30, 5 on Friday 09:00.

For each, print the local time you were given AND the UTC instant you are about
to send, then call crm_update_social_post.
```

```json
{
  "name": "crm_update_social_post",
  "arguments": {
    "postId": 993,
    "scheduledAt": "2026-08-26T06:00:00Z"
  }
}
```

```json
{
  "postId": 993,
  "platform": "linkedin",
  "scheduledAt": "2026-08-26T06:00:00Z",
  "status": "pending",
  "message": "Post updated."
}
```

**Verify.** Call `crm_list_social_posts` with `status` set to `pending` and read every `scheduledAt`. Four on real slots, one cancelled and gone from this list, and each instant three hours earlier than the local time you asked for. Which brings us to the part that goes wrong most often.

## Time zones: an offset or a `timeZone`, never neither

Two fields look like they do the same job. They are two ways to say the same thing, and you need exactly one of them.

| Field | Format | What it controls |
|---|---|---|
| `scheduledAt` | ISO 8601. With a trailing `Z` or an offset it names an exact instant. Without one it is a wall clock reading | The moment the post goes out |
| `timeZone` | IANA name, for example `Europe/Istanbul` | Which clock a `scheduledAt` with no offset is read in, and how the calendar reads the post back to you. It is validated, and an unknown zone is rejected |

An offset on `scheduledAt` wins outright and `timeZone` is ignored for it. A bare wall clock with `timeZone` set is converted from that zone, which is what the field is for. The broken case is neither: a bare `2026-08-26T09:00:00` with no `timeZone` is read as UTC. That is the single most common scheduling bug, and it fails silently: the post goes out, just at the wrong hour, and you find out a week later.

**The wrong version.** You want Wednesday 26 August 2026 at 09:00 in Istanbul. You write:

```json
{
  "name": "crm_schedule_social_post",
  "arguments": {
    "content": "Three things we learned migrating 40 support inboxes.",
    "platforms": ["linkedin", "x"],
    "accountIds": [12, 15],
    "scheduledAt": "2026-08-26T09:00:00Z",
    "timeZone": "Europe/Istanbul"
  }
}
```

That call is valid. It is simply not what you meant. The `Z` already names the instant, so `timeZone` is ignored for it, and Istanbul runs at UTC+03:00. This publishes at 12:00 local, in the middle of lunch, three hours after your audience checked the feed.

**The right version.** Subtract the offset before you write the instant:

```json
{
  "postId": 993,
  "scheduledAt": "2026-08-26T06:00:00Z",
  "timeZone": "Europe/Istanbul"
}
```

09:00 Istanbul is 06:00 UTC. The `timeZone` field is unchanged, because it was never the problem. Two other forms say exactly the same thing if you prefer to write local times: the offset form `2026-08-26T09:00:00+03:00`, or a bare `2026-08-26T09:00:00` with `timeZone` set to `Europe/Istanbul`, which the server converts for you. All three store 06:00 UTC. The one to avoid is a bare `2026-08-26T09:00:00` with no `timeZone` at all, because that is read as UTC and puts you back at noon.

Two follow-on traps worth knowing:

1. **Do not build next week by adding seven days of seconds.** Istanbul stays at UTC+03:00 year round, but `Europe/London`, `Europe/Berlin` and `America/New_York` shift twice a year. Adding 604800 seconds across a daylight saving boundary moves your post by an hour. Compute each instant from the local time you actually want.
2. **Make the assistant show its arithmetic.** The instruction in Step 6 asks it to print the local time and the UTC instant before calling the tool. That single line turns a silent failure into something you can catch in one glance.

## Keeping the voice consistent with a style file

Voice drift is the reason most generated calendars get abandoned in week three. The fix is a file, not a longer prompt. Keep it at `content-style.md` next to your working directory and have the assistant read it before every drafting step, as in Step 2.

Put these in it, and nothing else:

- **Who you write for.** One sentence, naming a role and a problem, not a demographic. "Support leads who inherited an inbox they did not design" beats "SMB decision makers".
- **Three sentences you have already published** and would happily publish again. Real sentences do more for voice matching than any adjective list.
- **Banned words.** Yours, specific, at least five. Include the ones you personally cannot stand, because you are the one rejecting drafts.
- **Formatting per platform.** Hashtag count, whether you use line breaks, whether you open with a question, whether emoji are allowed at all.
- **The exact nouns for your product and features.** If it is a "workspace" and never an "account", write that down. Nothing marks generated copy faster than the wrong internal noun.
- **Person and number.** First person plural for company posts, first person singular for founder posts. Pick one per account and record it.
- **Numbers policy.** The most valuable line in the file: the assistant may only use numbers that appear in the inputs you pasted or in a stats response. No estimates, no rounded-up milestones, no invented percentages.

Keep the file under 40 lines. A style file long enough to need its own summary will be summarised, and the summary is where the voice goes.

Review it once a month, and add a rule every time you reject a draft for the same reason twice. That is the whole maintenance loop.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Post published immediately | `publishNow: true` was passed | State "do not pass publishNow" in the drafting prompt, and check every instant is in the future |
| The whole call was rejected | Neither `scheduledAt` nor `publishNow` was passed, or the time was in the past | Give the assistant the parking slot verbatim, as in Step 4 |
| Post went out three hours late | `scheduledAt` was written as local time with a `Z` | Convert to UTC before writing, and make the assistant print both values |
| A post is still parked on 31 December | `crm_update_social_post` was called without `scheduledAt`, or the post is no longer `pending` | Re-send the call with the instant included, then read the post back |
| Posts all sound the same | Style file not read, or all five written in one turn | Confirm the read as in Step 2, and draft in two turns of two or three |
| `crm_schedule_social_post` missing from the list | Key lacks `posts:write`, or the server runs with `--read-only` | Check scopes at the API keys page, then check the client config `args` |
| Plan ignores what you shipped | Shipped list was never pasted in | It is manual input, no tool provides it, add it to your Monday checklist |
| Post counted as `failed` in stats | Platform rejected it after acceptance | See [04: Cross Post to Multiple Social Networks](./04-cross-posting-without-copy-paste.md) for handling partial failures |
| Wrong account posted | `accountIds` omitted, so a default was used | Call `crm_list_social_accounts` first and pass ids explicitly |

## Exercise

Run one full session and schedule four posts. The following Monday, before you plan anything, call `crm_social_post_stats` with `days` set to 7 and check that the calendar actually held: four planned, four published, none failed, none still parked. Then do the harder part, which needs a second source: open each platform's own analytics, note how the four performed, and work out whether the weakest one lost on the angle, the platform, the time, or the copy. Write the answer as one new line in `content-style.md`.

Do that four weeks running and the style file becomes the most valuable asset in the workflow. The prompt is replaceable, the file is not.

## Where to go next

- [04: Cross Post to Multiple Social Networks Without Copy and Paste](./04-cross-posting-without-copy-paste.md) takes one idea from this calendar and adapts it per platform instead of duplicating it.
- [02: AI DM Triage](./02-ai-dm-triage-workflow.md) applies the same approve-then-write pattern to the inbox.
- [05: Escalation and Human Handoff](./05-escalation-and-human-handoff.md) covers the replies a published post generates, and who owns them.
- [06: Social Media Reporting With AI](./06-social-media-reporting-with-ai.md) closes the loop on measurement.
- [07: MCP Security Checklist](./07-mcp-security-checklist.md) covers scopes and key handling before this runs on a shared machine.
- Full tool reference: [docs.crmsolid.com/integrations/mcp/](https://docs.crmsolid.com/integrations/mcp/)
- Package: [@crmsolid/mcp-server on npm](https://www.npmjs.com/package/@crmsolid/mcp-server) and [source on GitHub](https://github.com/CRM-Solid/crmsolid-mcp)
- Back to [the guide index](../README.md)
