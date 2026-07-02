---
name: gobuid-list-projects
description: List the projects under a GoBuid group. Use when the user asks which projects a group has, needs a projectId before other GoBuid work (tasks, forms, submissions), or wants to find a project by name. Reads from the gobuid-mcp-sse MCP server.
allowed-tools: mcp__gobuid-mcp-sse__project_get_self_projects, mcp__gobuid-mcp-sse__project_query_self_project_by_filter, mcp__gobuid-mcp-sse__user_self, mcp__gobuid-mcp-sse__set_context
---

## 使用時機

當用戶說「列出專案」、「這個 group 有哪些 project」、「列出某 group 底下的 projects」、「找某某專案的 ID」時使用此 skill。

其他 skill（如 `/gobuid-task-crud`）需要 `projectId` 時，也會用到此 skill 取得。

---

## 前置條件

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server。
2. 知道目標 `groupId`（見下方「確定 groupId」）。

---

## 核心概念

- Project 隸屬於一個 group。列 projects 一定要先有 `groupId`。
- 回傳的是**目前使用者在該 group 底下參與的 projects**（self projects），不是 group 內全部 projects。
- `projectId` 是 task / form / submission 等模組操作的前提。

---

## 執行流程

### Step 1：確定 groupId

依序判斷 `groupId` 來源：

1. 用戶已明講 groupId → 直接用。
2. 已透過 `set_context` 設定過 → 沿用該 group。
3. 都沒有 → 呼叫 `mcp__gobuid-mcp-sse__user_self`，從 `groups` 取得；多個 group 時列出請用戶選（或改用 `/gobuid-list-groups`）。

### Step 2：列出 projects

呼叫：

```
mcp__gobuid-mcp-sse__project_get_self_projects  →  groupId: <groupId>
```

需要依名稱關鍵字 / 是否封存篩選時，改用：

```
mcp__gobuid-mcp-sse__project_query_self_project_by_filter
  GroupId: <groupId>
  Keyword: <名稱關鍵字>        # 選用
  IsArchived: false            # 選用
```

### Step 3：列出給用戶

```
group「GoBuild 總部」底下的 projects：
- 「台北捷運工程」(projectId: 301)　角色: PM　📌已釘選
- 「示範工地」(projectId: 305)
```

每筆主要欄位：`projectId`、`projectName`、`roleId`（角色）、`isPined`（是否釘選）、`teamIds`。

---

## 注意事項

- `project_get_self_projects` 的 `groupId` 帶 query 參數即可；它回傳當前使用者在該 group 的 projects。
- 用戶以名稱指定 project 時，比對 `projectName` 取得 `projectId`；同名或找不到時列出候選請用戶確認，不要猜。
- 找不到任何 project，先確認 `groupId` 是否正確、是否真的有參與該 group 的專案。
- 取得 `projectId` 後可接續 `/gobuid-task-crud` 等需要 project 的操作。
- 一律透過 MCP tool 操作，不使用 CLI。
