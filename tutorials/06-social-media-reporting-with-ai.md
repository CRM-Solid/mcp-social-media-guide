# Social Media Reporting With AI: A Monthly Review You Will Actually Read

Social media reporting with AI works when you make the assistant produce the numbers first and the story second. Pull post stats, messaging stats and the inbox summary over MCP, put them in a table with the window and the tool call that produced each row, compare against the previous period, and only then ask for a narrative. Done in that order, a monthly review takes about fifteen minutes and survives being questioned. Done in the other order, you get fluent paragraphs built on numbers nobody checked.

This tutorial assumes your MCP server is connected. If not, start with [connect your first social MCP server](./01-connect-your-first-social-mcp-server.md). One convention: MCP tool output is camelCase, the v1 REST API behind it is PascalCase, and every JSON block here is MCP output.

## What a monthly report has to answer

Four questions. If a section of your report does not answer one of them, cut it.

1. Did we publish what we said we would publish?
2. Did anything fail on the way out, and on which account?
3. Did we answer more, and did the replies actually land?
4. How many conversations turned into something the business cares about?

Everything else is context for those four. A report that opens with follower growth is answering a question nobody asked.

## Pull the numbers before you ask for a story

Four tool calls give you the entire dataset. Run them, keep the raw output, and do not let the assistant paraphrase any of it yet.

**Publishing outcomes.** `crm_social_post_stats` takes a single `days` argument (1 to 365, default 30). Call it twice, once with `days: 30` and once with `days: 60`. It counts what went out and what did not: `total`, `published`, `pending`, `processing`, `failed`, `cancelled`, the same breakdown per platform, and `lastPublishedAt`.

**The posts themselves.** `crm_list_social_posts` accepts `status`, `platform`, `fromDate`, `toDate` and `limit` (1 to 100, default 25). Note the date arguments are `fromDate` and `toDate`, and use them for exact calendar boundaries. There is no cursor on this surface: if a window holds more than 100 rows, narrow it by platform or split the dates. Cursor pagination (`items`, `nextCursor`, `hasMore`, `after`) belongs to the v1 REST API, and mixing the two up is the easiest way to write a script that silently reports one page.

```json
{
  "id": 993,
  "platform": "linkedin",
  "externalAccountId": "acc_zx91",
  "content": "Three things we learned migrating 40 support inboxes.",
  "scheduledAt": "2026-08-05T09:00:00Z",
  "status": "published",
  "publishedUrl": "https://www.linkedin.com/feed/update/urn:li:share:7100000000000000000",
  "errorMessage": null,
  "publishedAt": "2026-08-05T09:00:11Z"
}
```

**Outbound messaging.** `crm_messaging_stats` takes `windowDays`, which accepts only 1, 7 or 30 (default 7), plus an optional `accountId`. It reads the outbound send queue, so it answers how many messages you tried to send and how many landed.

```json
{
  "windowDays": 30,
  "accountId": null,
  "since": "2026-07-25T09:12:00Z",
  "totals": { "queued": 12, "sent": 1980, "failed": 24, "total": 2016 },
  "successRate": 98.2,
  "generatedAt": "2026-08-24T09:12:00Z"
}
```

**The inbox right now.** `crm_social_inbox_summary` takes no arguments and returns the current state, not the period. `awaitingReply` holds at most ten conversations, oldest first, so read it as a sample of the backlog rather than a count of it.

```json
{
  "accounts": 4,
  "conversations": 132,
  "activeConversations": 34,
  "archivedConversations": 98,
  "unreadConversations": 11,
  "unreadMessages": 19,
  "lastMessageAt": "2026-08-24T08:41:12Z",
  "platforms": [
    { "platform": "instagram", "conversations": 71, "unreadConversations": 7, "unreadMessages": 12 },
    { "platform": "linkedin", "conversations": 38, "unreadConversations": 3, "unreadMessages": 5 },
    { "platform": "x", "conversations": 23, "unreadConversations": 1, "unreadMessages": 2 }
  ],
  "awaitingReply": []
}
```

That last distinction gets fumbled constantly. The inbox summary is a snapshot taken at the moment you call it, so it does not belong in a month-over-month table. Put it in its own section, labelled with the timestamp.

**Outcomes.** `crm_dashboard_summary` and `crm_list_deals` for what the conversations produced, plus `crm_top_contacts` if you want named accounts.

## Getting the windows right

The comparison is where most reports quietly break.

`crm_social_post_stats` gives you rolling windows, not calendar months. A 30 day window ending today is not August. If your stakeholders think in months, use `fromDate` and `toDate` on `crm_list_social_posts` for the post list and say clearly at the top of the report which window every number covers.

You can derive the previous period by subtracting the 30 day result from the 60 day result, but only for additive counters: posts published, failed, cancelled. Never subtract a rate, a median or an average. Subtracting a rate from a rate produces a meaningless number, and an assistant will happily do it if you do not forbid it. A rate for the previous period is recomputed from that period's own counts, never subtracted.

Second trap: `crm_messaging_stats` has no 60 day window at all. `windowDays` accepts 1, 7 or 30 and nothing else, so there is no larger call to subtract from. The previous month's messaging numbers can only come from the report you saved last month, which is the real reason the last section tells you to keep every table as a file.

Third trap: a post row exists per target account. One call that fans a single piece of copy to LinkedIn and X is two rows in `total`, not one. Count rows as delivery attempts, not as ideas, and say which you mean when a stakeholder asks how many posts went out.

## What these tools do not measure

Say this out loud before your first report, because it decides half the table.

**Nothing on this surface returns impressions, reach, engagement or follower counts.** `crm_social_post_stats` counts publishing outcomes: what went out, what is queued, what failed, per platform. It answers "did the calendar hold", never "did it work". There is no tool here that will tell you how many people saw a post. Those numbers live in each platform's own analytics: LinkedIn page analytics, X analytics, the Instagram professional dashboard, TikTok analytics, and the equivalents elsewhere. Export them for the same window, paste them in as a clearly labelled second source with its own date range, or leave the row reading "not available". Both are honest. Quietly implying that publishing more meant reaching more is not.

**Response time is not exposed either.** `crm_messaging_stats` reads the outbound send queue, so it gives you attempted sends, successful sends, failures and a success rate. It does not give you the gap between a customer's message and your reply, and no other tool here does. If you need median or p90 first response, you compute it yourself from the `sentAt` timestamps that `crm_list_social_messages` returns per conversation, or take it from the panel's own analytics screens. Either way, record in the report which of the two produced the number, because they will not always agree.

The discipline in the rest of this tutorial depends on this section. A metric with no source is not a metric, and "not available" is a legitimate cell in a table.

## The exact prompt

Paste this. The constraints are the whole point of it.

```text
Produce the monthly social report for the window 2026-07-25 to 2026-08-24,
comparing against 2026-06-25 to 2026-07-25.

STEP 1. Numbers only. Output a single markdown table with these columns:
metric | this window | previous window | change | source.
The source column must name the exact tool call and arguments that produced
the value, and the window it covers. For any value you derived rather than
read directly, write "derived: <the arithmetic>".
For any metric you cannot obtain, write "not available" and say where it would
have to come from. Reach, impressions, engagement and response time are not in
this dataset: never substitute a number for them. Do not estimate. Do not fill
a gap with an industry average.
Do not write a single sentence of commentary in this step.

STEP 2. Stop and wait for me to confirm the table.

STEP 3. After I confirm, write the narrative. Rules:
- Every claim must point at a row in the table.
- If two readings of a number are possible, give both.
- Separate what changed from why it changed, and mark every "why" as a
  hypothesis unless a number in the table supports it.
- Name the one thing that got worse. If nothing got worse, say the data does
  not show a regression, and do not invent one.
- Maximum 400 words.

STEP 4. List up to three actions for next month. Each action names the metric
it is meant to move and the number it has to beat.
```

Two constraints do most of the work. The source column stops fabrication, because a model that has to name the tool call cannot invent a number no tool returned. The stop after step one keeps the story from steering the arithmetic.

## Make the assistant show its work

Three habits, applied every month.

**Demand the table before any narrative.** Not a summary with numbers in it. A table, one metric per row, with the source. If the assistant leads with prose, reject it and re-send step one alone.

**Ask for the query behind any number you would repeat out loud.** "Which call produced 98.2 percent, with which arguments, over which window?" A correct answer names the tool, the arguments, the window, and the arithmetic. A wrong answer restates the number more confidently. That is your tell.

**Reject any claim the data does not support.** This part takes discipline, because the unsupported claims are usually the interesting ones.

| The assistant writes | Why it fails | What to accept instead |
|---|---|---|
| "Reach improved because we posted more video" | Nothing in this dataset records reach, and nothing records format | "We published 24 posts against 19. Reach is not in this dataset, so pull it from each platform's analytics before claiming it moved" |
| "Response times improved thanks to the assistant" | No response time in the dataset, no control group, and staffing changed too | "Outbound sends rose from 1,410 to 1,980 with the success rate flat. Whether anyone was answered faster is not measured here" |
| "LinkedIn is our best channel" | Best by which measure, over what window | "LinkedIn produced the most conversations that became deals: 9 of 17" |
| "Follower growth is up 4.2 percent" | True and irrelevant unless it moved something | Cut it, or pair it with the metric it was supposed to move |
| "We expect this trend to continue" | A forecast from two data points | Cut it |

One month of two windows is not a trend. Say so in the report. Three windows is a direction. Six is a trend.

## Where social media reporting with AI goes wrong

Three metrics look like performance and are not.

**Follower count.** Not an outcome, moved by things you do not control, inflated by accounts that will never message you. Interesting only when paired with a conversion: if followers rose 4 percent and conversations from that platform did not move, the number is telling you the audience is the wrong audience.

**Raw impressions.** Not in this dataset at all, so the first mistake is importing it from a platform dashboard and dropping it into the same table without a label. Once it is in, the second mistake follows: one post that travels can carry a third of a month's impressions and tell you nothing repeatable. Report the median per post next to the total. When the mean is double the median, the month was one lucky post, and next month will look like a collapse for no reason.

**Containment rate.** The share of conversations closed without a human is the easiest metric in support to game, because a conversation the customer gave up on counts as contained. It rewards silence. If you track it, always show it next to the seven day reopen rate and the escalation rate. Neither of those comes from a tool here either, which is exactly the point: if you cannot measure the counterweight, do not publish the flattering half on its own.

The pattern behind all three: each one measures activity or absence, not resolution.

## What to track instead

| Metric | Definition | Why it survives scrutiny |
|---|---|---|
| Median first response time | Minutes from an inbound message to your first human or assistant reply, median and p90 | The single strongest driver of whether a DM converts. The p90 exposes the ignored tail that a mean hides. Not returned by any tool here: compute it from `crm_list_social_messages` timestamps and say so |
| Publish reliability | Posts published divided by every attempt in the window, published plus failed plus cancelled | Catches a broken calendar before an executive does, and comes straight out of `crm_social_post_stats` |
| Failures by platform | Failed rows per platform | One broken account hides inside a healthy total. This is where it surfaces |
| Cadence by platform | Posts published per platform per window | Separates a plan that survived the week from one that quietly became a LinkedIn-only plan |
| Outbound send success rate | `successRate` from `crm_messaging_stats` | Catches a channel that is accepting your replies and not delivering them |
| Unanswered backlog | `unreadConversations` and `unreadMessages`, read at the same hour every month | A snapshot, not a period, but a comparable one if the time is fixed |
| Conversations that became deals | Count of conversations linked to a created deal | Ties the inbox to revenue and cannot be gamed by closing threads |
| Escalation rate and time to human | Share of conversations handed to a named person, and how long it took | Rising escalations means the automated path is failing. See [escalation and human handoff](./05-escalation-and-human-handoff.md) |

Pick five. A report with seven numbers gets read. A report with thirty gets filed.

## A worked example report

Example only. Every figure below is fictional and exists to show the shape and the internal consistency you should demand. Do not benchmark against it.

**Window:** 2026-07-25 to 2026-08-24. **Compared against:** 2026-06-25 to 2026-07-25. Both rolling 30 day windows, not calendar months.

| Metric | This window | Previous | Change | Source |
|---|---|---|---|---|
| Posts published | 24 | 19 | +26.3% | `crm_social_post_stats` days 30; previous derived: 43 (days 60) minus 24 |
| Posts failed | 3 | 7 | -57.1% | `crm_social_post_stats` days 30; previous derived: 10 (days 60) minus 3 |
| Posts cancelled | 1 | 2 | -50.0% | `crm_social_post_stats` days 30; previous derived: 3 (days 60) minus 1 |
| Publish reliability | 85.7% (24 of 28) | 67.9% (19 of 28) | +17.8pp | derived: published / (published + failed + cancelled), each window computed from its own counts |
| Reach, impressions, engagement | not available | not available | not available | not returned by any tool here. Platform analytics, not exported for this window |
| Outbound messages attempted | 2,016 | 1,445 | +39.5% | `crm_messaging_stats` windowDays 30, `totals.total`; previous from the July report file |
| Outbound messages sent | 1,980 | 1,410 | +40.4% | `crm_messaging_stats` windowDays 30, `totals.sent`; previous from the July report file |
| Outbound send success rate | 98.2% | 97.6% | +0.6pp | `crm_messaging_stats` windowDays 30, `successRate` |
| Median first response | not available | not available | not available | no tool returns it. Would have to be computed from `crm_list_social_messages` timestamps |
| Conversations that became deals | 17 | 11 | +54.5% | `crm_list_deals` filtered to social source |
| Escalations | 19 | not available | not available | task list, `[ESC]` titles. Previous window not tracked |

Posts published by platform, this window: LinkedIn 9, Instagram 7, X 5, Threads 3. Previous window: LinkedIn 8, Instagram 6, X 4, Threads 1. All three failures this window were Instagram rows; the single cancellation was LinkedIn.

Inbox snapshot, taken 2026-08-24T10:15Z, not part of the comparison: 132 conversations, 34 active, 11 unread conversations, 19 unread messages, and 10 entries in `awaitingReply`, which is the cap the tool returns rather than the true backlog.

**Narrative.**

The calendar held better than last month. We published 24 posts against 19, failures fell from 7 to 3, and reliability rose from 67.9 to 85.7 percent of attempts. On 28 attempts in both windows, that is five fewer things going wrong for the same amount of effort.

Whether more people saw any of it is not in this report. `crm_social_post_stats` counts publishing outcomes and nothing else, and nobody exported platform analytics for this window. Read every reach row as absent, not as flat. Publishing more is not the same claim as reaching more, and nothing in this table supports the second one.

What got worse: Instagram. Three of its ten attempts failed, and no other platform failed once. Three rows is a small number, but all three had the same cause, an empty media set on a network that requires media, so it repeats until the workflow changes rather than until the platform does.

Outbound messaging grew and kept its quality. Attempts rose 39.5 percent and successful sends 40.4 percent, with the success rate holding at 98.2 percent, so the extra volume did not come at the cost of delivery. Whether customers were answered faster is a different question and this dataset cannot answer it: `crm_messaging_stats` reads the send queue, not the gap between an inbound message and the reply to it.

Deals attributed to social rose from 11 to 17. There is deliberately no conversion rate here. A rate needs conversations opened inside the window, and the inbox summary is a snapshot rather than a period count, so the denominator does not exist yet. Example pipeline value: EUR 34,200 across 17 conversations, average EUR 2,012.

Not comparable this month: escalations were not tracked in the previous window, so the 19 figure has no baseline. The messaging rows compare against last month's saved file rather than a derived window, because `crm_messaging_stats` accepts only 1, 7 or 30 day windows and there is no 60 day call to subtract from.

**Actions for next month.**

1. Take Instagram failures from 3 to 0 by making a media URL a required field before a post is scheduled to that account.
2. Hold publishing at 24 and keep reliability above 85.7 percent, this window's figure.
3. Export reach for this window from each platform's own analytics into the same file, so the report can start answering whether anyone saw the posts. Today that row is empty, so any number beats it.

## Running it every month

Save the prompt in your repository, not in a chat history. Run it on the same day every month: a variable run date makes windows incomparable and hides weekly seasonality.

Keep every month's table as a plain markdown file. This is not tidiness, it is the only baseline some of these numbers will ever have: `crm_messaging_stats` cannot look back further than 30 days, and any figure you import from platform analytics is gone from the dashboard's default view within a quarter. After three months, paste all three files into the assistant and ask a question no single month can answer: which metrics move together, and which one moves first. That is where reporting starts changing decisions instead of describing them.

Two habits keep the numbers trustworthy. Run the reporting session read-only, so a reporting pass can never send a message or reschedule a post:

```jsonc
{
  "mcpServers": {
    "crmsolid-report": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--read-only", "--tools", "social,posts,analytics,deals"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

And keep the key used for reporting on read scopes only (`social:read`, `posts:read`, `analytics:read`). A reporting key that can post is a reporting key that will eventually post. [The MCP security checklist](./07-mcp-security-checklist.md) covers the rest.

## Related reading

- [Connect your first social MCP server](./01-connect-your-first-social-mcp-server.md)
- [The AI DM triage workflow](./02-ai-dm-triage-workflow.md)
- [Build an AI content calendar](./03-ai-content-calendar.md)
- [Cross-posting without copy and paste](./04-cross-posting-without-copy-paste.md)
- [Escalation and human handoff](./05-escalation-and-human-handoff.md)
- [MCP security checklist](./07-mcp-security-checklist.md)
- [Repository index](../README.md)

External references: the [MCP specification](https://modelcontextprotocol.io), the [Pinlyx MCP docs](https://docs.pinlyx.com/integrations/mcp/), the [npm package](https://www.npmjs.com/package/@crmsolid/mcp-server) and its [source repository](https://github.com/CRM-Solid/crmsolid-mcp).
