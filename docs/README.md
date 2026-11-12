# Documentation

Personal open-source project. Server **mcp-reference-server 1.0.0** (22 tools). It adds four extensions, plus sampling, roots, and progress, documented in [server/README.md](../server/README.md):

- [Groups](../server/README.md#groups-prototype) — `io.modelcontextprotocol/groups`, a prototype of the Primitive Grouping draft. Not a final SEP. Not in published npm 3.0.1.
- [Skills](../server/README.md#skills) — `io.modelcontextprotocol/skills`, SEP-2640. Workflows and files. A skill is not a tool.
- [Tasks](../server/README.md#tasks) — `io.modelcontextprotocol/tasks`, SEP-2663. `create_ticket` and `run_test_job` can return a handle the client polls. `tasks/stream` (SEP-2694, draft) replays that task's events. After `ttlMs`, `tasks/get` reports `data.status` `expired` until the retain window ends.
- [Events extension](../server/README.md#events-extension) — `io.modelcontextprotocol/events`, SEP-3415 draft. `events/list` and `events/poll` on the existing bus. Webhook is not offered.
- [Sampling, roots, and progress](../server/README.md#asking-the-connected-client) — `draft_reply`, `list_workspace_roots`, and `notifications/progress` on an open `run_test_job`. Core protocol, not an extension.

With the desk running, the same reference is at [Docs → Extensions](http://127.0.0.1:8787/help#help-specs). [DEMO.md](DEMO.md) still describes the published 3.0.1 build, so version strings in its captured output can differ from what 1.0.0 prints.

All curl examples assume a laptop HTTP server at **http://127.0.0.1:8787** unless a doc says otherwise.

| Doc | What's in it |
|---|---|
| [DEMO.md](DEMO.md) | **16 live curl demos** plus one-command setup — health, smoke, discovery, `run_query`, attribution, auth, events, MQTT, real captured JSON |
| [LOCAL.md](LOCAL.md) | Native stdio and laptop HTTP — `mcp.json`, `/health` `/test` `/admin`, CLI including `groups`, tests |
| [ADMIN-AND-SECURITY.md](ADMIN-AND-SECURITY.md) | Auth modes, API keys, scopes, tool gates, the groups switch, the SEP switches, `update_settings` patch |
| [BOB.md](BOB.md) | IBM Bob — `.bob/mcp.json`, Streamable HTTP `/mcp`, auto-approve |

Server API reference: [../server/README.md](../server/README.md).
