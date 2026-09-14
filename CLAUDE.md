# kanboard-mcp — Claude development guide

This is a Go MCP server that bridges the Kanboard JSON-RPC API to MCP-aware AI
clients (Claude Code, Claude Desktop, etc.). The active outstanding work is
listed in `BUGS.md`.

## Architecture

Single-file server. All code lives in `main.go`:

- Tool registrations (`s.AddTool(...)`) — declare each MCP tool's schema
- Handler functions (`*Handler`) — translate MCP calls → Kanboard JSON-RPC
- `RBACManager` — MCP-side permission gate (separate from Kanboard's own ACL)
- `kanboardClient` — HTTP client + auth header builder

Build artifact is `kanboard-mcp.exe` (Windows) / `kanboard-mcp` (Linux/macOS),
spawned by the MCP host as a stdio child process.

## Build & test loop

```bash
go build -o kanboard-mcp.exe .
```

After rebuilding you must **fully exit the MCP host** (Claude Code: `/exit`
then relaunch; Claude Desktop: quit from the tray, not just close the window)
so the child process is respawned with the new binary. `/mcp` reconnect alone
keeps the old process running.

Build scripts are provided for convenience:
- `build-release.sh` / `build-release.bat` — Linux/Windows
- `build-darwin.sh` — macOS, with auto Homebrew Go install

## Looking up Kanboard API documentation

Use the **context7** MCP server to fetch up-to-date Kanboard JSON-RPC docs
before adding or fixing a handler — endpoint names, parameter shapes, and
return types should be confirmed against the live docs rather than guessed.

Typical flow:
1. `mcp__claude_ai_Context7__resolve-library-id` with `kanboard` to get the
   library id
2. `mcp__claude_ai_Context7__query-docs` with that id and a topic
   (e.g. `closeTask`, `createTaskLink`, `getMyProjects`) to pull the relevant
   reference excerpt
3. Mirror the documented parameter names exactly in the handler's `params`
   map — Kanboard rejects unknown / misnamed fields with `code -32602
   Invalid params`.

## Patterns to keep consistent

1. **Tool schema and handler must agree on types.**
   `mcp.WithNumber("foo")` in the schema must be paired with
   `request.RequireInt("foo")` (or `GetInt`) in the handler. The
   `RequireString` + `strconv.Atoi` pattern fails at runtime because the SDK
   does not coerce numeric JSON to strings.

2. **Use Kanboard's dedicated endpoints for status changes.**
   `closeTask`, `openTask`, `moveTaskPosition`, etc. — do not overload
   `updateTask` with synthetic `status` strings.

3. **`app-admin` short-circuits project-scoped permission checks.**
   See `RBACManager.CheckPermission` in `main.go`.

## Auth modes

Selected via `KANBOARD_AUTH_METHOD`:

| Mode | Auth header | When to use |
|---|---|---|
| `global_token` (default) | `Basic jsonrpc:<token>` | Server-to-server using the global API key from Settings → API. `getMe` will not return a real user. |
| `user_token` | `Basic <username>:<token>` | Per-user token from a user's profile API tab. Required if you want `getMe` and audit fields populated. |
| `bearer` | `Bearer <token>` | Reverse-proxy / OIDC scenarios. |

For a single-admin local setup, prefer `user_token` so RBAC, `getMe`, and
activity logs all work correctly.

## Useful debug envs

- `KANBOARD_DEBUG=true` — verbose stderr logs (auth path taken, RBAC
  decisions, raw API responses)
- `KANBOARD_SKIP_RBAC=true` — bypass MCP-side RBAC entirely. Kanboard's own
  permissions still apply server-side. Useful when the MCP RBAC layer is
  too strict for solo use.
- `KANBOARD_INSECURE_SKIP_VERIFY=true` — accept self-signed / mismatched TLS
  certificates. Only for trusted links (e.g. Kanboard reached over Tailscale).
- `KANBOARD_USER_APP_ROLES`, `KANBOARD_USER_PROJECT_ROLES` — manual role
  pinning fallbacks, only consulted after `getMe` succeeds.

## Git boundaries for Claude

Claude **may** create commits on the current branch when explicitly asked.

Claude **must not**:

- Push to any remote (`origin`, `upstream`, or otherwise) — pushing is the
  user's call once they have reviewed the commit
- Force-push, rebase published history, or amend commits that already exist
  on a remote
- Create or merge pull requests
- Add new git remotes or rewrite remote URLs

If a task seems to require any of the above, stop and ask.

## Commit style

Imperative subject under ~70 chars, followed by a body of dash-bulleted
sentences explaining what changed and why:

```
Fix close_task to call Kanboard closeTask endpoint

- Replace updateTask + status="closed" with the dedicated closeTask method
- Resolves "Invalid params" error when calling mcp__kanboard__close_task
- Mirrors the open/close pair documented in the Kanboard JSON-RPC reference
```

Each commit should be **atomic** — one bug fix, one independent change. Avoid
bundling unrelated cleanups into a fix commit.

## Suggested workflow for the bug punch list

1. Create a branch: `git checkout -b fix/handler-bugs`
2. Walk `BUGS.md` top-to-bottom, one commit per entry
3. Rebuild + manually exercise each fix via the MCP host before committing the next
4. Open a single PR for the branch (or split into separate PRs per commit if
   the upstream prefers smaller reviews — `git cherry-pick` makes that cheap)
