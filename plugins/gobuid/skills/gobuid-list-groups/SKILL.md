---
name: gobuid-list-groups
description: List the GoBuid groups (workspaces) the logged-in user belongs to, and optionally switch the active group context. Use when the user asks which groups they have, wants to switch group, or needs a groupId before doing other GoBuid work. Reads from the gobuid-mcp-sse MCP server.
allowed-tools: mcp__gobuid-mcp-sse__user_self, mcp__gobuid-mcp-sse__set_context
---

## 使用時機

當用戶說「列出我的 groups」、「我在哪些群組」、「我有哪些 workspace」、「切換 group」、「設定 group context」時使用此 skill。

其他 skill（如 `/gobuid-task-crud`、`/gobuid-list-projects`）需要先有 group context 時，也會用到此 skill 取得 groupId。

---

## 前置條件

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server。
2. 已完成 OAuth 登入（第一次呼叫 tool 時會自動開瀏覽器登入）。

> **不需要先設定 group context** —— 列出 groups 本身就是為了取得 groupId。

---

## 核心概念

> **⚠️ GoBuid 沒有獨立的「列出我的 groups」API。** 使用者所屬的 groups 是包在 `user_self` 的回傳裡（`groups` 陣列）。要列 groups 一律呼叫 `mcp__gobuid-mcp-sse__user_self`，再從中取出 `groups`。

groupId 是後續幾乎所有 GoBuid API 呼叫的前提：透過 `mcp__gobuid-mcp-sse__set_context` 設定後，server 會在每個 request 自動帶上 `GroupId` header。

### `groups[]` 欄位（`UserSelfGroupViewModel`）

| 欄位                      | 說明                                 |
| ------------------------- | ------------------------------------ |
| `groupId`                 | 群組 ID（set_context 與後續 API 用） |
| `groupName`               | 群組名稱                             |
| `roleId`                  | 使用者在此群組的角色                 |
| `seatTypeName`            | 席位類型                             |
| `unreadNotificationCount` | 未讀通知數                           |
| `subscriptionExpiredDate` | 訂閱到期日                           |
| `favoriteAt`              | 加入最愛的時間（有值代表是最愛群組） |

---

## 執行流程

### Step 1：取得 groups

呼叫 `mcp__gobuid-mcp-sse__user_self`，從回傳中取出 `groups` 陣列。

### Step 2：列出給用戶

```
你所屬的 groups：
- 「GoBuild 總部」(groupId: 12)　角色: Admin　未讀: 3
- 「示範專案群組」(groupId: 45)　角色: Member
```

只有一個 group 時，直接告知並可逕行 Step 3。

### Step 3：設定 group context（選用）

若用戶要切換/設定 group，或接下來要做需要 group context 的操作，呼叫：

```
mcp__gobuid-mcp-sse__set_context  →  groupId: <選定的 groupId>
```

設定後回報：「已切換到 group「GoBuild 總部」(groupId: 12)」。

---

## 注意事項

- 列 groups 一律走 `user_self`，**不要**去找 `Group/QueryGroupByFilter`（那是後台/管理用的全量篩選，不是「我的 groups」）。
- 用戶以名稱指定 group 時，從 `groups` 比對 `groupName` 取得 `groupId`；同名或找不到時列出候選請用戶確認，不要猜。
- 設定 context 後，可接續 `/gobuid-list-projects` 列出該 group 底下的 projects。
- 一律透過 MCP tool 操作，不使用 CLI。
  </content>
