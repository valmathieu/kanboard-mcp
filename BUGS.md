# MCP handler bugs to fix

Each entry below is independent and meant to ship as its own atomic commit.
Bugs are ordered by suggested commit sequence.

---

## 1. `close_task` calls the wrong Kanboard endpoint

**File:** `main.go:7515-7530` (`closeTaskHandler`)

**Bug:** Sends `updateTask` with `task_id` and `status: "closed"`. Kanboard's
`updateTask` expects `id` (not `task_id`) and does not accept a string `status`
parameter, so the call fails with `code -32602 Invalid params`.

**Fix:** Call the dedicated `closeTask` JSON-RPC method, which accepts `task_id`
and is the proper Kanboard endpoint for this action.

```diff
- params := map[string]interface{}{"task_id": taskId, "status": "closed"}
- result, err := kc.callKanboardAPI(ctx, "updateTask", params)
+ params := map[string]interface{}{"task_id": taskId}
+ result, err := kc.callKanboardAPI(ctx, "closeTask", params)
```

**Reproduction:** `mcp__kanboard__close_task` on any task returns
`Failed to close task: kanboard API error (code -32602): Invalid params`.

**Status:** Patched in working tree (not yet committed).

**Suggested commit subject:** `Fix close_task to call Kanboard closeTask endpoint`

---

## 2. `open_task` has the same bug

**File:** `main.go:7498-7513` (`openTaskHandler`)

**Bug:** Same shape as #1 — calls `updateTask` with `status: "open"` instead of
the dedicated `openTask` method.

**Fix:**

```diff
- params := map[string]interface{}{"task_id": taskId, "status": "open"}
- result, err := kc.callKanboardAPI(ctx, "updateTask", params)
+ params := map[string]interface{}{"task_id": taskId}
+ result, err := kc.callKanboardAPI(ctx, "openTask", params)
```

**Suggested commit subject:** `Fix open_task to call Kanboard openTask endpoint`

---

## 3. Schema/handler type mismatch on `project_id` (4 handlers)

**Bug:** Four tool registrations declare `mcp.WithNumber("project_id")` in the
schema, but the handler reads `request.RequireString("project_id")`. The MCP
SDK rejects the call at runtime with `argument "project_id" is not a string`
because numeric JSON values aren't coerced.

| Tool | Schema line | Handler line |
|---|---|---|
| `create_column` | 1389 | 4387 |
| `reorder_columns` | 1466 | 4484 |
| `get_categories` | 1511 | 4521 |
| `create_category` | 1520 | 4545 |

**Fix:** Replace the `RequireString` + `Atoi` pair with `RequireInt`:

```diff
- projectIdStr, err := request.RequireString("project_id")
- if err != nil {
-     return mcp.NewToolResultError(err.Error()), nil
- }
- projectId, err := strconv.Atoi(projectIdStr)
- if err != nil {
-     return mcp.NewToolResultError(fmt.Sprintf("Invalid project_id: %v", err)), nil
- }
+ projectId, err := request.RequireInt("project_id")
+ if err != nil {
+     return mcp.NewToolResultError(err.Error()), nil
+ }
```

**Note:** `getColumnsHandler` (4344) and `assignUserToProjectHandler` (4904)
also use `RequireString("project_id")`, but their schemas declare `WithString`
(lines 1221, 1371) so they currently work. Migrating those to `WithNumber` +
`RequireInt` is a separate, optional consistency cleanup — not part of this
fix.

**Reproduction:** `mcp__kanboard__create_column` or
`mcp__kanboard__create_category` with `project_id: 6` returns
`argument "project_id" is not a string`.

**Suggested commit subject:** `Fix project_id type mismatch in 4 column/category handlers`

---

## 4. `app-admin` ignored for project-scoped permission checks

**File:** `main.go:495-534` (`RBACManager.CheckPermission`)

**Bug:** When `projectID != nil`, the function only consults
`userCtx.ProjectRoles[*projectID]`. A user with `app-admin` who has no explicit
project-member role gets `accessMap.DefaultRole` (typically empty or
`app-user`) and is denied. This contradicts Kanboard's own UI behavior, where
an `app-admin` can act on any project regardless of membership.

**Fix:** Short-circuit the project-scoped check when the user is `app-admin`:

```diff
  if projectID != nil {
      // Project-level permission check
      accessMap = rbac.config.AccessMaps.APIProject
+     if userCtx.IsAdmin {
+         // Application-level admin can act on any project
+         return true
+     }
      userRole = userCtx.ProjectRoles[*projectID]
      if userRole == "" {
          userRole = accessMap.DefaultRole
      }
  }
```

**Workaround:** Set `KANBOARD_SKIP_RBAC=true` in the MCP env (currently used in
the local Claude Code config to keep the demo unblocked).

**Reproduction:** With `user_token` auth as admin, calling
`mcp__kanboard__create_task` on a project where admin has no explicit project
role yields:

```
access denied: insufficient permissions for taskprocedure.createtask
(user: admin, app roles: [app-admin] (no project role assigned))
```

even though the user is `app-admin`.

**Suggested commit subject:** `Honor app-admin in project-scoped permission checks`

---

## Commit / branch plan

- Single feature branch `fix/handler-bugs` with one atomic commit per bug
- Each commit touches an independent code path → safe to revert/cherry-pick individually
- After `go build -o kanboard-mcp.exe .`, fully exit the MCP host (Claude
  Code / Claude Desktop) so the child process is respawned with the new binary
- Once #4 lands, `KANBOARD_SKIP_RBAC=true` can be removed from the MCP env
