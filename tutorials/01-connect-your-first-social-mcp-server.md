# Connect an MCP Server to Your AI Assistant in 10 Minutes

To connect an MCP server you do four things: install the runtime it needs, create an API key, add one block to your client's config file, and restart the client. This tutorial walks that path end to end with a social media server, so at the finish your assistant can read a DM inbox and schedule a post that you then cancel before it goes anywhere. Every stage ends in a check that fails if you skipped the stage before it, which is the difference between a tutorial and a wish.

The worked example uses `@crmsolid/mcp-server`, a stdio server covering DMs and posts on 12 platforms. The steps are the same shape for any stdio MCP server. Only the package name, the environment variable and the tool names change.

## What you need before you connect an MCP server

| Requirement | Why | Check |
|---|---|---|
| Node.js 20 or newer | The server is ESM and declares `node >= 20` | `node --version` |
| One MCP client | Claude Desktop, Claude Code and Cursor are covered below | Any recent build |
| An API key | The server authenticates to a hosted API with a bearer token | Created in step 2 |
| `curl` | Used to verify without a client | `curl --version` |

Time: about 10 minutes, most of it waiting for a client to restart.

## Step 1: Confirm your Node version

```bash
node --version
npx --version
```

Expected output is a version line starting with `v20.`, `v22.` or higher, then an npx version.

```text
v22.11.0
10.9.0
```

If `node` is missing or reports `v18` or lower, install the current LTS from nodejs.org, or use a version manager such as nvm or Volta. Do not skip this. A Node 18 runtime fails at import time with a message that has nothing to do with MCP, and you will spend twenty minutes reading the wrong logs.

**Verify:** rerun `node --version` in a new terminal window. Version managers often patch only the current shell, and your MCP client will launch a fresh one.

## Step 2: Create an API key and put it in your environment

Open [app.crmsolid.com/settings/developers](https://app.crmsolid.com/settings/developers), create a key, and grant it four scopes: `social:read`, `social:write`, `posts:read` and `posts:write`. Copy the value, which starts with `csk_live_`. It is shown once.

Scopes are per key, so give this one only what this tutorial needs. If you are only reading, grant the two read scopes and stop there.

```bash
export CRMSOLID_API_KEY="csk_live_replace_me"
```

```powershell
$env:CRMSOLID_API_KEY = "csk_live_replace_me"
```

**Verify:** ask the API who you are before involving any client.

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  https://api.crmsolid.com/v1/social/accounts \
  -H "Authorization: Bearer $CRMSOLID_API_KEY"
```

Expected output is `200`. A `401` means the key is wrong or the variable is empty. A `403` means the key exists but lacks `social:read`.

## Step 3: Run the server once, outside any client

```bash
npx -y @crmsolid/mcp-server --version
npx -y @crmsolid/mcp-server --help
```

The first call downloads the package. The second prints the flags the server accepts, including `--api-key`, `--base-url`, `--tools` and `--read-only`.

This step exists because a client that cannot start a server tells you almost nothing about why. Debugging in a terminal takes seconds. Debugging inside a desktop app's log file does not.

**Verify:** `--help` exits with status 0 and prints flag names. If it prints a stack trace, fix that before touching any config file.

## Step 4: Add the server to your client config

Pick your client. The JSON shape is the same everywhere.

### Claude Desktop

Edit the config file directly:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```jsonc
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server"],
      "env": { "CRMSOLID_API_KEY": "csk_live_replace_me" }
    }
  }
}
```

If the file already has an `mcpServers` object, add `crmsolid` inside it rather than pasting a second top level key.

### Claude Code

Let the CLI write the config:

```bash
claude mcp add crmsolid \
  --env CRMSOLID_API_KEY=csk_live_replace_me \
  -- npx -y @crmsolid/mcp-server
```

Everything after `--` is the command Claude Code will spawn. The default scope stores this in your user config. Add `--scope project` to write a `.mcp.json` your team shares, and keep the key out of it by using an environment variable reference instead of the literal value.

### Cursor

Create `~/.cursor/mcp.json` for every project, or `.cursor/mcp.json` inside one project:

```json
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server"],
      "env": { "CRMSOLID_API_KEY": "csk_live_replace_me" }
    }
  }
}
```

**Verify:** confirm the file is valid JSON before you restart anything. A trailing comma is the single most common failure at this step.

```bash
node -e "JSON.parse(require('fs').readFileSync(process.argv[1],'utf8'));console.log('config parses')" \
  ~/.cursor/mcp.json
```

## Step 5: Restart the client and confirm the server is listed

Closing the window is not restarting. Quit the application completely, including the tray icon on Windows and the dock icon on macOS, then start it again.

- **Claude Desktop:** open the tools or connectors menu in the composer. `crmsolid` appears with a tool count.
- **Claude Code:** run `claude mcp list` in a terminal, or `/mcp` inside a session. Expect `crmsolid: connected`.
- **Cursor:** open Settings, then MCP. The server shows as enabled with its tools listed.

With no filters applied, this server publishes 62 tools, 21 resources and 15 prompts. Seeing a smaller number is not a bug on its own: some clients cap how many tools they forward to the model. It does mean you should narrow the surface, which step 8 covers.

**Verify:** the server name appears and the tool count is greater than zero. If it is listed but shows an error badge, open the client's MCP log and read the first error, not the last one.

## Step 6: Make a read only call

Ask the assistant, in plain language:

```text
List my connected social accounts, then give me the inbox summary.
```

The model calls `crm_list_social_accounts` and `crm_social_inbox_summary`. Both are annotated read only, so most clients run them without asking for confirmation.

MCP tool output is camelCase. The v1 REST API returns PascalCase for the same data, which matters only if you mix the two in one script.

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
  ]
}
```

Then read one conversation:

```text
Show me the 5 most recent active Instagram conversations.
```

That is `crm_list_social_conversations` with `platform: "instagram"`, `status: "active"`, `limit: 5`. A conversation is either `active` or `archived`, never "open". MCP list tools take `limit` (1 to 100, default 25) and answer with a named array plus a `count`. There is no cursor to pass back: to see less, lower `limit`; to see something else, filter. Cursor pagination (`items`, `nextCursor`, `hasMore`, `after`) belongs to the v1 REST API, not to these tools.

**Verify:** you got real account names and real counts. If the counts are all zero, no social account is connected to the workspace yet, and the plumbing is still fine.

## Step 7: Make one write, with approval in front of it

A good first write is one you can undo. Schedule a post far enough ahead that you can cancel it long before it goes anywhere:

```text
Schedule a post for LinkedIn and X with the text
"Three things we learned migrating 40 support inboxes."
Use scheduledAt 2026-12-31T09:00:00Z. Do not publish it now.
```

That calls `crm_schedule_social_post` with `content`, `platforms` and `scheduledAt`. The rule to remember: `scheduledAt` is required unless you pass `publishNow: true`, and omitting both is rejected with "scheduledAt is required unless publishNow is true". There is no draft state, so an assistant that forgets to say when gets an error rather than a surprise publish. Your client should ask you to approve this call, because the tool is annotated as an open world write.

```json
{
  "count": 2,
  "postIds": [993, 994],
  "platforms": ["linkedin", "x"],
  "scheduledAt": "2026-12-31T09:00:00Z",
  "status": "pending",
  "skipped": null,
  "message": "Scheduled on 2 account(s) for 2026-12-31 09:00 UTC."
}
```

One row is created per target account, which is why `postIds` comes back as an array. Ids are integers here, not opaque strings.

Confirm they exist, then remove them:

```text
List my pending posts, then cancel the two you just created.
```

`crm_list_social_posts` with `status: "pending"` finds them. `crm_cancel_social_post` with each `postId` cancels one and returns a confirmation of what changed, not a data feed. No tool in this server both reads and writes, which is what makes approval decisions readable.

**Verify:** list pending posts again and confirm both are gone.

## Prove it works without a client

When a client misbehaves, take it out of the loop. The hosted endpoint speaks JSON-RPC directly.

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Count what came back:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).result.tools.length))"
```

If that prints a tool count and your client still shows nothing, the problem is the client config, not the key and not the server. That single fact removes most of the search space.

## Narrow the surface before you go further

Two local flags shrink what the model can even see. Both are applied by the local proxy, so a filtered tool is not listed and not callable.

```jsonc
{
  "mcpServers": {
    "crmsolid": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "social,posts", "--read-only"],
      "env": { "CRMSOLID_API_KEY": "csk_live_replace_me" }
    }
  }
}
```

`--tools social,posts` keeps the two families this guide uses. `--read-only` drops every write tool. Start read only for a day. It is the cheapest way to learn what the model reaches for before it can act.

## Troubleshooting: six failures and their fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Server missing from the client after restart | Config written to the wrong file, or the app was closed rather than quit | Confirm the exact path from step 4, then fully quit and relaunch |
| `spawn npx ENOENT` in the client log | Desktop apps do not inherit your shell `PATH` | Run `which npx` (macOS, Linux) or `where.exe npx` (Windows) and put the absolute path in `command` |
| Client reports an invalid config, or the entry is ignored | Trailing comma, or curly quotes pasted from a word processor | Run the `JSON.parse` check from step 4 and edit in a code editor |
| `401 Unauthorized` | Key typo, revoked key, or a misspelled variable name | The name is `CRMSOLID_API_KEY` exactly. Recreate the key and paste it once |
| `403` with a scope message | The key lacks the scope the tool needs | Grant `social:read`, `social:write`, `posts:read`, `posts:write` on the key, then restart the client |
| Tools appear but every write tool is missing | `--read-only` is set, or `--tools` excludes that family | Remove the flag from `args` and restart the client |

## Exercise

Create a second API key with only `social:read`. Add a second entry to your config named `crmsolid-readonly` that uses that key and passes `--read-only`. Restart the client, then ask the assistant to send a DM. It should not find a send tool at all, rather than trying and failing with a permission error.

Write down the tool count for each entry. The gap between them is exactly the authority you hand over when you enable writes, and it is a much better number to reason about than a vague sense of trust.

## Next: triage a real inbox

[Tutorial 02: AI DM triage workflow](./02-ai-dm-triage-workflow.md) turns these read calls into a repeatable pass over a real inbox, including the messaging window rules that decide which replies can still be delivered. If you are wiring this into a shared workspace, read [tutorial 07: MCP security checklist](./07-mcp-security-checklist.md) first. For scheduling that survives daylight saving, see [tutorial 03: AI content calendar](./03-ai-content-calendar.md).

Tool reference and full parameter lists: [docs.crmsolid.com/integrations/mcp/](https://docs.crmsolid.com/integrations/mcp/). Package: [@crmsolid/mcp-server on npm](https://www.npmjs.com/package/@crmsolid/mcp-server). Source: [github.com/CRM-Solid/crmsolid-mcp](https://github.com/CRM-Solid/crmsolid-mcp).
