# mcp-reference-server

[![npm version](https://img.shields.io/npm/v/mcp-reference-server.svg?style=for-the-badge)](https://www.npmjs.com/package/mcp-reference-server)
[![npm downloads](https://img.shields.io/npm/dm/mcp-reference-server.svg?style=for-the-badge)](https://www.npmjs.com/package/mcp-reference-server)
[![Node.js](https://img.shields.io/badge/Node.js-18+-339933?style=for-the-badge&logo=node.js&logoColor=white)](https://nodejs.org/)
[![MCP](https://img.shields.io/badge/MCP-Protocol-5A29E4?style=for-the-badge)](https://modelcontextprotocol.io/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue?style=for-the-badge)](https://github.com/markusvankempen/mcp-reference-server/blob/main/LICENSE)
[![GitHub](https://img.shields.io/badge/GitHub-mcp--reference--server-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/markusvankempen/mcp-reference-server)

A **fully-featured reference MCP server** with a realistic support-desk sample data set, over the
Model Context Protocol. This README describes server **1.0.0**, package **mcp-reference-server**.

This build adds four extensions. **Groups** (`io.modelcontextprotocol/groups`) is a prototype of the Primitive Grouping draft. **Skills** (`io.modelcontextprotocol/skills`, SEP-2640) are workflows and files, not tools. **Tasks** (`io.modelcontextprotocol/tasks`, SEP-2663) let `create_ticket` and `run_test_job` return a handle the client polls. **Events** (`io.modelcontextprotocol/events`, SEP-3415, draft) let a client poll or briefly stream the same bus as `GET /sse`. The desk reference is [Docs → Extensions](http://127.0.0.1:8787/help#help-specs) while the server is running. The sections below are the same reference.

- **22 tools.** Six run the ticket desk. Eight operate the server itself (`update_settings` changes any setting with one patch). Three are timers (`start_timer`, `stop_timer`, `push_timer`), one is an event (`push_log`), `watch_system` publishes the machine, `run_test_job` is a test job for long-running calls and tasks, and two talk back to a connected client: `draft_reply` (sampling) and `list_workspace_roots` (roots). Reads that are just a list go through `run_query` instead of a tool each: tickets, customers, settings, users, API keys, the machine, the events already sent, and the call trace, errors and counters. There is no `search_tickets`, `get_ticket`, `lookup_customer`, `get_settings`, `list_schemas`, `export_settings`, `export_log` or per-setting `set_*` tool (only `set_mqtt` stays).
- **Events.** Off by default. Ticket, timer, log and settings events stream to a client over SSE or Streamable HTTP as MCP log notifications.
- **Three transports.** stdio, Streamable HTTP at `/mcp`, and legacy SSE at `/sse`. Each can be switched off at runtime.
- **Schema discovery.** `get_schema` (no name lists every name; a name gives its shape) → `run_query`, instead of a pile of `query_*` tools.
- **Permissions.** Three auth modes, four scopes, API keys, users, per-tool gates and locks, rate limiting, PII redaction, an optional tenant header.
- **Results that explain themselves.** Every result and every refusal carries a `next` field that names one follow-up tool.
- **A desk in the browser.** `/admin`, `/log`, `/tools`, `/test`, `/health`, `/help`, plus copy-paste client pages at `/clients/`. Project, light and dark themes.

Runs as a local stdio child process, a local HTTP server, a container, or a public HTTPS host.
The desk at `/admin` is grouped as Access, Records, and Operate.

```
# From server/ after npm install
node src/index.js

npx mcp-reference-server
```

---

## Transports

```
MCP_MODE=stdio  node src/index.js     child process, no port, no URL (the default)
MCP_MODE=http   node src/index.js     Express: /mcp  /sse  /messages  + the desk pages
```

| Transport | Where | Notes |
|---|---|---|
| stdio | `MCP_MODE=stdio` | Credentials come from the process env: `MCP_API_KEY`, or `MCP_USERNAME` + `MCP_PASSWORD`. |
| Streamable HTTP | `POST /mcp` | Current remote transport. `initialize` opens a session so `tools/list_changed` reaches the client. A plain one-shot POST (curl) is stateless. |
| SSE | `GET /sse`, `POST /messages` | Legacy. Kept for older clients. |

An administrator can switch any transport off with `update_settings` (`protocols`). **At least one stays on.** A disabled HTTP transport answers `404` with the tool to call to turn it back on. Stdio exits with a message if it is off.

`/health`, `/test`, `/admin` and the other desk pages are a convention of this server, not part of the MCP spec.

---

## Tools

Six tools are the ticket desk. Eight more operate the server. Three are timers, one sends log lines, and `watch_system` publishes the machine. A client that has not negotiated groups gets every tool from `describe_server`, including what **you** may call right now. A client that has negotiated groups gets only `describe_server` and `get_schema` from that call, and the next step is `groups/get`.

Every tool has a short purpose that says **when to call it and when not to**, and an output schema. Read tools carry `readOnlyHint`. `close_ticket` and the tools that delete or replace things carry `destructiveHint`. Ticket ids and schema names auto-complete in prompts and resources.

### Groups prototype

Experimental extension `io.modelcontextprotocol/groups`, following the Primitive Grouping draft in progressive-disclosure-wg PR #19. This server follows that draft. It is not the draft, and it is not a final SEP. It is on by default in this build. Turn it off with `update_settings` `groups=false`, or the Groups switch on **Admin → Access → Security**.

A client that does not negotiate the extension still receives all 22 tools from `tools/list`, and `describe_server` still lists every tool. A client negotiates by sending `capabilities.extensions["io.modelcontextprotocol/groups"]` on `initialize`, or the same object under `params._meta["io.modelcontextprotocol/clientCapabilities"].extensions` on a one-shot request.

After negotiation, while groups are on:

1. `tools/list` contains only `describe_server` and `get_schema`.
2. `describe_server` returns who you are, your scopes, the auth mode and the rate limit. Its `tools` array is that same pair. Each group row has a description and `via: "groups/get"`, and does not name the tools inside the group. `next` says to call `groups/get` with the group whose description matches the task.
3. `groups/list` is metadata only (name, title, description, a scope hint). It needs no credential. The description is what the model uses to pick a group, so it names the tasks and does not name the tools.
4. `groups/get` returns that group's instructions and the full tool definitions, including `inputSchema`. An unknown name is JSON-RPC `-32002`.

`tools/call` is unchanged. A tool left off `tools/list`, or a group left out of `groups/list`, is still authorized by its scope. Discovery is not access control. With groups off, `groups/list` and `groups/get` are refused and a negotiated `tools/list` returns the full catalog again. An empty group list would be the wrong off state: the client already negotiated, and it would be stuck with the two catalog tools.

The server instructions say the same thing. If `describe_server` says to call `groups/get`, that is the next step. Instructions and catalog text that the model reads before `groups/get` do not name a scoped tool. `get_schema` lists names and shapes. It does not read or change a record.

`groups/list`, `groups/get` and a negotiated `tools/list` are written to the audit trail and the counters. The call trace records them while audit is on.

| Group | What the description is for | Tools, returned only by `groups/get` | Scope the calls still check |
|---|---|---|---|
| `query` | Read tickets, one ticket, a customer, settings, the user list, counters, errors, the call trace | `run_query` | read |
| `tickets-write` | Open, comment on, or close a ticket | `create_ticket`, `add_comment`, `close_ticket` | write |
| `admin` | Change settings, turn the call trace on, disable a tool, users, API keys | `update_settings`, `set_mqtt`, `create_user`, `delete_user`, `issue_api_key`, `revoke_api_key`, `import_settings` | admin |
| `events` | Timers and pushes | `start_timer`, `stop_timer`, `push_timer`, `push_log`, `watch_system` | mixed |
| `traffic` | Sample counter rows. Does not turn the call trace on | `generate_traffic` | any credential |
| `jobs` | Run a test job for a long call or a task. Changes no ticket and no setting | `run_test_job` | read |
| `client` | Ask the connected client's model for a draft, or ask which workspace roots it is sharing. A one-shot call has no client to ask | `draft_reply`, `list_workspace_roots` | read |

`groups/list` and `groups/get` are cached for 60 seconds, private to the caller. This build also accepts `params._meta["io.modelcontextprotocol/groups"].omit`, a list of group names to leave out of that one call. The tools stay callable. That knob is not in the draft.

The desk shows the same path. [Docs → Extensions](http://127.0.0.1:8787/help#help-specs) is the reference. [Docs → Tools](http://127.0.0.1:8787/help#help-tools) has the curl sequence that lists open tickets. [/tools → Groups](http://127.0.0.1:8787/tools#tools-groups) runs that sequence and does not create a ticket. The command line has `groups`, `groups get <name>` and `groups tickets`. **Admin → Operate → Samples** has **Generate via groups**, which creates tickets by calling `tickets-write` rather than writing the store directly. `/health` shows the group count when groups are on, and this machine's CPU, memory and disk (the same snapshot as `run_query` schema `system`).

### Skills

Official extension `io.modelcontextprotocol/skills` (SEP-2640, final). The server declares `directoryRead: true`. Three built-in workflows: `work-an-open-ticket` and `open-a-ticket` on the `tickets-write` group, and `read-the-queue` on the `query` group. A skill names the workflow. It does not include a tool schema, and reading a file does not grant a scope. `groups/get` still returns the tools, and it links the skill file as a resource on the group the skill belongs to.

| Method | What it returns | Credential |
|---|---|---|
| `skills/list` | Every skill: frontmatter (`name`, `description`) and a file manifest. Each file is a URI, a `sha256:` digest, and a byte size. `ttlMs` is 300000 (five minutes), cached as public. | none |
| `skills/get` | One entry, the same shape, looked up by URI. | none |
| `resources/read` | The file bytes. The digest in the list is the SHA-256 of those bytes. | none |
| `resources/directory/read` | One directory. | none |

An unknown skill, file, or list cursor is JSON-RPC `-32602`. There is no `notifications/skills/list_changed`. A host can show the previous list until `ttlMs` passes. Replacing a skill changes its digests, so a host that approved the old bytes asks again. These calls are written to the audit trail. [Docs → Extensions](http://127.0.0.1:8787/help#help-specs) is the reference. [/tools → Skills](http://127.0.0.1:8787/tools#tools-skills) runs the open-ticket sequence, and [Docs → Tools](http://127.0.0.1:8787/help#help-tools) has the curls.

#### Uploading a skill

An admin can add skills on **Admin → Records → Skills**, or with `POST /admin/skills`. Each uploaded skill is one folder in `~/.mcp-reference-server/skills` (set `MCP_SKILLS_DIR` to change it). The server reads that folder at startup and again after each upload or delete, so an upload is served at once and survives a restart. Tickets are still in memory. An upload adds instructions and supporting files. It does not add a tool or grant a scope.

The server checks each upload before saving it:

- The frontmatter is flat `name` and `description` lines, and `name` equals the folder name. The server must report what a YAML parser would read, so a value YAML would read differently has to be in double quotes.
- The description names no scoped tool. `describe_server` copies it, and clients that have not called `groups/get` must not see scoped tool names there. Name tools in the body.
- `allowed-tools` is refused, because the spec says hosts ignore it for skills served over MCP.
- Only `.md` and `.txt` files, at most 20 files, 64 KB a file, 256 KB a skill, and three folders deep. A nested `SKILL.md` is refused.
- Built-in skills cannot be replaced or deleted.
- An optional `group` decides which `groups/get` result links the skill.

Replacing a skill changes its digests, so a host that approved the old version asks again. There is no `list_changed` notification for skills, and `skills/list` has a five-minute `ttlMs`, so a host can show the old list for up to five minutes.

`examples/skills/close-stale-pending` is an example to upload. It closes pending tickets that are waiting on a customer who never replied, and it uses only `run_query`, `add_comment` and `close_ticket`. No tool deletes or archives a ticket. Closing one is the strongest action the tools allow.

```bash
# From this folder. Sign in; the session cookie is the credential.
curl -s -c /tmp/mcp_cookies.txt -o /dev/null -X POST 'http://127.0.0.1:8787/admin/login' \
  -d 'username=demo&password=demo'

cd examples/skills/close-stale-pending
jq -n --rawfile skill SKILL.md --rawfile when references/when-to-close.md \
  '{group: "tickets-write", files: {"SKILL.md": $skill, "references/when-to-close.md": $when}}' \
| curl -s -b /tmp/mcp_cookies.txt -X POST 'http://127.0.0.1:8787/admin/skills' \
    -H 'Content-Type: application/json' -H 'Accept: application/json' -d @- | jq .

# Delete it again
curl -s -b /tmp/mcp_cookies.txt -X POST 'http://127.0.0.1:8787/admin/skills/close-stale-pending/delete' \
  -H 'Accept: application/json' | jq .

# Download one skill. The file is the upload body, so posting it back replaces the skill.
curl -s -b /tmp/mcp_cookies.txt -o close-stale-pending.json \
  'http://127.0.0.1:8787/admin/skills/close-stale-pending/download'

# Every skill that is served, and a blank template (name my-skill) that already passes the checks.
curl -s -b /tmp/mcp_cookies.txt -o skills.json 'http://127.0.0.1:8787/admin/skills/download'
curl -s -b /tmp/mcp_cookies.txt -o skill-template.json 'http://127.0.0.1:8787/admin/skills/template'
```

A download is JSON with `name`, `group`, and `files`, the same fields `POST /admin/skills` reads. `GET /admin/skills/download` is every served skill, built-in and uploaded. `GET /admin/skills/<name>/download` is one of them. `GET /admin/skills/template` is a blank skill named `my-skill`. Change that name before you upload it as your own. A built-in skill can be downloaded. Uploading it back under the same name answers 409, so rename the copy. Each download is written to the audit trail as `skill.download`.

A new upload answers 201, and a replacement answers 200. A rejected upload answers 400 with `error` and `next`. A built-in name answers 409, and a request without a session answers 401. Uploads and deletes are written to the audit trail as `skill.upload` and `skill.delete`. `npm run test:skill-upload` runs the whole path against a temporary folder.

### Tasks

Official extension `io.modelcontextprotocol/tasks` (SEP-2663, final). The client declares it on the request, under `params._meta["io.modelcontextprotocol/clientCapabilities"].extensions`. The server decides per call. Here the task tools are `create_ticket` and `run_test_job`. Every other tool still returns its result on the same response. A nested `params.task` from an older client is ignored.

`tools/call` returns a flat handle. There is no nested task object.

| Field | Meaning |
|---|---|
| `resultType` | `"task"` when the call creates a task. `tasks/get`, `tasks/update`, and `tasks/cancel` answer `"complete"`. |
| `taskId` | The handle. There is no `tasks/list`. |
| `status` | `working`, `input_required`, `completed`, `failed`, or `cancelled`. There is no `pending`. |
| `createdAt`, `lastUpdatedAt` | When the task was stored, and when it last changed. |
| `ttlMs` | How long the handle lives. Default 60 seconds (`MCP_TASK_TTL_MS`). A test job adds its `seconds` to that. |
| `pollIntervalMs` | `500`. Poll about that often. |

When the status is `completed`, `tasks/get` carries `result`, the full tool result. A tool result with `isError: true` is still `completed`. Only a JSON-RPC error is `failed`. `tasks/result` is JSON-RPC `-32601`. Read the tool result from `tasks/get`.

`tasks/get`, `tasks/update`, and `tasks/cancel` from a client that did not declare the extension are `-32021`, with `data.requiredCapabilities`. The same `tools/call` without the extension returns the ticket or the job on that response. An unknown task id is `-32602` (`Task not found`). After `ttlMs`, the same code says `Task has expired` and `data.status` is `expired`. SEP-2663 does not add `expired` to the status enum, so the task object is not returned. The record stays for `MCP_TASK_EXPIRE_RETAIN_MS` (default 60 seconds) and is then dropped. A later get is `Task not found`. A different caller gets that same not-found error: tasks are owner-bound. On Streamable HTTP send `Mcp-Name: <taskId>` and `Mcp-Method` with those three methods.

`create_ticket` without `requester_email` stays `input_required` until `tasks/update` sends the email. Declining that elicitation cancels the task, and no ticket is opened. With the email, the task works for about 400 ms and then completes with the ticket. A call that does not declare the extension still opens the ticket on the same response. Leaving the email out leaves the service account as the owner. Cancel is cooperative: a running `run_test_job` stops at once; `create_ticket` does not abort mid-write.

Records live on the process, not on one session, so a later one-shot `POST /mcp` can still poll. The store keeps at most 1000 open tasks. A restart drops them. [/tools → Tasks](http://127.0.0.1:8787/tools#tools-tasks) opens one ticket this way. [Docs → Extensions](http://127.0.0.1:8787/help#help-specs) is the reference.

#### A resumable event stream

`tasks/stream` is SEP-2694, a draft on top of SEP-2663. Not a final SEP. The same declaration as the other task methods. The request stays open. The server sends `notifications/tasks/event` on that response, then an empty result when it closes the stream. A session shows the notifications. A one-shot JSON body does not.

Each event has `seq` (the first is 0), `taskId`, `type`, `payload`, and `_meta["io.modelcontextprotocol/related-task"].taskId`. `after` is exclusive: the next call sends events with a greater `seq`. A second `tasks/stream` for the same task on the same connection is `-32602`. Cancel the stream with the normal request cancellation. That does not cancel the task. `tasks/cancel` does.

This server keeps the last 64 events (`MCP_TASK_EVENT_LIMIT`) for as long as `tasks/get` can still return the task. A cursor older than that log is `-32010`, `data.reason` `events_gone`, with `earliestSeq`. Omit `after` to start at the oldest event still kept. Event types are `io.github.markusvankempen.ticket.status` on each status change, and `io.github.markusvankempen.ticket.job.progress` (`done`, `total`) once a second while `run_test_job` is the task. Closing the stream, or the task expiring, does not invent a status named `expired`.

#### A test job for every task status

`run_test_job` (read scope, group `jobs`) works for `seconds` (0 to 600, default 5), then ends the way `outcome` says. It reads and changes nothing. It is for testing clients against long-running calls.

| `outcome` | Without a task | As a task |
|---|---|---|
| `completed` (default) | the job, after the wait | `completed`, the job in `result` |
| `tool_error` | a result with `isError: true` | `completed`, with `isError: true` in `result` |
| `failed` | a JSON-RPC error, `-32603` | `failed`, with that error in `error` |
| `input_required` | refused (`isError`) | `input_required` until `tasks/update` sends `inputResponses.proceed`; then it runs |
| `working` | refused (`isError`) | `working` until `tasks/cancel`, or until `ttlMs` passes |

SEP-2663 puts a tool result with `isError: true` in `completed`. Only a JSON-RPC error makes a task `failed`. Without a task, the request stays open for the whole wait, so `seconds` is at most 30. On that open request, put `progressToken` in `params._meta` and the server sends `notifications/progress` once a second (`progress`, `total`, and a message). A session stream shows them. A one-shot JSON body does not, and the job still returns. A task does not use that notification: `tasks/get` reports the same wait in `statusMessage` (`Working: 3 of 10 s.`). `tasks/cancel` stops a running job at once. The task's `ttlMs` is the configured TTL plus the job's seconds. `proceed: false`, `decline`, or `cancel` cancels the job. There is no `pending` status in the extension. A job that is waiting on the client is `input_required`, and a job that never ends is `working`. [/tools → Test job](http://127.0.0.1:8787/tools#tools-job) has that panel, with Cancel, Send input, and Decline.

```bash
META='{"io.modelcontextprotocol/clientCapabilities":{"extensions":{"io.modelcontextprotocol/tasks":{}}}}'
TASK=$(curl -s -X POST 'http://127.0.0.1:8787/mcp' \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"run_test_job","arguments":{"seconds":20,"outcome":"failed"},"_meta":'"$META"'}}' \
  | jq -r '.result.taskId')

curl -s -X POST 'http://127.0.0.1:8787/mcp' \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H 'Mcp-Method: tasks/get' -H "Mcp-Name: $TASK" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tasks/get","params":{"taskId":"'"$TASK"'","_meta":'"$META"'}}' \
  | jq '.result | {status, statusMessage, error}'
```

### Events extension

Draft extension `io.modelcontextprotocol/events` (SEP-3415, [PR #3415](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3415)). Not a final SEP. The client declares it on the request, the same `_meta` object as tasks. `events/list`, `events/poll`, and `events/stream` read the bus documented under [Events](#events). They do not replace `GET /sse` or `GET /mcp`.

`cursor: null` starts from now and returns no history. Pass the returned cursor on the next poll. `truncated: true` means the ring buffer dropped events that cursor still named. `hasMore: true` means poll again now. An unknown name is `-32023`. An admin kind without the admin scope is `-32024`. A request that does not declare the extension is `-32021`.

`events/stream` sends `notifications/events/active` and `notifications/events/event` on that response, then closes. It replays, waits up to two seconds for one new event, and returns. A tail that stays open is still `GET /sse` or `GET /mcp`. A session stream shows the notifications. A one-shot JSON body does not.

`events/subscribe` and `events/unsubscribe` are `-32026`. The draft's webhook mode needs an authenticated principal and an https callback. This server can run with auth off, and it does not POST to a URL the caller chooses.

The bus still has to be announcing before a new event is stored. Ticket kinds take `arguments.ticket_id`. Every kind takes `arguments.topic`.

### Asking the connected client

These are core protocol features, not extensions. Both tools are read scope, in the `client` group. A one-shot `POST /mcp` has no client to ask, so each returns an error and `next`. [Docs → Extensions](http://127.0.0.1:8787/help#help-specs) says the same thing.

| Tool | What it calls | What it does not do |
|---|---|---|
| `draft_reply` | `sampling/createMessage` | Does not post the draft. Call `add_comment` yourself if you want it on a ticket. The client declares `capabilities.sampling` on `initialize`. |
| `list_workspace_roots` | `roots/list` | Does not read the files. Returns the URIs the client chose to share. The client declares `capabilities.roots`. `notifications/roots/list_changed` makes the server ask again. |

### Ticket desk

| Tool | Scope | What it does |
|---|---|---|
| `describe_server` | read, open in every mode | Call it first, and after any denial. Identity, auth mode, your scopes, rate-limit budget, resources, prompts, protocols. Without groups negotiation it also lists every tool. With negotiation it lists only itself and `get_schema`, and `next` is `groups/get`. |
| `create_ticket` | write | Open a ticket. **Always pass `requester_email`**, or the service account owns it and every reply goes to the bot. |
| `add_comment` | write | Comment on a known `ticket_id`. |
| `close_ticket` | write | Resolve a ticket. Optional `resolution` becomes the last comment. Idempotent. |
| `get_schema` | read | Discovery, in one tool. With no `name` it lists every name with a `kind` (`data`, `query` or `tool`). With a `name` it gives that shape or that tool's arguments. It does not read or change a record. |
| `run_query` | read | Exact-match query on `tickets`, `customers`, `assets`, `timers`, `system`, `settings`, `events`, `users`, `api_keys`, `trace`, `errors` or `counters` only, with `filter`, `fields`, `limit`. **Tickets:** `filter` takes `id` (one ticket, with every field including body and comments), `status` (`open`, `pending`, `solved`; `all` or omitted lists every status), `requester_email`, `attribution` and `query` (keyword in subject or body, any case). A list returns short fields (`id`, `subject`, `status`, `requester_email`, `attribution`, `created_at`); `fields` can add `body`, `comments` and `resolved_at`. `limit` is up to 50, default 10, newest first, and there is no default status filter. **Customers:** `filter` takes `email` or `region`; `phone` is the text `REDACTED` unless the caller holds `pii`. `users`, `api_keys`, `trace`, `errors` and `counters` need the `admin` scope even while auth is `off`; the last three give at most 50 rows per call (`counters` is one row per tool, so use `limit` 50). `trace` records only while audit is on (`update_settings` with `audit=true`); `errors` always. A filter key that is not in the schema's `filterable` list is refused, and the error names the keys you can use. |

### Operating the server

| Tool | Scope | What it does |
|---|---|---|
| `update_settings` | admin | **Every setting is one patch**, shaped like the settings row and all fields optional: `authMode`, `rateLimit` {`enabled`, `limit`, `windowSeconds`}, `audit`, `uiAuth`, `groups`, `protocols` {`stdio`, `streamableHttp`, `sse`}, `events` {`enabled`, `types`, `autoTimer`, `autoSystem`}, `toolGates` {tool: bool, `false` switches it off and answers `503`}, `toolAuthOverrides` {tool: bool, `true` requires a credential whatever the auth mode} and `toolScopeOverrides` {tool: `read`, `write`, `pii`, `admin` or `default`}. Several parts can go in one call. It is checked first and is **all or nothing**: a refusal is `{ok:false, error, next}` and nothing changes. A success is `{ok, settings, changed[], next}`. Admin scope even while auth is `off`. MQTT is not part of it. |
| `set_mqtt` | admin | Publish the events to an MQTT broker (publish only): URL, topic prefix, credentials, QoS, retain, test message. See [MQTT](#mqtt). |
| `create_user`, `delete_user` | admin | Manage users. Passwords are stored as hashes and never returned. The built-in admin cannot be replaced or removed. List them with `run_query` and `schema=users`. |
| `issue_api_key`, `revoke_api_key` | admin | Manage API keys. The secret is returned once, at issue. Secrets are never listed. List them with `run_query` and `schema=api_keys` (filter by `id`, `label` or `active`). |
| `import_settings` | admin | Restore a settings document. Export is a read: `run_query` with `schema=settings` returns the row, which `import_settings` takes as it is (`kind` is optional). It omits API keys and passwords. |
| `generate_traffic` | read, **any credential** | Runs scripted calls so the log has something in it. Changes no tickets or settings. |

The call trace, the errors and the counters are reads too, not tools: `run_query` with `schema=trace`, `errors` or `counters`. All three need admin and give at most 50 rows per call.

![/tools — tool inventory showing the tools, scopes, and live auth status](../docs/assets/screenshots/tools.png)

### Timers

Tools start, stop and push a timer. There is **no `get_timer`**: you read a timer through the schema, with `run_query` and `schema=timers`. That is the point of the demo. Actions are tools; reads come from the schema.

| Tool | Scope | What it does |
|---|---|---|
| `start_timer` | read | `seconds` (1 to 3600) starts a countdown. Leave `seconds` out for a stopwatch that counts up. Optional `label`. Returns the timer, with an id like `TMR-1`. |
| `push_timer` | read | `timer_id`, `every_seconds` (1 to 60), `for_seconds` (1 to 3600), `topic`. Makes the timer report its time as events. See Events below. |
| `stop_timer` | read | `timer_id`. The timer keeps its elapsed time and its state becomes `stopped`. Stopping a timer that is already stopped or finished changes nothing and returns `alreadyOver: true`. |

```bash
mcp-reference-server call start_timer seconds=30 label=demo
mcp-reference-server call run_query schema=timers 'filter={"id":"TMR-1"}'
mcp-reference-server call stop_timer timer_id=TMR-1
```

A plain timer has no clock running. A timer is a start time, an optional length and an optional stop time. `state` (`running`, `finished` or `stopped`), `elapsed_ms` and `remaining_ms` are worked out when you query, so a 30 second countdown shows `finished` once 30 seconds have passed. Only a timer you push (`push_timer`, see Events) gets a real interval, and while events are on a countdown also announces `timer.finished`. The `next` hint tells the model how long is left and not to poll in a loop.

- The server keeps at most 50 timers, **in memory**. A restart, a factory reset on `/admin`, or a host that scales to zero (Code Engine) clears them.
- The timer tools are `read` scope, so they follow the same rules as the other read tools: open while auth is `off` or `write`, a credential needed under `all`. Gates, locks, the rate limit and the call trace apply to them like any tool.
- **Declared once.** The timer tools are defined in `src/timer-tools.js`: name, scope, purpose, input and output schemas, annotations, an example call and the function that does the work. The catalog row, the MCP registration, what `get_schema name=start_timer` shows, and the example on the Tools page are all generated from that one definition.

### Events

The server can announce what happens on it and push those events to a client that is listening. **Events are off by default.** Timer events switch themselves on when you call `push_timer` (no admin credential needed). Everything else is turned on with `update_settings` and `events.enabled=true` (admin), the `ANNOUNCE_EVENTS` environment variable at startup, or in the **Events** tab on `/admin` (it uses the connection set on Remote config) and in the Events panel on on `/clients/browser.html`.

| Tool | Scope | What it does |
|---|---|---|
| `push_timer` | read | `timer_id`, optional `every_seconds` (1 to 60, default 5; 0 stops the push), optional `for_seconds` (1 to 3600: report for that long, then end; a stopwatch is stopped then) and `topic`. A running timer reports its time as a `timer.tick` event, the first at once. Up to 10 timers push at a time. If timer events are off it switches them on (see Auto-timer below). |
| `push_log` | write | `line` (up to 200 characters) and optional `topic`. Sends one `log.line` event to the listeners and says how many received it. |
| `update_settings` `events` | admin | `enabled`, `types` (for example `["ticket.*","timer.*","log.line"]`), `autoTimer` and `autoSystem` (both default true). Chooses what the server announces. |

### System

`run_query` with `schema=system` reads the machine the server is running on, as one row: hostname, CPU count and load, memory, and disk space on `/`. **Inside Docker, Podman or Kubernetes the memory and CPU figures are the container's cgroup limit** (`memory.max`, `cpu.max`), not the host's, because a container otherwise reports the host. On a Mac, memory used leaves out cached pages (free + inactive + speculative from `vm_stat`), because `os.freemem()` counts that cache as used and the gauge would sit near full on a quiet laptop. Disk is the filesystem the process can see. The snapshot carries no environment variables and no network addresses.

The same snapshot is a `system.info` event. `watch_system` publishes it every few seconds (the first at once) and, like `push_timer`, switches that kind on itself when `autoSystem` is true, so a dashboard needs no admin credential. `every_seconds` 0 stops it. `run_query` with `schema=events` reads the buffer the server already announced, newest first, which is how a page shows events and log lines by calling a tool instead of holding a stream open. `filter.type` takes the same patterns a listener uses (`ticket.*`), and a caller only sees kinds they may receive.

The page at `/clients/dashboard.html` draws the gauges, the tickets, the timers and the log. **Tools** (the default) polls `run_query` (schemas `tickets`, `system`, `timers` and `events`). **Events** listens on the stream and moves the gauges when `system.info` arrives; **Publish system info** calls `watch_system`.

| Tool | Scope | What it does |
|---|---|---|
| `watch_system` | read | `every_seconds` (1 to 60, default 5; 0 stops), optional `for_seconds` (1 to 3600) and `topic` (default `system`). One watch at a time. |
| `run_query` `schema=system` | read | One snapshot. `cpu.used_percent` is null the first time and the change since the previous call after that. It does not announce anything. No filter keys; use `fields` to pick parts. |
| `run_query` `schema=events` | read | Newest events from the buffer, `limit` 1 to 50. `filter` by `type` (a name or a group) and `topic`. |

#### Getting events to flow

Three things have to be true: the server announces the kind, a client is listening, and the thing happens. Most "I hear nothing" cases are the first one.

| You want | What you need | How |
|---|---|---|
| **Timer ticks** (the easiest) | A read-scope call | Call `start_timer`, then `push_timer`. If timer events are off, `push_timer` switches them on itself, so **no admin credential is needed**. The answer says `switchedOn: true`. |
| **Machine snapshots** | A read-scope call | Call `watch_system`. If `system.info` is off, `watch_system` switches it on itself (`autoSystem`, default true). Or open `/clients/dashboard.html`. |
| Log lines, ticket events, settings events | An admin credential, once | `update_settings` with `events.enabled=true` and the kinds you want. Or start the server with `ANNOUNCE_EVENTS`, which needs no credential at all. |
| To listen | Nothing, or a key for more kinds | Connect to `/sse` or `/mcp` (see below). An anonymous listener sees the open kinds. |

**Switch events on at startup, with no credential.** Set `ANNOUNCE_EVENTS` in the server's environment. `on` (or `*`) announces every kind; a list announces just those:

```bash
ANNOUNCE_EVENTS=on npx mcp-reference-server --http
ANNOUNCE_EVENTS="timer.*,log.line,ticket.*" MCP_MODE=http PORT=8787 node src/index.js
```

**Get an admin credential** (for `update_settings`, which is admin scope even while auth mode is `off`). Pick one:

1. **Username and password.** On a laptop the default is `demo` / `demo` (set `ADMIN_USER` and `ADMIN_PASSWORD` for anything else; on a public bind the default is disabled). Enter them in the Connection fields on `/admin` → Remote config or on `/clients/browser.html`, or send HTTP Basic: `curl -u "$ADMIN_LOGIN" …`, where `ADMIN_LOGIN` holds the user name and password joined by a colon (on a laptop both are `demo`).
2. **An API key with the admin scope.** On `/admin` → API Keys, tick `admin`, create the key and copy it once. Paste it as the bearer key, or send `Authorization: Bearer <key>`.
3. **Over stdio.** Put the key in `MCP_API_KEY` (or `MCP_USERNAME` and `MCP_PASSWORD`) in the server's env block.

A refusal reads `update_settings requires a credential with the admin scope, even when auth mode is "off"`. It is not a bug and retrying unchanged will not help: add one of the credentials above, or use `push_timer`, or start with `ANNOUNCE_EVENTS`.

**Auto-timer.** `push_timer` switches on `timer.started`, `timer.tick`, `timer.finished` and `timer.stopped` when they are off, and turns announcing on as a whole. It adds only those four kinds and leaves your other choices alone. The switch-on is written to the audit trail and sent as a `settings.changed` event with `by: "push_timer"` (admin listeners see it). An administrator who does not want a read key to do that sets `events.autoTimer=false` with `update_settings`; then `push_timer` answers that nobody will hear the ticks and how to allow it. `run_query` with `schema=settings` shows `events.autoTimer`.

**If you hear nothing**

| Symptom | Cause | Fix |
|---|---|---|
| The status says "The server is not announcing events" | Events are off | Press **Turn events on** in the Events panel (admin), call `push_timer`, call `update_settings` with `events.enabled=true`, or start with `ANNOUNCE_EVENTS=on` |
| `update_settings` is refused: "requires a credential with the admin scope" | No admin credential | See "Get an admin credential" above, or use `push_timer` for timer events |
| `push_log` answers `delivered: false` | `log.line` is not announced | Needs `update_settings` with `events` (admin) or `ANNOUNCE_EVENTS`; `push_timer` does not switch on log lines |
| `Listening over Streamable HTTP` but the list stays empty | Nothing happened yet, or the kind is filtered | Press **Run the demo**; check the Events filter says `*` and the Topic is empty |
| Admin kinds (settings, users, keys) never arrive | The listener is not admin | Connect with an admin credential; the hello message lists what you receive |
| Ticks stop after a while | The timer ended (30 s countdown) or the server restarted | Start another timer; timers and streams are in memory |
| Nothing arrives from `curl -X POST` | A one-shot POST has no stream | Hold a session open with `/sse` or `GET /mcp` |

**The kinds.** `log.line` · `ticket.created`, `ticket.commented`, `ticket.closed`, `ticket.updated`, `ticket.deleted` · `timer.started`, `timer.tick`, `timer.finished`, `timer.stopped` · `settings.changed`, `user.created`, `user.deleted`, `key.issued`, `key.revoked`, `server.reset`. `get_schema` with `name=events` lists them. The admin kinds (`ticket.updated`, `ticket.deleted` and the settings, users, keys and reset kinds) go only to a listener that holds the admin scope. An event never carries ticket text, an email address, a password or a key secret.

**Listening.** An event is a standard MCP logging notification: `notifications/message`, with the kind as `logger` and the event as `data`. A client has to hold a session open, so a one-shot `curl` POST cannot listen.

| Transport | How |
|---|---|
| Legacy SSE | `GET /sse?events=timer.*,log.line&topic=demo`, then POST `initialize` and `notifications/initialized` to the `/messages?sessionId=…` URL the stream names |
| Streamable HTTP | POST `initialize` to `/mcp`, POST `notifications/initialized` with the `Mcp-Session-Id` header, then `GET /mcp?events=…&topic=…` with that header |
| stdio | set `MCP_EVENTS=ticket.*,timer.*` (and `MCP_EVENTS_TOPIC`) in the environment of the process |

`events` takes names and patterns (`*`, `ticket.*`). `topic` keeps only events sent to that topic. The first message on every stream is `events.subscribed`: whether the server is announcing, what this connection receives, and what it asked for but may not have. A listener with no or a wrong key is treated as anonymous, not refused.

```text
event: message
data: {"method":"notifications/message","params":{"level":"info","logger":"timer.tick","data":{"id":3,"type":"timer.tick","at":"2026-10-02T14:50:00.453Z","data":{"timer_id":"TMR-1","label":"demo","state":"running","elapsed_ms":104,"remaining_ms":5896,"every_seconds":2},"topic":"demo"}},"jsonrpc":"2.0"}
```

- **Run the demo.** One button on the Events tab (`/admin#adm-events`) and in the browser client: it turns events on (`log.line`, `ticket.*`, `timer.*`), starts listening, starts the timer chosen in its settings (a 30 s countdown every 5 s unless you change it) and has it report its time, so ticks appear in the list at once. With an admin credential on the connection it switches on the full set; **without one it still works for timers** (`push_timer` switches those on itself) and tells you the log line and ticket events need an admin credential.
- **Timer settings (Countdown or Clock).** Above **Run the demo** the panel has three settings that both **Run the demo** and **Push a timer** use, and the button label shows them (for example `Push a timer (120 s, every 5 s)`). *Timer*: **Countdown** counts down to zero and ends with `timer.finished`; **Clock** is a heartbeat that sends the time as it counts up and is stopped by the server after the time you chose, ending with `timer.stopped`. *Runs for*: 1 to 3600 seconds. *Every*: 1 to 60 seconds. The panel refuses an *Every* longer than *Runs for*. By hand: a countdown is `start_timer {"seconds":120}` then `push_timer {"timer_id":"TMR-1","every_seconds":5}`; a clock is `start_timer {}` (no seconds, a stopwatch) then `push_timer {"timer_id":"TMR-1","every_seconds":5,"for_seconds":120}`.
- **Test it.** The Events panel has **Test the subscription**: it opens a stream with a topic of its own, sends a line with `push_log`, and checks that the line arrives, over either transport.
- Timers that finish or push use real timers on the server. They are **in memory**, and a host that scales to zero drops them and every open stream.
- Events and the settings of which kinds are on are part of the settings row (and so of an export), and so are the MQTT settings (never the broker password).
- Declared once, like the timers: `src/event-tools.js` and `src/events.js`.

#### Logging level (`logging/setLevel`)

Every event goes out at an MCP logging level, and a client chooses the lowest level it wants with `logging/setLevel`. The level is **per connection**: one client can ask for `warning` while another keeps everything. Events below a client's level are not sent to it. A client that never calls it receives everything.

| Level | Events |
|---|---|
| `info` | `ticket.created`, `ticket.commented`, `ticket.closed`, `ticket.updated`, every `timer.*`, and the `events.subscribed` greeting |
| `notice` | `ticket.deleted`, `settings.changed`, `user.created`, `user.deleted`, `key.issued`, `key.revoked` |
| `warning` | `server.reset` |
| any level | `log.line` carries the `level` its sender gave to `push_log` (`debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`; default `info`) |

```bash
# on a Streamable HTTP session (SESSION is the Mcp-Session-Id of the stream you opened)
curl -s -X POST localhost:8787/mcp -H "Mcp-Session-Id: $SESSION" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":9,"method":"logging/setLevel","params":{"level":"warning"}}'
```

The Events panel (admin, browser client) has a **Minimum level** choice next to Listen, and a **Level** next to *Send a line now*. Every event also carries its level in the MQTT payload. The level is not applied to the `events.subscribed` greeting that opens a stream, which is sent at `info`.

#### MQTT

The server can also publish every event to an **MQTT broker**, so anything that speaks MQTT (Node-RED, Home Assistant, a dashboard, `mosquitto_sub`) can follow the desk. It is **publish only** and it brings no new npm package: a small MQTT 3.1.1 client is built in (`src/mqtt.js`), over TCP (`mqtt://host:1883`) or TLS (`mqtts://host:8883`), with QoS 0 or 1. It is not a broker and it does not subscribe: bring your own (mosquitto, EMQX, HiveMQ, a cloud broker). MQTT over WebSocket (`ws://`) is not supported.

```bash
# 1. a broker on this machine (mosquitto), and a listener on every topic of the server
mosquitto_sub -h localhost -t 'mcp-reference-server/events/#' -v

# 2. the server, with events and the bridge on from startup
ANNOUNCE_EVENTS=on MQTT_URL=mqtt://localhost:1883 MCP_MODE=http PORT=8787 node src/index.js

# 3. cause something
curl -s -X POST localhost:8787/mcp -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"push_log","arguments":{"line":"hello broker","level":"warning","topic":"demo"}}}'
```

```text
mcp-reference-server/events/log.line/demo {"id":6,"type":"log.line","at":"2026-10-02T16:38:22.726Z","level":"warning","data":{"line":"hello broker","level":"warning"},"topic":"demo"}
```

- **Topics.** `<prefix>/<kind>`, for example `mcp-reference-server/events/ticket.created`. An event sent to a topic adds that as a last segment (`…/timer.tick/demo`), made safe for MQTT (`+ # /` and spaces become `_`). The default prefix is `mcp-reference-server/events`. Subscribe to `<prefix>/#` for everything, or `<prefix>/timer.tick/#` for one kind.
- **Payload.** The event as JSON: `id`, `type`, `at`, `level`, `data` and `topic` when there is one. Never ticket text, an email, a password or a key.
- **What is published follows Events.** Nothing leaves while events are off, and only the kinds you selected. `push_timer` still switches the timer kinds on by itself, so timer ticks reach the broker too.
- **Admin kinds are never published** (`settings.changed`, `user.*`, `key.*`, `server.reset`, `ticket.updated`, `ticket.deleted`): a broker has no idea who may read what, and anyone who can subscribe to it would see them.
- **Settings.** Admin tab **Events** → **MQTT**, the `set_mqtt` tool, the CLI (`mcp-reference-server mqtt --enabled on --broker mqtt://localhost:1883 --test`), or the environment at startup. `url`, `topic_prefix`, `username`, `password`, `qos` (0 or 1), `retain`, `enabled`; `test=true` publishes one message to `<prefix>/test`. Leaving a field out keeps it. Do not put the user name or password in the URL; it is refused.
- **The password** is kept in memory only. It is not in `run_query schema=settings` (which shows `hasPassword`), the export, the audit trail, the call trace or an event, and an import never carries it. Sending an empty password clears it.
- **The connection.** `run_query schema=settings` → `mqtt.status` shows `state` (`off`, `connecting`, `connected`, `error`), the last error, how many messages were published or failed, and the last topic. A broker that is not there yet is retried with a growing wait (1 s up to 30 s); up to 50 messages wait for the connection. A refused login (wrong user or password) is not retried until you change the settings. An enabled bridge never keeps the process alive.
- **Not supported:** subscribing, MQTT 5, WebSocket, client certificates, a will message.

| Variable | Purpose |
|---|---|
| `MQTT_URL` | Broker, `mqtt://host:1883` or `mqtts://host:8883`. Setting it turns the bridge on. |
| `MQTT_USERNAME` / `MQTT_PASSWORD` | Broker login |
| `MQTT_TOPIC_PREFIX` | Default `mcp-reference-server/events` |
| `MQTT_QOS` / `MQTT_RETAIN` | `0` or `1` / `true` |

### Schema discovery: data, query, tool

`get_schema` with no `name` returns every name with a `kind`, and with a `name` the shape or the tool's arguments, so a model cannot mistake a shape for a tool. One tool does both jobs because a model that has just listed the names needs the next call to be the same tool:

| `kind` | Names | Use |
|---|---|---|
| `data` | `server`, `ticket`, `customer` | A shape only. Not a tool and not a `run_query` name. The result says which tool reads it. |
| `query` | `tickets`, `customers`, `assets`, `timers`, `system`, `settings`, `events`, `users`, `api_keys`, `trace`, `errors`, `counters` | The **only** names `run_query` accepts (12). `users`, `api_keys`, `trace`, `errors` and `counters` need `admin`. |
| `tool` | Every tool name | The argument shape for that tool. It does not run the tool. Call the tool to do the work. |

```
tickets    id · subject · status · requester_email · attribution · created_at
           filterable: status · requester_email · attribution

customers  email · name · plan · region
           filterable: email · region

assets     id · name · site · status
           filterable: status · site

timers     id · label · seconds · state · started_at · ends_at · stopped_at · elapsed_ms · remaining_ms
           filterable: id · state · label
```

### Resources and prompts

```
resources   ticket://TCK-1001     one ticket
            tickets://open        the open list
            schema://tickets      a query shape (tickets | customers | assets | timers — never a tool name)

prompts     search-open-tickets · attribution-scar · schema-discovery
            close-ticket-flow · diagnose-server

skills      skill://work-an-open-ticket/SKILL.md
            skill://open-a-ticket/SKILL.md
            skill://read-the-queue/SKILL.md
```

`attribution-scar` is the prompt that creates a ticket with no requester email, so the service account owns it. Skill files are listed by `skills/list` and read with `resources/read`. They need no credential. Ticket and schema reads follow the same auth as `run_query` and `get_schema`. A denied read is a JSON-RPC error, not a pin-able fake ticket.

The server also sends **instructions** on connect. The model reads them before its first call: start with `describe_server`. If that result says to call `groups/get`, do that with the one group whose description matches the task, then call a tool from the `groups/get` result. If `describe_server` lists every tool, call the tool that matches the task. Pass `requester_email` on `create_ticket`, stop on an empty result, and follow the `next` field once.

### Results and errors

A successful call returns its JSON in the text content **and** in `structuredContent`. A refusal or error sets `isError: true` and carries the JSON in the text content only:

```json
{
  "ok": false,
  "error": "create_ticket requires authentication because auth mode is \"write\". Send Authorization: Bearer <api key> (create one on /admin → API keys) or HTTP Basic with a username and password. Over stdio set MCP_API_KEY in the server env. Call describe_server to see the scopes you hold. Do not retry this call unchanged.",
  "status": 401,
  "denied": true,
  "principal": "anonymous",
  "next": "Call describe_server to see the auth mode and the scopes you hold. Present a credential with the scope named in the error, then retry this tool once."
}
```

A key that lacks a scope gets a `403` that lists the scopes it holds and names the one it needs. Either way the refusal says where to get a key, so the model asks for one instead of retrying.

SEP-3313 (draft) adds `_meta["io.modelcontextprotocol/toolFailure"]` on that same `isError` result. `class` is `capability`, `reliability`, or `governance`. `agentGuidance` repeats `next`. `correlationId` is also stored on the error-log row for a handler error such as an unknown ticket. A rate limit is `reliability` with `retryable: true` and `retryNotBefore`. A client may auto-retry only when that flag is true and the tool's `idempotentHint` is true: `run_query` may, `create_ticket` must not. An argument the SDK rejects before the handler comes back as `isError` text and still has no classification. Any other JSON-RPC error has none either. `input_required` is a task waiting for input, not a tool failure. A deliberate `run_test_job` `tool_error` uses the server code `requested_tool_error`, which is not in the draft's baseline list.

---

## Specifications and the SEP control plane

Every specification rule and Standards Enhancement Proposal (SEP) this server implements has its own switch. All 15 are **on by default**. Turn one off and the server behaves like a server that never implemented it, so you can test how a client copes and audit what it relies on. The definitions live in `SEP_DEFINITIONS` in `src/security.js`.

| SEP | Key | Status | When it is off |
|---|---|---|---|
| 2026-07-28 §2.1 | `handshake` | Accepted | `initialize` answers `-32601`. The server is purely stateless and capabilities travel in `params._meta` on every request |
| SEP-1686 | `tasks` | Accepted | `io.modelcontextprotocol/tasks` is not offered. No task handle, no `tasks/*` methods |
| SEP-2640 | `skills` | Accepted | `io.modelcontextprotocol/skills` is not offered. No `skills/list`, `skills/get`, or `skill://` reads |
| SEP-2575 | `stateless` | Accepted | A one-shot `POST /mcp` without `initialize` answers `-32000` |
| SEP-2567 | `sessionless` | Accepted | A `POST /mcp` without `Mcp-Session-Id` answers `-32000` |
| SEP-2127 | `serverCard` | Accepted | The server-card routes answer `404`, and `initialize` omits `serverInfo.card` |
| SEP-3415 | `mcpEvents` | Draft | `io.modelcontextprotocol/events` is not offered. No `events/list`, `events/poll`, `events/stream` |
| SEP-3313 | `toolFailure` | Draft | Failed tool results carry no `_meta["io.modelcontextprotocol/toolFailure"]` |
| SEP-19 | `groups` | Draft | Groups are not negotiated. `tools/list` returns the full catalog. Groups also need **Enable groups** on `/admin` → Security |
| SEP-2636 | `roots` | Accepted | `roots` is not declared, and `list_workspace_roots` refuses with the SEP to re-enable |
| SEP-1577 | `sampling` | Accepted | `sampling` is not declared, and `draft_reply` refuses with the SEP to re-enable |
| SEP-1036 | `elicitation` | Accepted | `elicitation` and `io.modelcontextprotocol/elicitation` are not declared |
| SEP-1850 | `toolAnnotations` | Accepted | `tools/list` and `groups/get` omit `readOnlyHint`, `idempotentHint`, `destructiveHint`, `openWorldHint` |
| SEP-2549 | `ttlCaching` | Accepted | `tools/list`, `groups/list`, `groups/get`, `skills/list` omit `ttlMs` and `cacheScope` |
| SEP-2164 | `resourceNotFound` | Accepted | A missing resource URI answers `-32603` instead of `-32002` |

Tasks (SEP-1686) is the same switch the tasks extension section calls SEP-2663. The old name `SEP-2663` still resolves to `tasks`.

### Turning a SEP on or off

Any of these takes effect on the next request. A SEP can be named by its id (`SEP-1686`), its key (`tasks`), or an unambiguous id prefix of six characters or more. Aliases `server-card`, `tool-failure` and `events-protocol` also resolve. An unknown name is refused and nothing changes.

| Way | How |
|---|---|
| The desk | `/admin` → Access → **SEPs**. One toggle per SEP, plus **Enable all** and **Accepted only** presets |
| MCP tool | `update_settings` with `seps`, e.g. `{"seps": {"SEP-3313": false, "sampling": false}}`. Admin scope. All or nothing, like every other patch |
| HTTP | `POST /admin/sep` with `sep` and `enabled` for one, `POST /admin/seps` with `preset=all`, `preset=accepted` or a `seps` object for many (needs sign-in). The desk routes tell connected sessions that `tools/list` changed |
| Settings document | `seps` is part of `run_query` `schema=settings`, `/admin/settings/export` and `import_settings` |

```bash
# Turn the draft SEPs off, then back on, from the terminal
npx mcp-reference-server call update_settings --user demo --password demo \
  --args '{"seps":{"SEP-3313":false,"SEP-3415":false,"SEP-19":false}}'
npx mcp-reference-server call update_settings --user demo --password demo \
  --args '{"seps":{"SEP-3313":true,"SEP-3415":true,"SEP-19":true}}'
```

### Server card (SEP-2127)

With `serverCard` on, the server publishes its identity, capabilities, declared extensions and the state of every SEP:

```bash
curl -s http://127.0.0.1:8787/.well-known/mcp/server-card.json   # also GET /mcp/server-card
```

The same card is returned in `initialize` as `serverInfo.card` and by the `server/card` method. The card follows the switches: turn `elicitation` off and its extension drops out of `capabilities.extensions`. `seps` in the card is the key-to-boolean map. The card routes use the same origin check as `/mcp`.

`npm run test:seps` checks this section against the code: the defaults, the card, the handshake, the capability toggles, the annotation and TTL toggles, the `-32002` toggle, and that this README names every tool.

---


## Quick start

### From npm (what an IDE spawns)

```bash
npx mcp-reference-server
# or
MCP_MODE=stdio npx mcp-reference-server
```

Add it to your IDE's `mcp.json`:

```json
{
  "mcpServers": {
    "mcp-reference-server": {
      "command": "npx",
      "args": ["mcp-reference-server"]
    }
  }
}
```

### HTTP desk from npm

```bash
MCP_MODE=http PORT=8787 npx mcp-reference-server
# mcp-reference-server http on 127.0.0.1:8787
```

`PORT` defaults to **8080** when you leave it out.

### From a source checkout

With the server source on disk (the `server/` folder of the source repository):

```bash
cd server
npm install
npm run http       # http://127.0.0.1:8787
```

`npm run http` stops whatever is already listening on the port and starts the desk, so you can rerun it after an edit. Pick another port with `PORT=8799 npm run http`.

| Script | What it does |
|---|---|
| `npm run http` | Desk on `127.0.0.1:8787` (frees the port first) |
| `npm run stdio` | stdio transport. It waits on stdin, so use it from an editor |
| `npm start` | `node src/index.js`, mode from `MCP_MODE` |
| `npm test` | Small smoke test |
| `npm run test:tools` | The full tool suite. 271 checks, starts its own server |
| `npm run test:ops` | The operating tools (`update_settings`, users, keys, `import_settings`, the trace, errors and counters queries, auth modes, gates and scopes) and the Remote config page, including its Demo setup and the hooks the command line uses, and the curl commands on `/help`. 438 checks, starts its own server |
| `npm run test:cli` | The command line client, run for real against its own server, including a check that it and the Remote config page agree on the Demo setup, and that `/clients/cli.html` stays accurate. 252 checks |
| `npm run test:events` | Event streaming, including logging/setLevel on both transports: the kinds, who may receive what, `push_timer` and auto-timer, `push_log`, `update_settings` events, `watch_system`, `run_query` on `system` and `events`, `ANNOUNCE_EVENTS` at startup, real SSE and Streamable HTTP streams, the subscription test and the Events panel. 349 checks |
| `npm run test:mqtt` | The MQTT bridge against a small broker written in the test: the packets, settings checks, topics and payloads, QoS 1, a broker that is late or says no, `set_mqtt`, export and import. 77 checks |
| `npm run test:system` | The machine snapshot: host figures, a Docker cgroup limit, CPU percent, and the system watch. 17 checks |
| `npm run test:groups` | The groups prototype: negotiation, `groups/list`, `groups/get`, an unknown group, a hidden tool that is still callable, omit, the audit trail, groups off, lab generate, and `/health`. 37 checks |
| `npm run test:skills` | SEP-2640: `skills/list`, `skills/get`, `resources/read`, `resources/directory/read`, digests, an unknown skill, the group links, and the curls on `/tools` and `/test`. 22 checks |
| `npm run test:tasks` | SEP-2663: both task tools, the flat handle, `tasks/get`, `tasks/update`, `tasks/cancel`, the error codes, every `run_test_job` outcome, the expiry state, and the panels on `/tools`. 35 checks |
| `npm run test:task-stream` | SEP-2694 draft: `tasks/stream` replays a `run_test_job`, resumes with `after`, refuses a second stream on the same connection, and returns `events_gone` when the cursor is older than the retained log. 11 checks |
| `npm run test:tool-failure` | SEP-3313 draft: class, code, and correlation id on the failures this server already returns. `next` stays. Protocol errors and `input_required` have no class. 11 checks |
| `npm run test:seps` | The [SEP control plane](#specifications-and-the-sep-control-plane): all 15 on by default, the server card, the handshake, the roots/sampling/elicitation, annotation/TTL and `-32002` toggles, and that this README names every tool. 11 checks |
| `npm run test:mcp-events` | SEP-3415 draft: `events/list`, `events/poll` (cursor, filters, truncation, error codes), a short `events/stream`, and the refusal of webhook. 17 checks |
| `npm run test:client` | Sampling (`draft_reply`), roots (`list_workspace_roots` and `notifications/roots/list_changed`), and `notifications/progress` on an open `run_test_job`. A one-shot call has no client. A task does not emit progress notifications. 13 checks |
| `npm run test:skill-upload` | Upload, replace, reject, download, and delete a skill against a temporary `MCP_SKILLS_DIR` |
| `npm run test:all` | Smoke, system, tools, ops, cli, events, mqtt, groups, skills, tasks, task streams, tool failures, protocol events, client, skill upload, and SEPs in a row |
| `npm run test:smoke` | Smoke test against `PORT=8799` |

### Open these in a browser

| Page | What it does |
|---|---|
| `/health` | Liveness and posture. Version, tool count, auth mode, the group count when groups are on, and this machine's CPU, memory and disk. `cwd` only on localhost. JSON at `/health?format=json`. |
| `/test` | Read-only smoke. `/test?write=1` creates and closes a ticket, after admin sign-in. JSON at `/test?format=json`. |
| `/admin` | Sign in (`demo` / `demo` on a laptop). **Access** is Security, Users, API keys, and Tool gates. **Records** is Tickets and Skills. **Operate** is Events, Remote, and Samples. |
| `/log` | Tool counters, admin audit trail, call trace, error log. Admin only. |
| `/tools` | One row per tool: group, when to use it, the scope in force now, a Try button, and the curl that calls it. Tabs for Groups, Skills, Tasks, and Test job run those extensions. Open [All tools](http://127.0.0.1:8787/tools#tools-all). |
| `/help` | Guides. Tabs: Quick start, Walkthrough, Extensions (groups, skills, tasks), Security, Pages, and Tools. The Tools tab has a curl for every tool. Open [Extensions](http://127.0.0.1:8787/help#help-specs). |
| `/clients/` | Copy-paste setup pages for Cursor, VS Code, Claude Desktop, Windsurf, a browser client, a chat page (Ollama, Qwen by default), the command line, and a settings page. |

![/health — HTML dashboard showing status, transport, tool count, auth mode, and raw JSON](../docs/assets/screenshots/health-html.png)

![/test — read-only smoke: seed tickets, schemas, and run_query](../docs/assets/screenshots/test.png)

---

## Command line

The same `mcp-reference-server` command is also a client for a running server, local or deployed. With **no arguments** it is still the MCP server, so every `mcp.json` above keeps working. With a **command** it talks to a server over MCP, the way an agent would, and prints the answer.

```bash
npx mcp-reference-server status                                   # http://127.0.0.1:8787/mcp by default
npx mcp-reference-server status --url https://my-host.example.com # or MCP_URL; /mcp is added for you
npx mcp-reference-server call run_query schema=tickets 'filter={"status":"open"}' limit=5  # any tool, values typed from its schema
npx mcp-reference-server users list --user demo --password-stdin  # the operate tools need an admin credential
npx mcp-reference-server demo apply --user demo --password demo   # the Demo setup from Remote config
```

From a clone, `node src/index.js status` is the same thing.

| Command | What it does |
|---|---|
| `status` | Version, auth mode, rate limit, call trace, transports, users, keys, locked and disabled tools |
| `tools` | The tools the server offers |
| `call <tool> [key=value ...]` | Run any tool. `--args '{...}'` or `--args-file f.json` for nested input. The words win over `--args` |
| `settings [show \| export [--out f] \| import f]` | Read, save or restore the whole settings document |
| `auth-mode [off\|write\|all]` | Who must sign in |
| `rate-limit [--limit n] [--window s] [--on \| --off]` | The per-caller limit |
| `trace [show \| on \| off]` | The call trace. `show --limit 50` lists recent calls |
| `gate <tool> <on\|off>` · `lock <tool> <on\|off>` | Switch a tool off for everyone, or require a credential for it |
| `scope <tool> <read\|write\|pii\|admin\|default>` | Change the scope one tool requires |
| `mqtt [--enabled on\|off] [--broker URL] [--prefix P] [--broker-user U] [--broker-password PW] [--qos 0\|1] [--retain on\|off] [--test]` | Show or set the MQTT bridge. `--user`/`--password` are your login to this server; the `--broker-*` ones are the broker's |
| `protocols [--stdio on\|off] [--http ...] [--sse ...]` | The transports |
| `ui-auth [on\|off]` | Hide the desk pages until an admin signs in |
| `users [list \| create <name> \| delete <name>]` | Users that sign in with HTTP Basic |
| `keys [list \| create \| revoke <id>]` | API keys. A secret is shown once, when it is created |
| `groups [list \| get <name> \| tickets]` | Negotiates the extension. `list` prints the catalog pair and the group names. `get` prints one group's instructions and tool definitions. `tickets` then calls `run_query` for open tickets |
| `demo <apply \| check \| remove>` | Four users, four keys, trace on, sample traffic, auth `write`, 30 calls per 60 s. `check` acts as each role and as nobody. `--skip users,keys,auth,rate,audit,traffic`, `--keys-file f` |

`mcp-reference-server --help` lists every option; `mcp-reference-server help <command>` explains one.

**Connecting.** `--url` (or `MCP_URL`) takes the MCP endpoint or the server's base address, and a pasted `/admin` or `/health` address works too. A credential is `--key` (or `MCP_API_KEY`), or `--user` with a password. A key wins when both are given. The password comes from `MCP_PASSWORD`, from `--password-stdin` (first line), from a hidden prompt at a terminal, or from `--password`, which also works but is visible in your shell history, so the command says so. A local server has the admin `demo` / `demo`. The operate tools (rate limit, users, keys, settings, log) need the `admin` scope even while auth is off. Every call goes through `/mcp`, so the rate limit, scopes and call trace apply to the CLI like any other client.

**Changing things.** Commands that delete, revoke, import, turn a transport off, or apply or remove the demo setup ask first. With no terminal they refuse rather than guess; pass `--yes` (`-y`) to go ahead, for scripts.

**Scripting.** `--json` prints one JSON document on stdout and nothing else, errors included: `{"ok": false, "error": "...", "status": 403, "exitCode": 1}`. Exit codes: `0` done, `1` the server refused or a check failed, `2` usage error, `3` the server could not be reached. Secrets are printed only where you asked for them (`keys create`, `demo apply`) and never in `keys list`, `users list`, errors or `--json` output of the check commands. Examples:

```bash
KEY=$(npx mcp-reference-server keys create --label ci --scopes read --json --key "$ADMIN_KEY" | jq -r .key)
npx mcp-reference-server demo apply -y --keys-file demo-keys.json --user demo --password-stdin <<< demo
npx mcp-reference-server demo check --keys-file demo-keys.json --user demo --password-stdin <<< demo || echo "demo is not set up"
```

---

## Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /mcp` | Streamable HTTP MCP transport (JSON-RPC 2.0) |
| `GET /sse`, `POST /messages` | Legacy SSE MCP transport |
| `GET /health` | Liveness. Version, tool count, group count when groups are on, CPU, memory and disk. `cwd` only on localhost |
| `GET /test` | Read-only smoke. `?write=1` needs admin |
| `GET /tools` | Tool inventory page (`?format=json` for JSON) |
| `GET /help` | Guides |
| `GET /admin` | Admin desk (needs sign-in) |
| `GET /admin/api/status` | Security snapshot, recent tickets and audit as JSON (needs sign-in) |
| `GET /admin/settings/export`, `POST /admin/settings/import` | Settings document in and out (needs sign-in) |
| `GET /admin/log/export` | Counters, errors and trace as JSON (needs sign-in) |
| `GET /log` | Call trace page (needs sign-in) |
| `GET /clients/…` | Client setup pages, `theme.css`, `theme.js`, `icon.svg`, `favicon.ico` |
| `GET /.well-known/mcp/server-card.json`, `GET /mcp/server-card` | Server card (SEP-2127). `404` while that SEP is off. See [Server card](#server-card-sep-2127) |
| `POST /admin/sep`, `POST /admin/seps` | Switch one SEP, or apply `preset=all`, `preset=accepted` or a `seps` object (needs sign-in). See [Turning a SEP on or off](#turning-a-sep-on-or-off) |

Sign-in sets a session cookie named `mcp_admin`.

---

## curl reference

All examples assume the server is on `http://127.0.0.1:8787`. Replace the host with the URL of your deployed server (for example `https://<your-app>.onrender.com`) for remote testing.

**`/mcp` needs both `Accept` values.** `Accept: application/json` alone is refused with `Not Acceptable: Client must accept both application/json and text/event-stream`.

### Health and smoke test

```bash
curl -s "http://127.0.0.1:8787/health?format=json" | jq .

# Public read-only smoke: describe, schemas, query. No writes.
curl -s "http://127.0.0.1:8787/test?format=json" | jq .

# Exit-code style check: 200 = all steps passed, 500 = a step failed
curl -s -o /dev/null -w "%{http_code}" "http://127.0.0.1:8787/test?format=json"
```

### MCP tool calls over HTTP (JSON-RPC 2.0)

```bash
BASE=http://127.0.0.1:8787
MCP=(-H "Content-Type: application/json" -H "Accept: application/json, text/event-stream")

# Discover the server. Always call this first.
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"describe_server","arguments":{}}}' | jq .

# List all tools (22). A client that did not negotiate groups gets the flat catalog.
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | jq .

# Groups prototype: one-shot client declares the extension, so tools/list shrinks
# to describe_server and get_schema. groups/list needs no credential.
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":20,"method":"groups/list","params":{"_meta":{"io.modelcontextprotocol/clientCapabilities":{"extensions":{"io.modelcontextprotocol/groups":{}}}}}}' | jq .
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":21,"method":"groups/get","params":{"name":"query","_meta":{"io.modelcontextprotocol/clientCapabilities":{"extensions":{"io.modelcontextprotocol/groups":{}}}}}}' | jq .

# Find open tickets (run_query, schema tickets)
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"run_query","arguments":{"schema":"tickets","filter":{"status":"open"},"limit":5}}}' | jq .

# Create a ticket WITH requester_email (correct)
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"create_ticket","arguments":{"subject":"Test from curl","body":"Sent via terminal.","requester_email":"ada@example.com"}}}' | jq .

# Create a ticket WITHOUT requester_email. The call succeeds and the service account owns it.
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"create_ticket","arguments":{"subject":"No requester","body":"No email. The service account will own this."}}}' | jq .

# Get one ticket (all fields, with body and comments)
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":6,"method":"tools/call","params":{"name":"run_query","arguments":{"schema":"tickets","filter":{"id":"TCK-1001"}}}}' | jq .

# Add a comment
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"add_comment","arguments":{"ticket_id":"TCK-1001","body":"Confirmed from the terminal.","author":"ada@example.com"}}}' | jq .

# Schema discovery, then a query (get_schema with no name lists every name)
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":8,"method":"tools/call","params":{"name":"get_schema","arguments":{}}}' | jq .

curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":9,"method":"tools/call","params":{"name":"get_schema","arguments":{"name":"tickets"}}}' | jq .

curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":10,"method":"tools/call","params":{"name":"run_query","arguments":{"schema":"tickets","filter":{"status":"open"},"limit":5}}}' | jq .

# Read a customer: the phone number is REDACTED without the pii scope
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":11,"method":"tools/call","params":{"name":"run_query","arguments":{"schema":"customers","filter":{"email":"ada@example.com"}}}}' | jq .

# Resources and prompts
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":12,"method":"resources/list","params":{}}' | jq .

curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":13,"method":"resources/read","params":{"uri":"ticket://TCK-1001"}}' | jq .

curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":14,"method":"prompts/list","params":{}}' | jq .
```

The tool result is JSON-RPC whose `content[0].text` is itself a JSON string. To read a success as plain JSON, pull out `structuredContent`. A refusal has no `structuredContent`, so parse the text:

```bash
# success
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":15,"method":"tools/call","params":{"name":"describe_server","arguments":{}}}' \
  | jq '.result.structuredContent'

# refusal or error: isError is true, the JSON is in the text
curl -s -X POST $BASE/mcp "${MCP[@]}" \
  -d '{"jsonrpc":"2.0","id":16,"method":"tools/call","params":{"name":"run_query","arguments":{"schema":"tickets","filter":{"id":"TCK-9999"}}}}' \
  | jq '.result | {isError, body: (.content[0].text | fromjson)}'
```

### Authenticated calls (auth mode `write` or `all`)

```bash
KEY=mcpk_your_api_key_here

# Authorization header
curl -s -X POST $BASE/mcp "${MCP[@]}" -H "Authorization: Bearer $KEY" \
  -d '{"jsonrpc":"2.0","id":16,"method":"tools/call","params":{"name":"create_ticket","arguments":{"subject":"Authenticated ticket","body":"Sent with an API key.","requester_email":"ada@example.com"}}}' | jq .

# x-api-key header (alternative)
curl -s -X POST $BASE/mcp "${MCP[@]}" -H "x-api-key: $KEY" \
  -d '{"jsonrpc":"2.0","id":17,"method":"tools/call","params":{"name":"run_query","arguments":{"schema":"tickets","filter":{"status":"all"}}}}' | jq .

# HTTP Basic with a user (USER_LOGIN is a login from MCP_USERS, name and password joined by a colon)
curl -s -X POST $BASE/mcp "${MCP[@]}" -u "$USER_LOGIN" \
  -d '{"jsonrpc":"2.0","id":18,"method":"tools/call","params":{"name":"describe_server","arguments":{}}}' | jq '.result.structuredContent.you'
```

### Admin from the terminal

```bash
# 1. Sign in and keep the session cookie
curl -s -c /tmp/mcp_cookies.txt -X POST $BASE/admin/login -d 'username=demo&password=demo'

# 2. Set the auth mode to write
curl -s -b /tmp/mcp_cookies.txt -X POST $BASE/admin/security -d 'authMode=write'

# 3. Read the security snapshot as JSON
curl -s -b /tmp/mcp_cookies.txt $BASE/admin/api/status | jq .security

# 4. Export settings and the call log
curl -s -b /tmp/mcp_cookies.txt $BASE/admin/settings/export | jq .
curl -s -b /tmp/mcp_cookies.txt $BASE/admin/log/export | jq .

# 5. Fill the store and the log (Lab panel)
curl -s -b /tmp/mcp_cookies.txt -X POST $BASE/admin/generate-data -d 'count=10'
curl -s -b /tmp/mcp_cookies.txt -X POST $BASE/admin/generate-traffic -H "Accept: application/json" -d 'rounds=5' | jq .
```

The same operations are also tools: `update_settings` (with `authMode`), `run_query` with `schema=settings`, `trace`, `errors` or `counters`, and `generate_traffic`. All but the last need an `admin` credential, even when auth mode is `off`. Set `ADMIN_LOGIN` in your shell first to the admin user name and password joined by a colon (on a laptop both are `demo`). For example:

```bash
curl -s -X POST $BASE/mcp "${MCP[@]}" -u "$ADMIN_LOGIN" \
  -d '{"jsonrpc":"2.0","id":19,"method":"tools/call","params":{"name":"update_settings","arguments":{"authMode":"write","rateLimit":{"enabled":true,"limit":60,"windowSeconds":60}}}}' | jq .
```

From the command line: `mcp-reference-server call update_settings authMode=write 'rateLimit={"enabled":true,"limit":60,"windowSeconds":60}'`, and `mcp-reference-server call run_query schema=trace`.

---

## Auth model

```
mode   effect
─────  ──────────────────────────────────────────────────────────
off    All tools callable with no credential. The laptop default.
write  Read tools are open. Write tools need a credential.
all    Every tool call needs a credential.
```

Set the mode with `AUTH_MODE`, on `/admin → Security`, or with `update_settings` (`authMode`). The older `WRITE_TOOLS_LOCKED=1` is read as `write`.

### Scopes

| Scope | Lets you call |
|---|---|
| `read` | Read tools: schemas, `run_query` (tickets, customers, settings and the other schemas) |
| `write` | Write tools (`create_ticket`, `add_comment`, `close_ticket`). Includes `read`. |
| `pii` | Shows the customer `phone` that `run_query` otherwise returns as `REDACTED`. No tool needs `pii`. Includes `read`. |
| `admin` | Everything, including `update_settings`, `set_mqtt`, the user, key and import tools, and the `users`, `api_keys`, `trace`, `errors` and `counters` queries. |

### Credentials

```
HTTP   Authorization: Bearer <api key>
       Authorization: Basic base64(user:pass)
       x-api-key: <api key>

stdio  MCP_API_KEY=<key>                        in the server process env
       MCP_USERNAME + MCP_PASSWORD              in the server process env
```

Issue API keys on `/admin → API Keys`, with `issue_api_key`, or register one at boot with `API_KEY` and `API_KEY_SCOPES`. Users come from `/admin → Users`, `create_user`, or `MCP_USERS="alice:secret:read,write"`.

**An unknown key is refused, even in `off` mode.** Any Bearer or `x-api-key` value is looked up as an API key. A made-up token gets `401` on every tool except `describe_server`. Send a real key, or send nothing.

### What gets checked, in order

1. The tool is switched off by a gate → `503`
2. A credential was sent and it is not recognised → `401` (`describe_server` is exempt)
3. `TENANT_ID` is set and a write or admin call has no matching `x-tenant-id` → `403`
4. The mode or a tool lock needs a credential and there is none → `401`
5. The credential lacks the scope → `403`, naming the scope
6. The caller is over the rate limit → `429` with the seconds to wait

`describe_server` stays open in every mode. A client that cannot ask "what do you need from me?" can only guess.

### Per-tool gates and locks

- **Gate** (`/admin → Tool Gates`, `update_settings` with `toolGates`) switches one tool **off**. It answers `503`.
- **Lock** (`update_settings` with `toolAuthOverrides`) requires a credential for one tool, whatever the auth mode.
- **Scope** (`/admin → Tool Gates`, `update_settings` with `toolScopeOverrides`, CLI `scope <tool> <scope>`) changes which scope a tool needs: `read`, `write`, `pii` or `admin`; `default` restores the tool's own. `describe_server` shows the result as `required_scope`, with `own_scope` next to it when it differs.
  - **Raising** a scope (say `run_query` to `pii`) makes the tool need a credential with that scope, even while the auth mode is `off`.
  - **Lowering** one to `read` opens it in modes `off` and `write` (say `create_ticket` to `read`). Mode `all` still wants a credential.
  - **Admin tools cannot change.** Neither can `describe_server` (always open) or `generate_traffic` (any credential). That is why a lowered scope can never open the control plane.
  - Overrides are part of the settings row and so of `import_settings` (`toolScopeOverrides`). A document without that field leaves them alone; an empty map clears them. Like the rest of the settings they live in memory.
- `describe_server` can be neither gated nor locked.
- Saving a gate, lock or auth-mode change sends `notifications/tools/list_changed` to connected SSE and Streamable HTTP sessions.

### Rate limit

`RATE_LIMIT` calls per `RATE_LIMIT_WINDOW_MS` for each caller (default 60 per minute, one fixed window). A refused call says how many seconds to wait. `describe_server` shows the remaining budget.

### PII

No tool needs `pii`. `run_query` with `schema=customers` is a `read` call, and the `phone` field comes back as the text `REDACTED` unless the caller presents a credential holding `pii`. It is the same tool and the same endpoint; the scopes decide which fields you see, not whether the call works.

### Naive mode (demo)

`/admin → Security → Naive mode` makes anonymous denials on write tools return a bare `403` with no scope and no `next`. A model then retries in a loop. Turn it off and the same call gives the actionable refusal. It exists to show that contrast, and it is off by default.

### Desk sign-in

Optional, off by default. Turn it on from **Admin → Security → Require sign-in for the desk**, with `update_settings` `uiAuth=true`, with `npx mcp-reference-server ui-auth on`, or by starting the process with `UI_AUTH=1`.

While it is on, `/health`, `/tools`, `/test`, `/help` and `/clients` answer with the sign-in page. `/admin` and `/log` always do. After a correct password the browser returns to the page that asked. `/mcp`, `/sse` and `?format=json` stay open, and tool calls are unchanged.

### Public deployments

- `demo` / `demo` is for a laptop. When the server binds publicly (`CONTAINER`, `CODE_ENGINE_PROJECT`, or `HOST=0.0.0.0`) **admin sign-in stays disabled until `ADMIN_PASSWORD` is set**.
- `/health` hides `cwd` off localhost, and `/test` stays read-only.
- Requests with an `Origin` header are accepted from localhost, the same host, or an origin listed in `CORS_ORIGINS`.
- Use `AUTH_MODE=write` (or `all`) on anything with a public URL.
- Tickets, users, keys and settings live **in memory**. A restart clears them. User passwords and API-key secrets are held as hashes.

---

## Settings document

`run_query` with `schema=settings` (or `/admin/settings/export`) returns the settings row. `import_settings` takes that row as it is. The row looks like this (abridged: it also carries `events` and `mqtt`, and a saved export from `/admin/settings/export` adds `kind`):

```json
{
  "kind": "mcp-reference-server-settings",
  "version": "1.0.0",
  "authMode": "write",
  "rateLimit": { "enabled": true, "limit": 60, "windowSeconds": 60 },
  "audit": false,
  "uiAuth": false,
  "groups": true,
  "seps": { "handshake": true, "tasks": true, "toolFailure": true, "…": "one boolean per SEP key" },
  "protocols": { "stdio": true, "streamableHttp": true, "sse": true },
  "toolGates": { "describe_server": true, "create_ticket": true, "…": "one boolean per tool" },
  "toolAuthOverrides": { "describe_server": false, "create_ticket": false, "…": "one boolean per tool" },
  "toolScopeOverrides": {},
  "users": [],
  "activeKeyCount": 0
}
```

`import_settings` restores auth mode, rate limit, audit, UI sign-in, the groups switch, the SEP switches, protocols, gates, locks and scope overrides (a field left out is left alone). It does **not** restore users or keys, and the export never contains their secrets. The document's `kind` must match, and `describe_server` is never gated or locked by an import.

---

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `MCP_MODE` | `stdio` | `stdio` or `http` |
| `PORT` | `8080` | HTTP port. `npm run http` uses `8787` |
| `HOST` | `127.0.0.1` | `0.0.0.0` inside a container (set automatically) |
| `AUTH_MODE` | `off` | `off` / `write` / `all` |
| `ADMIN_USER` / `ADMIN_PASSWORD` | `demo` / `demo` | `/admin` login. On a public bind, sign-in stays off until `ADMIN_PASSWORD` is set |
| `CORS_ORIGINS` | unset | Extra `Origin` values allowed on `/mcp`. Localhost and the same host are always allowed |
| `API_KEY` / `API_KEY_SCOPES` | unset / `read,write` | Register one key at boot |
| `MCP_USERS` | unset | `"alice:secret:read,write"`, comma-separated, for extra logins |
| `MCP_API_KEY` | unset | stdio: the key this process presents |
| `MCP_USERNAME` / `MCP_PASSWORD` | unset | stdio: the Basic credential this process presents |
| `RATE_LIMIT` / `RATE_LIMIT_WINDOW_MS` | `60` / `60000` | Calls per window per caller |
| `RATE_LIMIT_ENABLED` | `1` | `0` turns rate limiting off |
| `ANNOUNCE_EVENTS` | unset | `on` or a list such as `timer.*,log.line`: announce events from startup |
| `MQTT_URL` (+ `MQTT_USERNAME`, `MQTT_PASSWORD`, `MQTT_TOPIC_PREFIX`, `MQTT_QOS`, `MQTT_RETAIN`) | unset | Publish the events to an MQTT broker. See [MQTT](#mqtt) |
| `MCP_SKILLS_DIR` | `~/.mcp-reference-server/skills` | Folder of uploaded skills. Read at startup and after each upload or delete. Tickets stay in memory |
| `MCP_TASK_TTL_MS` | `60000` | How long a task handle lives. A `run_test_job` task adds its `seconds` to this |
| `MCP_TASK_EXPIRE_RETAIN_MS` | `60000` | How long an expired task stays queryable. `tasks/get` in that window is `-32602` with `data.status` `expired`. Then the id is not found |
| `MCP_TASK_EVENT_LIMIT` | `64` | How many `tasks/stream` events are kept per task. A cursor older than that is `-32010` `events_gone` |
| `TENANT_ID` | unset | Writes and admin calls also need an `x-tenant-id` header with this value. stdio sends it for you |
| `CONTAINER` / `CODE_ENGINE_PROJECT` | unset | Mark a cloud or container run: bind `0.0.0.0` and apply the public-deployment rules |

---

## The desk

### Admin

| Tab | What it does |
|---|---|
| Security | Auth mode, rate limit, audit mode, the groups switch, settings export and import |
| SEPs | One switch per specification and SEP, with Enable all and Accepted only presets. See [the SEP control plane](#specifications-and-the-sep-control-plane) |
| API keys | Issue a key (shown once), revoke, see scopes, call count and last use |
| Tool gates | Switch a tool off, or require a credential for it |
| Users | Create and delete users with scopes |
| Tickets | Newest 50: create, edit, delete. Factory reset back to the seed tickets |
| Skills | Built-in skills plus uploads. Download one, all, or a blank template |
| Events | Which events the server announces, and the MQTT broker |
| Remote | Call another MCP server: rate limit, users, keys, gates, sample setup |
| Samples | Add sample tickets or record sample calls, including **Generate via groups**. Also stop or restart the server process (restart works only when started with `node` or `npx`, not in a container) |

![/admin — sign-in page (laptop default: demo / demo)](../docs/assets/screenshots/admin-login.png)

![/help — Quick start, Walkthrough, Extensions, Security, Pages, and Tools](../docs/assets/screenshots/help.png)

### Samples (tickets and recorded calls)

The Lab fills the store and produces log entries without an LLM:

- **Generate data** seeds up to 40 realistic tickets with mixed statuses, emails and comments. A few omit `requester_email`, so the service account owns them. It writes the store directly.
- **Generate via groups** creates the same kind of tickets by negotiating the extension and calling `create_ticket` (and sometimes `add_comment` and `close_ticket`) inside `tickets-write`. The desk's admin credential is the caller, so writes succeed when auth is `write` or `all`. Groups off refuses the request. Pending status is not set this way. The rate limit applies.
- **Generate traffic** runs N rounds of scripted calls covering every log category: happy reads, successful writes, bad ticket ids (error log), invalid credentials (denied counter), valid queries, and unknown-schema queries (error log). Turn audit mode on first to capture the full call trace on `/log`.

### Log

`/log` shows the tool counters (success, error, denied per tool), the admin audit trail, the error log of the last 50 failed calls, and, with audit mode on, the trace of the last 200 calls (the same data as `run_query` with `schema=trace`, `errors` or `counters`, which give up to 50 rows per call). `groups/list`, `groups/get` and a negotiated `tools/list` appear on that trail. Turning audit on clears the trace, so turn it on before the calls you want to see. Secrets in arguments (`password`, `api_key`, `secret`, `document`) are recorded as `[redacted]`.

Every table on `/log` sorts: click a column header, click again to reverse. Times are stored in UTC and shown in the **browser's own timezone** (for example `9:24:23 PM` rather than `01:24:23`; an earlier day adds the date). Hover a time for the full local date, the zone name and the original UTC value. `/admin/log/export` (the download route) keeps the raw UTC ISO timestamps.

### Try a tool

On `/tools`, every tool row has the curl that calls it, with a copy button, and **Try** opens a dialog with example arguments, calls `/mcp`, and shows the response pretty-printed with the inner JSON unpacked. Admin tools and `generate_traffic` include `-u demo:demo` in that curl. The Groups, Skills, Tasks, and Test job tabs run the extension sequences.

### Themes

The header on every desk and client page has a theme switch: **project**, **light** or **dark**. The choice is saved in the browser (`localStorage` key `mcp-desk-theme`) and applied before the page paints, so there is no flash.

### Client pages

`/clients/` has a page each for Cursor, VS Code, Claude Desktop, Windsurf, a browser client, a chat page, the [command line](#command-line) (`cli.html`), and a settings client that queries `settings` and calls `update_settings` (saving needs an admin key). The chat page asks Ollama (Qwen by default, on this machine at port 11434) and then calls the tools. The MCP URL can be this desk or a deployed MCP server. The desk only forwards model calls to `http://127.0.0.1:11434`. Each one can call the server and shows the request and the raw response. The browser client has forms to read and set the rate limit and call trace, list and create users and API keys (with scope checkboxes, a one-time key box and a "Use as bearer key" button), search tickets, and call any tool from a form built from that tool's schema. A self-test creates a throwaway user and API key, checks the password and secret are never listed, then deletes and revokes them. The operate tools need an `admin` credential even when auth is off, so the page takes a bearer key or a username and password (HTTP Basic, `demo` / `demo` on a laptop).

The same forms are built into the desk as **Admin → Remote config**. Point its MCP URL at this server or at any deployed one (Render, Code Engine, your own host) and manage that server from the page: rate limit, users, API keys, any tool. The page sends its own credential to `/mcp`, since the admin sign-in cookie is not accepted there, and it never saves the credential. A server on another site answers the page only for localhost, its own host or an origin in its `CORS_ORIGINS`; the tab's **Browser access** section builds that line and runs a test from your browser. The tab was called Browser before; old `#adm-browser` links still open it.

**Demo setup** at the top of the tab turns a fresh server into something you can test every feature on. **Apply demo setup** creates four users (`demo-reader`, `demo-writer`, `demo-pii`, `demo-admin`, password `demo-pass`) and four labelled API keys (read, write, pii, admin), turns the call trace on, records sample traffic, sets auth mode to `write` and the rate limit to 30 calls per 60 s. Each key is shown once, with Copy and Use as bearer key. It is safe to run twice: the old demo users and keys are replaced. **Check demo setup** then tests it by acting as each role, anonymous included: who can read, write, see PII or use admin tools, that a wrong password is refused, that passwords and secrets never appear in lists or the log. It leaves one solved "Demo check" ticket behind. **Remove demo setup** revokes the keys, deletes the users and sets auth mode off, the trace off and the rate limit back to 60 per 60 s. Untick a box to skip that part. Saving a rate limit resets every caller's budget, so Apply sets it last and Remove sets it first; if a caller has used its whole budget, Remove stops at Connect and shows the wait time.

---

## What this teaches

```
Lesson 1 — Put the nouns in a schema, keep the verbs as tools
  Tickets are read with one tool: run_query schema=tickets, and a
  filter says which. The tools left are verbs: create_ticket,
  add_comment, close_ticket. Compare to the first draft:
  query_tickets_by_status / query_tickets_by_requester / query_open_tickets.
  Technically correct. The model picked wrong every other call.

Lesson 2 — The silent tool-count failure
  tools/list returns 0 tools with no error when the server starts
  but the cwd is wrong or MCP_MODE is missing. Nothing fails.
  Nothing warns. The tool list is just empty.

Lesson 3 — A successful call can still leave the wrong owner
  create_ticket without requester_email still succeeds.
  The service account owns the ticket and every reply goes
  to the bot, not the customer. Pass requester_email.
  Inside a task, omitting it waits at input_required instead.

Lesson 4 — Schema discovery over tool proliferation
  get_schema → run_query replaces a pile of
  query_tickets / query_assets / query_with_filter tools.
  And every name carries a kind — data, query or tool — so a
  shape is never mistaken for a tool, and a tool is never
  passed to run_query.

Lesson 5 — The protocol doesn't say who is allowed to call it
  MCP describes tools. It doesn't describe permission. This server
  invents its own: off / write / all modes, four scopes, keys and
  users, gates and locks, and refusals that name the scope and
  where to get a key, so the model asks instead of looping.

Lesson 6 — Laptop paths don't survive a container boundary
  Native stdio uses an absolute local cwd. Podman and Code
  Engine use the image. The path that worked on your laptop
  is meaningless in the container. And the IDE config files
  (.vscode/ .cursor/ .bob/ and the global Windsurf and Cline
  files) all drift independently once a URL changes.

Lesson 7 — Results are instructions
  Every result and every error has a next field that names one
  follow-up tool. The model follows it instead of guessing, and the
  same sentence tells a human what to do.
```

---

## Connecting an MCP client

Any MCP-capable client (Bob, VS Code, Cursor, Windsurf, Cline, Claude Desktop) can use this server. Add it to the client's `mcp.json` (the file name and location differ per client; see [docs/BOB.md](../docs/BOB.md) for the paths).

Native stdio, the client spawns the server:

```json
{
  "mcpServers": {
    "mcp-reference-server": {
      "command": "npx",
      "args": ["-y", "mcp-reference-server"]
    }
  }
}
```

Streamable HTTP, against a server that is already running (add `headers` with `Authorization: Bearer <api-key>` when auth is on):

```json
{
  "mcpServers": {
    "mcp-reference-server": {
      "type": "streamable-http",
      "url": "http://127.0.0.1:8787/mcp"
    }
  }
}
```

![IBM Bob MCP settings — mcp-reference-server connected and enabled](../docs/assets/screenshots/Ide-mcp-enabled.png)

To check the connection, open `/health` and `/test` on the desk, or run `npx mcp-reference-server status`.

---

## Requirements

- Node.js >= 18

## Dependencies

- `@modelcontextprotocol/sdk` — MCP server SDK
- `express` — HTTP transport and the desk
- `zod` — schema validation

---

## Author

**Markus van Kempen**

| | |
|---|---|
| 🌐 | [markusvankempen.github.io](https://markusvankempen.github.io/) |
| 💻 | [github.com/markusvankempen/mcp-reference-server](https://github.com/markusvankempen/mcp-reference-server) |

> *No bug too small, no syntax too weird.*

*Personal open-source demo.*
