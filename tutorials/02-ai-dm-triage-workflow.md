# AI DM Triage: Build a Morning Routine That Clears a Social Inbox

AI DM triage means handing your unanswered social messages to an assistant, having it sort them into a few fixed categories, draft the easy replies, and return a table you approve before anything is sent. This tutorial builds that routine end to end over the Model Context Protocol. It uses the CRM Solid MCP server as the bridge to Instagram, LinkedIn, X and nine other platforms, but the shape of the routine works against any MCP server that exposes a social inbox.

You finish with one saved prompt, a short list of things the assistant is never allowed to answer, and two numbers that tell you whether the routine earned its place.

## What you need before the first run

| Requirement | Detail |
|---|---|
| MCP client | Claude Desktop, Claude Code, Cursor, ChatGPT, or any client that speaks MCP |
| Server | `@crmsolid/mcp-server` over stdio |
| Scopes on the key | `social:read`, `social:write`, plus `tasks:write` for escalations |
| Connected accounts | At least one social account linked in the CRM |
| Time | About 20 minutes to build, about 6 minutes a day to run |

If the server is not connected yet, work through [01: Connect Your First Social MCP Server](./01-connect-your-first-social-mcp-server.md) first. The client config is short:

```jsonc
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "social,tasks,contacts"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

The `--tools` filter runs inside the local proxy, before the client ever sees a tool list. A family you leave out is not listed and not callable, so the assistant cannot wander into finance data during an inbox pass.

One naming note before the JSON starts: MCP tool output is camelCase, while the v1 REST API behind it is PascalCase. Every example below is MCP output, so every field is camelCase.

## Step 1: Read the summary before you read a single message

Start with the cheapest call in the set. It tells you whether today needs 6 minutes or 30.

```text
Call crm_social_inbox_summary and show me the result as a short list.
Do not open any conversations yet.
```

```json
{
  "name": "crm_social_inbox_summary",
  "arguments": {}
}
```

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
  "awaitingReply": [
    { "conversationId": 4821, "platform": "instagram", "participantName": "Dilara K.", "contactId": 91043, "unreadCount": 2, "lastMessageAt": "2026-08-24T08:41:12Z", "lastMessagePreview": "is the 12 month plan still available?" }
  ]
}
```

Read `awaitingReply` as the conversations sitting unanswered, oldest first, capped at ten. Those are the ones that decide your order of work today, whatever platform they are on, and `unreadConversations` tells you how much is behind them.

**Verify.** You got the count fields, a `platforms` array and an `awaitingReply` array. If the call fails with a scope error, the key is missing `social:read`. Regenerate it at `https://app.crmsolid.com/settings/developers` and restart the client so the proxy picks up the new key.

## Step 2: Pull one small, filtered queue

Do not ask for the whole inbox. Ask for the slice you will actually finish.

```text
Call crm_list_social_conversations with status "active" and limit 8.
List the results as a numbered table: id, platform, participantName, unreadCount,
lastMessageAt, lastMessagePreview. Nothing else.
```

```json
{
  "name": "crm_list_social_conversations",
  "arguments": { "status": "active", "limit": 8 }
}
```

```json
{
  "count": 3,
  "conversations": [
    {
      "id": 4821,
      "platform": "instagram",
      "participantName": "Dilara K.",
      "participantUsername": "dilarak",
      "contactId": 91043,
      "unreadCount": 2,
      "status": "active",
      "lastMessageAt": "2026-08-24T08:41:12Z",
      "lastMessageOutgoing": false,
      "lastMessagePreview": "is the 12 month plan still available?"
    },
    {
      "id": 4822,
      "platform": "x",
      "participantName": "Ops Weekly",
      "participantUsername": "opsweekly",
      "contactId": 88120,
      "unreadCount": 1,
      "status": "active",
      "lastMessageAt": "2026-08-24T07:02:44Z",
      "lastMessageOutgoing": false,
      "lastMessagePreview": "third day with no answer on ticket 4417"
    },
    {
      "id": 4823,
      "platform": "instagram",
      "participantName": "growth.tips.daily",
      "participantUsername": "growthtipsdaily",
      "contactId": 90887,
      "unreadCount": 1,
      "status": "active",
      "lastMessageAt": "2026-08-24T05:19:08Z",
      "lastMessageOutgoing": false,
      "lastMessagePreview": "check my page for free followers"
    }
  ]
}
```

A conversation is `active` or `archived`, never "open". The MCP list tools take `limit` (1 to 100, default 25) and return a named array plus a `count`, with no cursor to pass back: you narrow with `platform`, `status`, `contactId` or `unreadOnly` instead of paging. Cursor pagination (`items`, `nextCursor`, `hasMore`, `after`) is the v1 REST API's, not this surface's.

**Verify.** Count the rows in the table the assistant printed against `count`. If they differ, the assistant summarised instead of listing. Say "list every row, do not summarise" and run it again. This is the earliest place sloppiness shows up, and it is worth catching here rather than three steps later.

## Step 3: Classify in one pass, with four categories and no others

Four categories is the right number. Two is too coarse to act on, seven produces arguments about edge cases.

```text
For each conversation in the table, call crm_list_social_messages with that
conversationId and limit 6. Then classify each conversation as exactly one of:
question, complaint, lead, spam.

Rules:
- lead means they asked about buying, pricing, plans, a demo, or availability.
- complaint means they are reporting a failure, a delay, or a broken promise.
- question means everything else that needs an answer.
- spam means promotion, follower schemes, or a bot.
If two apply, pick the one with the higher business cost of being wrong.

Output a table: id, platform, participantName, category, one line reason.
Do not draft anything yet. Do not send anything.
```

```json
{
  "name": "crm_list_social_messages",
  "arguments": { "conversationId": 4821, "limit": 6 }
}
```

```json
{
  "conversationId": 4821,
  "platform": "instagram",
  "count": 1,
  "messages": [
    {
      "id": 88213,
      "direction": "inbound",
      "senderName": "Dilara K.",
      "text": "is the 12 month plan still available?",
      "attachmentUrl": null,
      "attachmentType": null,
      "transcript": null,
      "translation": null,
      "status": "delivered",
      "sentAt": "2026-08-24T08:41:12Z"
    }
  ]
}
```

`direction` is `inbound` or `outbound`, and nothing else. To walk back through a long thread, pass `beforeMessageId` with the oldest id you already have: history pages backwards by id, not by cursor.

**Verify.** Every row has exactly one category and a reason short enough to read at a glance. Spot check the two rows you would have classified differently. If the assistant labelled the ticket 4417 message as `question` rather than `complaint`, your rules are not explicit enough about the word "third day".

## Step 4: Draft only the easy ones

Now the assistant writes. Restrict it to the categories where a wrong reply is cheap.

```text
Draft replies for the rows classified question or lead only.
Skip complaint and spam entirely.

Each draft must:
- be under 60 words
- answer the actual question asked, not a general version of it
- use the contact's first name once
- contain no price, no discount, no promise about a date
- end without a call to action if the message was purely informational

Show the drafts in a table: id, participantName, category, draft text.
Still do not send.
```

Clients that support MCP prompts can use the packaged `dm-reply-draft` prompt instead, which takes a numeric `conversationId` and an optional `tone` (friendly by default, or professional or urgent). It pulls the message history itself, which saves a turn per conversation and keeps the draft anchored to the real thread rather than to your summary of it. It drafts and never sends, so it is safe to run across the whole queue.

**Verify.** Read every draft against the message it answers. The failure mode is a fluent reply to a question nobody asked. If a draft contains a number you did not supply, delete it and rewrite that one by hand: the model invented it.

## Step 5: Approve the table, then send

This is the only step that touches another person's inbox, so it gets its own turn and its own confirmation.

```text
Send drafts 1 and 4 exactly as written. Do not send 2 or 3.
For each send, call crm_send_social_message with the conversationId and the text.
Send them one at a time and report each result before starting the next.
```

```json
{
  "name": "crm_send_social_message",
  "arguments": {
    "conversationId": 4821,
    "text": "Hi Dilara, yes, the 12 month option is still available. I can send you the current terms and set it up on your account today if you want to go ahead."
  }
}
```

```json
{
  "status": "sent",
  "messageId": 88214,
  "conversationId": 4821,
  "platform": "instagram",
  "contactId": 91043,
  "externalMessageId": "aWdfZG1fMTo...",
  "sentAt": "2026-08-24T09:02:41Z",
  "message": "Message sent on instagram to Dilara K."
}
```

The result is a confirmation of what changed, not a data feed. That is deliberate across the whole server: no tool both reads and writes, so a write can never quietly hand the model a fresh pile of context to act on. `crm_send_social_message` is annotated openWorld and non-idempotent, and it takes no idempotency key, so nothing deduplicates a retry for you. What protects you instead: the annotation makes your client ask before each send, and a platform rejection comes back as an error (for example "The platform rejected this message: outside the 24 hour window (code 10)") rather than as a silent retry. Send one at a time and read each result, and there is nothing to retry blindly.

Two side effects are worth knowing before the first send. The send marks an operator takeover, which pauses the CRM's own AI agent for that contact so a bot cannot talk over you, and it writes a message activity to the contact timeline.

**Verify.** Open one of the sent conversations in the CRM or on the platform itself and read the outbound message. Do this on every one of your first five runs. After that, spot check one per week.

## Step 6: Turn everything else into a task

Anything not sent must leave the inbox with an owner and a due time. Otherwise the routine just moves the backlog around.

```text
For every row classified complaint, create a task with crm_create_task.
Title: "Reply: <participantName> (<platform>)".
Include the conversation id and the last message text in the description.
Due in 2 hours. Assign to me.
Do not create tasks for spam.
```

```json
{
  "name": "crm_create_task",
  "arguments": {
    "title": "Reply: Ops Weekly (x)",
    "description": "conversation 4822: third day with no answer on ticket 4417",
    "dueAt": "2026-08-24T11:30:00Z"
  }
}
```

The task family is documented alongside every other tool at [docs.crmsolid.com/integrations/mcp/](https://docs.crmsolid.com/integrations/mcp/). Check the argument list there before scripting this step, since the task tools predate the social ones and carry their own fields.

**Verify.** Task count equals complaint count. If your task list is empty on a day with complaints, the assistant treated "do not create tasks for spam" as "do not create tasks".

## Step 7: Mark read, and only what you handled

```text
Call crm_mark_social_conversation_read for every conversation you replied to
or created a task for. Do not mark the spam rows read, I will block those
manually. Confirm each one.
```

```json
{
  "name": "crm_mark_social_conversation_read",
  "arguments": { "conversationId": 4821 }
}
```

```json
{
  "conversationId": 4821,
  "unreadCount": 0,
  "message": "Conversation marked read."
}
```

This tool is annotated idempotent, so a repeat call is harmless.

**Verify.** Call `crm_social_inbox_summary` again. `unreadConversations` should have dropped by exactly the number you marked. If it dropped by more, something else marked conversations read while you worked, which usually means a colleague is in the same inbox.

## The saved prompt you end up with

Paste this into your client as a saved prompt, a Claude Code slash command, or a Cursor rule. It is the seven steps compressed into one turn with a hard stop before any write.

```text
Run the morning inbox pass.

1. Call crm_social_inbox_summary. Report active, unread, and everything in
   awaitingReply.
2. Call crm_list_social_conversations with status "active" and limit 8.
3. For each one, call crm_list_social_messages with limit 6.
4. Classify each as exactly one of: question, complaint, lead, spam.
5. Draft replies for question and lead rows only. Under 60 words. No price,
   no discount, no date promise. Never draft for complaint or spam.
6. HARD STOP. Print one table: row, id, platform, participantName, category,
   proposed action (send / task / ignore), and the draft text.
   Do not call any write tool in this turn.

Escalate instead of drafting if the message contains any of: refund, chargeback,
invoice, lawyer, legal, GDPR, data deletion, cancel my contract, or if the tone
is angry. Mark those "task" and say why.

Wait for me to name the rows to send.
```

The hard stop at line 6 is the whole design. Everything before it is reading, everything after it needs my sign off.

## Making the model behave

Three habits do most of the work.

**Be explicit about the format, not just the task.** "Classify these" produces prose. "Output a table with these six columns, one row per conversation, no summary" produces a table you can act on. Name the categories, name the columns, name what to skip. A model that is guessing your format is also guessing your standards.

**Keep batches small.** Eight conversations per pass, five while you are still tuning. Small batches keep the message history inside a comfortable context window, keep review from becoming skimming, and limit the blast radius when the rules are wrong. Twenty rows of drafts is not a review, it is a rubber stamp.

**Always ask for a table of proposed actions before any send.** This is not politeness, it is the enforcement point. The read tools and the write tools are separate on purpose, and the table is where you exercise that separation. It also makes the diff visible: you see "send" next to a draft, and you can strike one line without unpicking a paragraph of reasoning. If you want the guarantee enforced by the software rather than by the prompt, run a second server entry with `--read-only`, which drops every write tool locally before the client sees the list, and switch to the writable entry only for the send turn.

## Measuring whether AI DM triage actually helped

Track two numbers. Both are cheap.

**First response time, before and after.** Take a baseline for one week before you start: for each inbound message, how long until an outbound one. Then compare. The honest comparison is median, not mean, because one weekend message will wreck the average. `crm_messaging_stats` gives you the volume side, and the length of `awaitingReply` from the summary call is a daily proxy you can log by hand in thirty seconds, remembering it stops counting at ten. A routine that does not move median first response time is not saving you anything, it is just relocating the work.

**Share of drafts sent unedited.** Count it for two weeks. Below 40 percent means the assistant does not have the context it needs: your instructions are too vague, or the drafts are answering a generic version of each question. Above 90 percent means one of two things, and you need to know which. Either your inbox is genuinely repetitive, in which case those replies belong in a saved reply or an automation rather than a model, or you have stopped reading. Test by planting one deliberately wrong draft in your own review and seeing whether you catch it.

The pair matters more than either number. Falling response time with a falling edit rate is a working routine. Falling response time with a rising edit rate means you are typing faster, not working less.

## What not to automate, ever

Three categories never get a drafted reply, no matter how good the model looks that week.

**Anything involving money.** Refunds, discounts, price quotes, invoice disputes, billing dates. A model that invents a discount has made a commitment your customer will screenshot. Keep prices out of the draft rules entirely, as in Step 4, and let the assistant route these to a task.

**Anything with legal weight.** Words like lawyer, legal, GDPR, data deletion, chargeback, breach, or cancel my contract change who should be answering. The right response is a task with a two hour due time, not a friendly paragraph. A polite wrong answer in writing is worse than a slow right one.

**Angry customers.** An upset person can tell when a reply was assembled rather than written. Even a technically correct answer reads as dismissive when the tone is off. Route it to a human, and have the human open with the acknowledgement the model would have skipped.

The pattern behind all three: automate where the cost of being wrong is a slightly awkward sentence. Escalate where the cost of being wrong is money, a legal position, or a relationship. [05: Escalation and Human Handoff](./05-escalation-and-human-handoff.md) turns this into a rule set with owners and timers.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `crm_send_social_message` is not in the tool list | Key lacks `social:write`, or the server is running with `--read-only` | Check the key's scopes, then check the `args` array in your client config |
| Tool list is empty after editing the config | JSON syntax error, or the client was not restarted | Validate the file, then fully quit and reopen the client |
| Duplicate replies in one conversation | A send was retried blind after a timeout, and the tool takes no idempotency key | Send one at a time, read each result, and check the thread with `crm_list_social_messages` before resending |
| Assistant sends before you approve | The approval turn was in the same message as the send instruction | Split into two turns, and keep the HARD STOP line in the saved prompt |
| Drafts mention prices you never gave | The model filled a gap | Add "no price, no discount, no date" to the draft rules and reject the batch |
| The queue never empties | You are working the whole inbox instead of a slice of it | Filter by `platform`, `status` or `unreadOnly`, keep `limit` at 8 |
| Summary counts do not match what you see in the app | Another person is working the same inbox | Re-run `crm_social_inbox_summary` at the start of each pass |
| Classification drifts week to week | Category rules live in your head, not the prompt | Move the definitions into the saved prompt verbatim |

## Exercise

Run the routine for five working days and log two numbers each morning: the length of `awaitingReply` from the first call, and the share of drafts you sent unedited. On day six, change exactly one thing in the saved prompt, most usefully the draft length limit or the category definitions, and run five more days. Compare. One variable at a time is the only way to learn which line of the prompt is carrying the routine.

For a harder version: add a fifth category called `partner`, for people proposing collaborations, and give it its own action (task with a one week due date, no draft). Fifth categories are where classification rules usually break, so this tells you how brittle yours are.

## Where to go next

- [03: AI Content Calendar](./03-ai-content-calendar.md) applies the same approve-then-write pattern to publishing instead of replying.
- [06: Social Media Reporting With AI](./06-social-media-reporting-with-ai.md) turns the two metrics above into a weekly report.
- [07: MCP Security Checklist](./07-mcp-security-checklist.md) covers scope minimisation and key rotation before you put this on a shared machine.
- Full tool reference and argument lists: [docs.crmsolid.com/integrations/mcp/](https://docs.crmsolid.com/integrations/mcp/)
- Package: [@crmsolid/mcp-server on npm](https://www.npmjs.com/package/@crmsolid/mcp-server) and [source on GitHub](https://github.com/CRM-Solid/crmsolid-mcp)
- Back to [the guide index](../README.md)
