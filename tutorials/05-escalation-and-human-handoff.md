# AI Customer Support Escalation: Handing a Social Conversation to a Human

AI customer support escalation is the moment your assistant stops drafting replies and puts a named person on the hook with a deadline. It is not a fallback for when the model gets confused. It is a rule you write down in advance, apply during triage, and audit afterwards. This tutorial builds the whole path over the Model Context Protocol: decide what must escalate, detect it while triaging the social inbox, create and assign the task, write a handoff note a human can act on without reading the thread, hold the assistant off that one contact, and confirm the loop closed.

It follows on from [the AI DM triage workflow](./02-ai-dm-triage-workflow.md). If your MCP server is not connected yet, start with [connect your first social MCP server](./01-connect-your-first-social-mcp-server.md).

One convention before the examples: MCP tool output is camelCase, and the v1 REST API behind it is PascalCase. Every JSON block below is MCP tool output.

## What must escalate

Five categories are non-negotiable. Each one either costs money, starts a legal clock, or has already failed once.

| Category | What it looks like in a DM | Why a human takes it | Clock |
|---|---|---|---|
| Refunds and cancellations | "I want my money back", "cancel my subscription and refund July" | A refund moves money and usually needs a system a bot cannot see | Same business day |
| Chargebacks and payment disputes | "I have contacted my bank", "I am disputing the charge" | Card networks give you a fixed window to respond with evidence | 2 hours |
| Legal language | "lawyer", "consumer rights", "GDPR", "delete all my data", "small claims" | A wrong sentence becomes evidence. Data deletion requests are statutory | 1 hour to acknowledge |
| Repeat contacts | "I already asked twice", "still waiting since Tuesday" | The automated path has demonstrably failed for this person | 4 hours |
| Anything with a stated deadline | "before my flight on Friday", "the event is on the 3rd" | The customer has told you exactly when this becomes a failure | Their deadline minus one day |

Two more are worth adding once the first five are working: an outage report from someone describing a shared failure ("nobody on my team can log in"), and a churn signal from an account you can identify as high value.

Just as important is the list of things that must never escalate. An escalation policy that fires on everything is the same as no policy, because the queue stops being read. Delivery status, opening hours, "do you support X", "where do I change my password", and anything answered on a public page you can link to are assistant work. If your team is escalating more than roughly one conversation in ten, the problem is the policy, not the volume.

## Detect escalation during triage

Escalation is a classification step inside the triage pass, not a separate job. The tool sequence is short.

1. `crm_social_inbox_summary` to size the queue.
2. `crm_list_social_conversations` with `status: "active"` and a `limit` you can actually process.
3. `crm_list_social_messages` with `conversationId` and `limit: 20` for each candidate, most recent first.
4. Classify each conversation against the policy.
5. For anything that matches, stop drafting a customer reply and switch to the handoff path.

The summary tells you whether you are triaging 11 conversations or 400.

```json
{
  "accounts": 4,
  "conversations": 132,
  "activeConversations": 34,
  "archivedConversations": 98,
  "unreadConversations": 11,
  "unreadMessages": 19,
  "lastMessageAt": "2026-08-24T09:12:40Z",
  "platforms": [
    { "platform": "instagram", "conversations": 71, "unreadConversations": 7, "unreadMessages": 12 },
    { "platform": "linkedin", "conversations": 38, "unreadConversations": 3, "unreadMessages": 5 },
    { "platform": "x", "conversations": 23, "unreadConversations": 1, "unreadMessages": 2 }
  ]
}
```

A conversation that will escalate looks no different from any other in the list. The signal is in the messages.

```json
{
  "id": 4830,
  "platform": "instagram",
  "participantName": "Mert A.",
  "participantUsername": "merta",
  "contactId": 88210,
  "unreadCount": 3,
  "status": "active",
  "lastMessageAt": "2026-08-24T09:12:40Z",
  "lastMessageOutgoing": false,
  "lastMessagePreview": "this is the third time I am asking about order 4471"
}
```

```json
{
  "id": 88301,
  "direction": "inbound",
  "senderName": "Mert A.",
  "text": "This is the third time I am asking about order 4471. If the refund is not done by Friday I will ask my bank to reverse it.",
  "attachmentUrl": null,
  "attachmentType": null,
  "transcript": null,
  "translation": null,
  "status": "delivered",
  "sentAt": "2026-08-24T09:12:40Z"
}
```

That single message trips three categories at once: refund, repeat contact, and a stated deadline. When categories stack, the tightest clock wins.

## Trigger phrases and the action each one maps to

Give the assistant a literal list for the cheap wins and let it judge meaning for the rest. Substring matching alone will miss a polite customer; meaning alone will occasionally miss the word "chargeback" sitting in a long paragraph. Use both.

| Phrase or pattern | Category | Action | Clock |
|---|---|---|---|
| refund, money back, reimburse | Refund | Acknowledge, create task, assign to billing owner, hold assistant | Same business day |
| chargeback, dispute the charge, my bank, section 75 | Chargeback | Acknowledge, create task marked urgent, notify billing owner directly | 2 hours |
| lawyer, solicitor, legal action, small claims, consumer rights | Legal | Acknowledge receipt only. No explanation, no admission, no offer | 1 hour to acknowledge |
| GDPR, delete my data, subject access request, DSAR | Legal and privacy | Route to the named privacy owner. Never handle in the DM thread | 1 hour to acknowledge |
| third time, already asked, still waiting, nobody replied | Repeat contact | Escalate regardless of topic. Note how many prior attempts | 4 hours |
| by Friday, before my flight, event is on, deadline | Deadline | Escalate with the customer's own date recorded in the task | Their date minus one day |
| cancel my account, closing our account, moving to | Churn risk | Escalate to the account owner, not to support | 4 hours |
| nobody on my team, everyone is getting, since the update | Possible incident | Escalate to on-call before replying to anyone else | 15 minutes |
| Anything the assistant cannot answer from a documented source | Unknown | Acknowledge honestly, escalate, never guess | Same business day |

Keep a separate phrase list per language you actually receive. An English list will not catch the Turkish "iade" or "şikayet", and a customer writing in their own language is often the one who has already tried twice.

## A worked AI customer support escalation policy you can copy

Paste this into your assistant's project instructions or system prompt and edit the names. It is deliberately short. A policy nobody can hold in their head is a policy nobody applies.

```text
ESCALATION POLICY v1

You triage social DMs. You may draft and send routine replies.
You must NOT reply on your own to any conversation that matches a category below.

CATEGORIES
  R  Refund or cancellation of a paid order
  C  Chargeback or payment dispute
  L  Legal, regulatory, privacy, or data deletion language
  P  Repeat contact: the customer says or shows they asked before
  D  Any request tied to a date the customer has stated
  I  A failure affecting more than one person
  U  Anything you cannot answer from a source you can name

WHEN A CATEGORY MATCHES
  1. Send exactly one acknowledgement in the thread. Say a person is taking it
     over and by when. Do not explain, promise, apologise for fault, or offer
     compensation.
  2. Create a task titled: [ESC] <category letter> <participantName> <conversationId>
     Due: R same business day, C +2h, L +1h, P +4h, D their date minus 1 day,
     I +15m, U same business day.
     Assign to the named owner for that category. Never to a team.
  3. Add a handoff note to the contact using the HANDOFF NOTE template.
  4. Tag the contact escalated.
  5. Stop. Do not draft further replies in this conversation.

BEFORE YOU DRAFT ANY REPLY
  Check open tasks for the contact. If a task titled [ESC] is open for this
  conversation, do not reply. Report it to me instead.

OWNERS
  R, C  -> billing owner
  L     -> privacy owner
  P, D  -> support lead
  I     -> on-call engineer
  U     -> support lead

NEVER
  Never state a refund amount. Never confirm a refund has been issued.
  Never agree to a deadline on the company's behalf.
  Never quote policy you have not been given.
```

Version the policy in the same repository as your prompts. When escalations start going wrong, the first question is always which version of the policy was live that week.

## Create the task and assign it

The task is the durable record. Everything else (the note, the tag, the thread) is context around it.

Ask the assistant in plain language and let it map that onto the task tools. In a client such as Claude Desktop, Claude Code, Cursor or ChatGPT, this reads as a normal instruction:

```text
Conversation 4830 matches R, P and D. Take the tightest clock.
Create the escalation task, assign it to the billing owner, add the handoff
note to contact 88210, and tag the contact escalated. Show me the task
before you create it.
```

The task itself carries four things and nothing else: a title that identifies the conversation without opening it, a due timestamp, one named assignee, and a link to the contact. The shape below is illustrative. Confirm the exact argument names for the task family from `tools/list` in your client or from the tools reference before your first live run.

```json
{
  "title": "[ESC] R+P+D Mert A. conversation 4830",
  "dueAt": "2026-08-27T15:00:00Z",
  "assignee": "billing-owner",
  "contactId": 88210,
  "priority": "high"
}
```

Two details save you later. Put the conversation id in the title, so a person scanning a task list can jump straight to the thread. Set the due timestamp from the customer's stated deadline minus one day, not from your internal SLA, because the customer's date is the one that turns into a complaint.

Do not create the task and the reply in the same breath. Create the task first. If task creation fails, you want to have not yet promised a human is coming.

## What a good handoff note contains

The test for a handoff note is simple: hand it to a colleague who has never seen the account and ask what they would do next. If they have to ask you a question, a field is missing.

Four things are mandatory.

**Context.** Who this is, on which platform, since when, and what they bought or asked about. Two sentences.

**What was promised.** The exact words your side has already used, quoted. This is the field people skip and the one that causes contradictions. If the assistant said "someone will contact you within four hours", the human needs to know the clock started at 09:14.

**What is blocked.** The specific thing that stops this from being finished, named as an action someone can take. "Needs refund approval over the auto-approve limit" is useful. "Customer is unhappy" is not.

**The deadline.** An absolute timestamp with a timezone, and whose deadline it is (the customer's stated date, a card network window, or your internal SLA). Relative time ("by tomorrow") rots the moment the note is read a day late.

Two optional fields earn their place: the customer's own words for the ask, quoted rather than summarised, and a one-line sentiment read so the human knows whether to open with an apology.

```text
HANDOFF NOTE  conversation 4830  instagram  @merta  contact 88210

CONTEXT
Mert A. has messaged three times since 2026-08-19 about order 4471, placed
2026-08-11. Instagram DM. No prior escalation on this contact.

CUSTOMER'S WORDS
"This is the third time I am asking about order 4471. If the refund is not
done by Friday I will ask my bank to reverse it."

PROMISED SO FAR
2026-08-20 09:31Z assistant: "our team will check this and come back to you".
2026-08-24 09:31Z assistant: "a member of our team is taking this over and
will reply by Wednesday". No amount, no date for the refund itself, has been
given to the customer.

BLOCKED ON
Refund for order 4471 needs approval from the billing owner. The order is
outside the automatic window. Nothing has been refunded yet.

DEADLINE
2026-08-27T15:00Z internal, driven by the customer's stated Friday
(2026-08-28, Europe/Istanbul). Chargeback risk if missed.

SENTIMENT
Firm, not abusive. Has been patient twice. Open with an apology for the delay.
```

Store the note against the contact with `crm_add_contact_note` so it survives the conversation, and keep the same field order every time. Humans read a familiar shape four times faster than a well-written paragraph.

## Pause the assistant for one contact

A handoff fails if the assistant keeps replying underneath the human. Start with the part you get for free: the acknowledgement itself. `crm_send_social_message` marks an operator takeover on that conversation, which pauses the CRM's own AI agent for that contact so a bot cannot answer over the person now holding the thread, and it writes a message activity to the contact timeline. So sending step 1 of the policy is not only a courtesy to the customer, it is the moment the automated replies stop.

That hold covers the CRM's agent. It does not cover the assistant sitting in your MCP client, which will happily draft into the same thread on your next triage pass. For that you need a hold that is visible to the model, survives a restart, and can be lifted cleanly. Three layers, from most portable to most blunt.

**Layer one: the hold is an open task.** The `[ESC]` task is already the record. Make the triage prompt check it. Before drafting any reply, the assistant lists open tasks and skips any conversation whose id appears in an open `[ESC]` title. This is the layer that matters, because it is data, not configuration, and it works in every MCP client without any local change.

**Layer two: a tag for humans.** `crm_tag_contact` with `escalated` puts the state where a person can see it in the CRM. Treat this as signage, not enforcement. Tag tools generally add rather than remove, so do not build the resume path on removing a tag.

**Layer three: take the write tools away.** While a human is working a queue, run the server read-only. Nothing can be sent by accident, including by you.

```jsonc
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--read-only", "--tools", "social,contacts,tasks"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

`--read-only` drops every write tool in the local proxy before the client ever sees the list, so `crm_send_social_message` is not merely refused, it is not listed. `--tools social,contacts,tasks` narrows the surface further. Both filters run locally. A filtered tool is not callable at all.

There is a limit worth stating plainly: layer three is per client session, not per contact. It pauses everything, not just one conversation. Use layer one for per-contact holds and keep layer three for the hour a human is actively working the inbox.

## Resume, and confirm the loop closed

Resuming is the reverse of the hold, in a fixed order.

1. The human finishes the work in the system where it lives (the refund, the deletion, the fix).
2. The human, or the assistant on instruction, sends the closing message with `crm_send_social_message`. The tool takes no idempotency key, so read the result before doing anything else: a success returns `status: "sent"` with a `messageId`, and a platform rejection returns an error. Never fire a second attempt on a timeout without reading the thread back first.
3. Complete the escalation task with `crm_complete_task`. This is what lifts the hold, because the triage prompt reads open tasks.
4. Mark the conversation read with `crm_mark_social_conversation_read`.
5. Add a closing note to the contact: what was actually done, and the amount or reference the customer now holds.

Then audit. Once a week, ask the assistant for every open `[ESC]` task past its due timestamp, grouped by assignee and category. That report is the only reliable measure of whether escalation works, and it takes one tool call.

```text
List open tasks whose title starts with [ESC] and whose due date has passed.
Group by assignee. For each, give me the category letter, the contact name,
the conversation id, and how many hours past due it is. No commentary.
```

If the same category is always late, the deadline is wrong or the owner is overloaded. If the same conversation ids keep reappearing after being closed, your closing messages are not actually resolving anything.

## Failure modes worth designing against

**Escalating to a team.** A task assigned to "support" is assigned to nobody. Always a person, with a named backup for holidays.

**The silent handoff.** The customer is moved to a human and never told. They message again, which creates a second escalation for the same problem. Always send exactly one acknowledgement, and always with a time.

**The assistant contradicting the human.** Almost always caused by skipping the hold check. Put the check before the drafting step in the prompt, not after.

**Deadline drift.** When the assignee is out, reassign the task. Extending the due date without telling the customer converts a late reply into a broken promise.

**Duplicate sends.** Nothing deduplicates a retry of `crm_send_social_message`, so any path that resends on a timeout will eventually double-send an apology, which reads worse than silence. Check the thread with `crm_list_social_messages` before the second attempt.

**Policy rot.** Trigger phrases age. Review the list monthly against the conversations that escalated late, and add the phrases you missed.

## Related reading

- [Connect your first social MCP server](./01-connect-your-first-social-mcp-server.md)
- [The AI DM triage workflow](./02-ai-dm-triage-workflow.md)
- [Build an AI content calendar](./03-ai-content-calendar.md)
- [Cross-posting without copy and paste](./04-cross-posting-without-copy-paste.md)
- [Social media reporting with AI](./06-social-media-reporting-with-ai.md)
- [MCP security checklist](./07-mcp-security-checklist.md)
- [Repository index](../README.md)

External references: the [MCP specification](https://modelcontextprotocol.io), the [Pinlyx MCP docs](https://docs.pinlyx.com/integrations/mcp/), the [npm package](https://www.npmjs.com/package/@crmsolid/mcp-server) and its [source repository](https://github.com/CRM-Solid/pinlyx-mcp).
