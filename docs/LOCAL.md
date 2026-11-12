# Local — native stdio and laptop HTTP

## Native stdio (what the editor uses)

This is **not Docker**. The client launches `node` as a child process. There is **no port and no URL**. JSON-RPC 2.0 goes over stdin/stdout.

```bash
cd mcp-reference-server/server
npm install
npm run stdio          # MCP_MODE=stdio node src/index.js
```

`cwd` is the server package so `node_modules` resolve. Relative `src/index.js` — not a laptop absolute path, not a container image.

Cursor / VS Code / IBM Bob spawn that after the mcp files are in place.

`.vscode/mcp.json`:

```json
{
  "servers": {
    "mcp-reference-server": {
      "type": "stdio",
      "command": "node",
      "args": ["src/index.js"],
      "cwd": "${workspaceFolder}/mcp-reference-server/server",
      "env": { "MCP_MODE": "stdio" }
    }
  }
}
```

`.cursor/mcp.json` and `.bob/mcp.json` use the same command with `cwd: "mcp-reference-server/server"`. Bob also lists the six ticket-desk tools in `alwaysAllow`.

A Docker / Code Engine remote is a **second** server (`mcp-reference-server-remote`). Connecting remote does not replace native stdio.

Reload the editor window after the file changes so the client respawns the process.

## HTTP on the laptop (diagnostics)

stdio has no `/health`. For the pages, run the same code as a server:

```bash
cd mcp-reference-server/server
MCP_MODE=http PORT=8787 node src/index.js
```

| URL | What it is |
|---|---|
| http://127.0.0.1:8787/health | Process up? version? group count when groups are on? CPU, memory, disk? `cwd` on localhost |
| http://127.0.0.1:8787/test | Read-only smoke: schemas, `run_query`, seed tickets (`?write=1` needs admin sign-in) |
| http://127.0.0.1:8787/admin | `demo` / `demo`. Access, Records, Operate |
| http://127.0.0.1:8787/tools | Every tool, with its curl. Tabs for groups, skills, tasks, and the test job |
| http://127.0.0.1:8787/help#help-specs | Reference for groups, skills, and tasks |
| http://127.0.0.1:8787/sse | Local SSE (same path as Code Engine) |
| http://127.0.0.1:8787/mcp | Streamable HTTP |

## From the terminal

With the HTTP server running, the same `mcp-reference-server` command is a client for it:

```bash
cd mcp-reference-server/server
node src/index.js status                       # or: npx mcp-reference-server status
node src/index.js groups                        # negotiate, then the catalog pair and the seven group names
node src/index.js groups get query              # instructions and the run_query definition
node src/index.js groups tickets                # groups/get query, then open tickets
node src/index.js call run_query schema=tickets 'filter={"status":"open"}' limit=5
node src/index.js users list --user demo --password-stdin     # type demo, Enter
node src/index.js demo apply --user demo --password demo     # then: demo check, demo remove
```

The default target is `http://127.0.0.1:8787/mcp` (`--url` or `MCP_URL` to change it). Run with **no arguments** it is the stdio server, which is what the `mcp.json` entries above start, so adding the command line did not change them. The operate commands (`users`, `keys`, `rate-limit`, `settings`) need an admin credential: the built-in `demo` / `demo` on a laptop. `groups` negotiates `io.modelcontextprotocol/groups` on each call. The extensions are [groups](../server/README.md#groups-prototype), [skills](../server/README.md#skills) (SEP-2640), [tasks](../server/README.md#tasks) (SEP-2663), and [events](../server/README.md#events-extension) (SEP-3415, draft). Every command is in [server/README.md](../server/README.md#command-line).

**Tests:** from `server/`, `npm run test:all` runs every suite (smoke, system, tools, ops, CLI, events, MQTT, groups, skills, tasks, task streams, tool failures, protocol events, client, and skill upload). With [mise](https://mise.jdx.dev/) installed, `mise run test-all` from the repo root does the same and pins Node 22. `mise run test-groups` runs only the groups prototype.

## A real call with real output

Ask the agent:

> Search open tickets. Then create a ticket for ada@example.com with subject "Need the remote URL". Then create a second ticket *without* a requester and tell me who owns each one.

Expected:

1. At least TCK-1001 / 1002 / 1003 from the seed data.
2. New ticket owned by `ada@example.com`.
3. Second ticket owned by `mcp-bot@service.local`, status 201, warning in the tool result.

## Edge cases written down

| What you see | What it means | What to try |
|---|---|---|
| Empty `run_query` on `tickets` | No rows for those filters | Drop the `status` or `requester_email` filter (no `status` lists every ticket). Do not retry the same filters |
| `create_ticket` 201 + warning | Service account is the requester | Pass `requester_email` |
| Write tool says it is locked | `/admin` lock is on | Token in `Authorization: Bearer` or unlock |
| Nothing answers on `/health` | HTTP server is not running | Start it with `MCP_MODE=http` (see above) |
