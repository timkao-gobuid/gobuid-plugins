---
name: gobuid-task-crud
description: Create, read, update and delete GoBuid tasks and subtasks via the v2 MCP tools. Handles listing/filtering, kanban board, assignees, due dates, repeat rules, status changes, subtasks and reordering. Operates entirely through the gobuid-mcp-sse MCP server.
allowed-tools: mcp__gobuid-mcp-sse__*
---

## 使用時機

當用戶說「建立任務」、「新增 task / subtask」、「指派任務給 XXX」、「列出某專案的任務」、「把任務標成完成 / 進行中」、「刪除任務」、「調整任務順序」時使用此 skill。

---

## 前置條件

此 skill 透過 GoBuid MCP Server（`gobuid-mcp-sse`）操作 API，執行前須確認：

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server
2. 已設定 context：呼叫 `mcp__gobuid-mcp-sse__set_context` 選擇 group

驗證：呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常。

> **⚠️ Server 版本需求：** create / update 走 multipart form-data，需要 server 端已包含 `appendForm` 修正（陣列改用 repeated key、物件陣列改用 `key[i].prop`）。**舊版 server 會把 `assignUserIds`、`linkItems`、`subTasks` 等欄位序列化壞掉**（指派人綁不進去、子任務/關聯失效）。若指派人沒生效，先確認 mcp.gobuid.com 已部署含此修正的版本。

---

## 核心概念

### Task 與 Subtask 階層

```
Task（task_list，隸屬於一個 project）
 └── Subtask（task_list_sub_task，隸屬於一個 task）
```

- Task 綁在某個 `ProjectId` 底下；Subtask 綁在某個 `TaskListId`（task id）底下。
- v1（`task_list_*`）與 v2（`task_list_v2_*`）操作同一份 entity（同一組 id）。**本 skill 一律使用 v2 RESTful tools**（id 走 path、delete 正常、列表/看板/filter 完整）。

> **⚠️ tool 名稱有重複前綴**，這是 codegen 產生的，照抄即可：
> 例如建立 task 的 tool 是 `mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_post`。

### Status（狀態）

| 對象    | 可用值                                                |
| ------- | ----------------------------------------------------- |
| Task    | `Open` / `InProgress` / `OnHold` / `Done` / `Overdue` |
| Subtask | `Open` / `InProgress` / `OnHold` / `Done`             |

> **`Overdue` 是衍生狀態，不可手動指定。** 它由「到期時間已過且未完成」自動推導，回傳時透過 `isOverdue` 旗標呈現。建立/更新時 `Status` 只送 `Open` / `InProgress` / `OnHold` / `Done`。

### 日期

- `From` / `To` 為 ISO 8601 date-time 字串，例如 `"2026-06-25T09:00:00Z"`。
- `IsAllDay: true` 代表全天任務（時間部分忽略）。

### 指派 / 通知

- `AssignUserIds`、`NotifyUserIds` 都是 **userId 陣列（number[]）**，不是名字、不是 email。
- 用戶給的是人名時，先用 member 查詢 tool 解析成 userId（見「解析 ProjectId 與 userId」）。

### Repeat（重複規則）

建立/更新 task 時，repeat 以**扁平參數**傳入（注意參數名用底線）：

| 參數                      | 說明                                                                        |
| ------------------------- | --------------------------------------------------------------------------- |
| `Repeat_Mode`             | `Daily` / `Weekdays` / `EveryMonth` / `Yearly` / `Custom`（含 `Friday` 等） |
| `Repeat_Interval`         | 間隔數（number），搭配 `Custom`                                             |
| `Repeat_Unit`             | `Day` / `Week` / `Month` / `Year`                                           |
| `Repeat_Ends_Type`        | `Never` / `On` / `After`                                                    |
| `Repeat_Ends_Date`        | `Ends_Type=On` 時的結束日期（ISO date-time）                                |
| `Repeat_Ends_Occurrences` | `Ends_Type=After` 時的次數（number）                                        |

不需要重複就整組省略。

### LinkItems（關聯其他模組，選用）

`LinkItems` 是物件陣列，把 task 關聯到其他模組項目：

```json
[{ "type": "Submission", "id": 123 }]
```

`type` 可為 `Task` / `Subtask` / `Activity` / `Submission`。

---

## MCP Tool 參考

### Task

```
# 列表（含 filter / keyword / 分頁）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks
# 看板（依 status 分欄）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_board
# 單筆詳情（含 subtask 統計、assignee、linkItems）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_by_id
# 下拉選單（依 ProjectId 搜尋 task 名稱）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_dropdown

# 建立
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_post
# 更新（全欄位）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_by_id_put
# 只改狀態（輕量 PATCH）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_by_id_status
# 刪除
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_by_id_delete

# 排序 / 看板拖移
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_reorder
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_kanban_move
```

### Subtask

```
# 列出某 task 的所有 subtask
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_tasks_by_id_subtasks
# 單筆詳情
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_subtasks_by_id

# 建立
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_subtasks
# 更新（全欄位）
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_subtasks_by_id_put
# 只改狀態
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_subtasks_by_id_status
# 刪除
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_subtasks_by_id_delete

# 排序
mcp__gobuid-mcp-sse__task_list_v2_task_list_v2_subtasks_reorder
```

### 輔助查詢

```
# 列出可用 projects（取 ProjectId）
mcp__gobuid-mcp-sse__project_get_self_projects
# 解析人名 → userId（Keyword 帶名字）
mcp__gobuid-mcp-sse__member_query_member_projects_by_page_filter
```

---

## 欄位 Schema 速查

### 建立 Task — `..._tasks_post`

| 參數                                  | 必填 | 說明                           |
| ------------------------------------- | ---- | ------------------------------ |
| `ProjectId`                           | 是   | 任務所屬專案                   |
| `Title`                               | 是   | 任務標題                       |
| `Notes`                               | 否   | 描述                           |
| `Status`                              | 否   | 預設 `Open`                    |
| `From` / `To`                         | 否   | 起訖時間（ISO date-time）      |
| `IsAllDay`                            | 否   | 全天任務                       |
| `Location` / `Latitude` / `Longitude` | 否   | 地點                           |
| `AssignUserIds`                       | 否   | 指派人 userId[]                |
| `NotifyUserIds`                       | 否   | 通知對象 userId[]              |
| `FileIds`                             | 否   | 附件 file id[]（需先上傳）     |
| `LinkItems`                           | 否   | 關聯項目 `[{type,id}]`         |
| `Repeat_*`                            | 否   | 重複規則（見核心概念）         |
| `SubTasks`                            | 否   | 建立時一併帶入的子任務（見下） |

**建立時內嵌子任務 `SubTasks`**（物件陣列，key 用 camelCase）：

```json
[
  {
    "title": "現場拍照",
    "note": "拍 3 個角度",
    "status": "Open",
    "from": "2026-06-25T09:00:00Z",
    "to": "2026-06-25T12:00:00Z",
    "isAllDay": false,
    "assignUserIds": [42],
    "linkItems": [],
    "fileIds": []
  }
]
```

> 若不確定子任務細節，**建議「先建 task，再逐筆建 subtask」**（用 `..._subtasks` tool），流程較直觀也好除錯。

### 更新 Task — `..._tasks_by_id_put`

- `taskListId`（path，必填）+ 與建立相同的欄位。
- **這是整筆覆寫**：未帶的欄位可能被清空，更新前先用 `..._tasks_by_id` 取得現值，合併後再送。
- 不含 `SubTasks`（子任務透過各自的 tool 管理）。

### 只改狀態 — `..._tasks_by_id_status`

```
taskListId: <task id>
status: "Done"
```

> 單純要把任務標完成/進行中時用這個，不要動整筆 PUT。

### 建立 Subtask — `..._subtasks`

| 參數                                  | 必填 | 說明                                             |
| ------------------------------------- | ---- | ------------------------------------------------ |
| `TaskListId`                          | 是   | 所屬 task id                                     |
| `Title`                               | 是   | 子任務標題                                       |
| `Note`                                | 否   | 描述（注意：task 用 `Notes`，subtask 用 `Note`） |
| `Status`                              | 否   | 預設 `Open`（subtask 無 `Overdue`）              |
| `From` / `To` / `IsAllDay`            | 否   | 時間                                             |
| `Location` / `Latitude` / `Longitude` | 否   | 地點                                             |
| `AssignUserIds`                       | 否   | 指派人 userId[]（subtask 無 `NotifyUserIds`）    |
| `FileIds` / `LinkItems` / `Repeat_*`  | 否   | 同 task                                          |

### 更新 / 改狀態 / 刪除 Subtask

- 更新：`..._subtasks_by_id_put`，`taskListSubTaskId`（path）+ 欄位（整筆覆寫，同 task 規則）。
- 改狀態：`..._subtasks_by_id_status`，`taskListSubTaskId` + `status`。
- 刪除：`..._subtasks_by_id_delete`，`taskListSubTaskId`。

---

## 執行流程

### 解析 ProjectId 與 userId

動手前先把人名/專案名轉成 id：

1. **ProjectId**：呼叫 `project_get_self_projects`，依名稱比對取得 id。用戶只有一個專案或已指定時可略過。
2. **userId**：用戶用人名指派時，呼叫 `member_query_member_projects_by_page_filter`（`Keyword` 帶名字）解析成 userId。**找不到或同名多人時，列出候選請用戶確認，不要猜。**

### 查詢 / 列表

- **列出某專案任務**：`..._tasks`，傳 `ProjectIds`（省略則涵蓋當前 group 全部專案）。可加 `Statuses`、`AssigneeIds`、`Keyword`(標題模糊查)、`Due`(`Overdue`/`Today`/`ThisWeek`/`DateRange`/`AllTime`)、`From`/`To`、分頁。
- **看板視圖**：`..._tasks_board`，回傳依 status 分欄的結果（`Overdue` 不是欄位，改看每筆的 `isOverdue`）。
- **單筆 + 子任務**：`..._tasks_by_id` 取詳情；`..._tasks_by_id_subtasks` 取該 task 的子任務清單。

### 建立 Task

1. 解析 `ProjectId` 與 `AssignUserIds` / `NotifyUserIds`。
2. 整理欄位（Title 必填；用戶有提到期限/重複/指派才帶對應欄位）。
3. 呼叫 `..._tasks_post`。
4. 需要子任務時，用回傳的 task id 逐筆呼叫 `..._subtasks`（或建立時帶 `SubTasks`）。

### 建立 Subtask

1. 確認 `TaskListId`（所屬 task）。
2. 解析 `AssignUserIds`。
3. 呼叫 `..._subtasks`。

### 更新

1. 先用 `..._tasks_by_id`（或 `..._subtasks_by_id`）取得現值。
2. 在現值上修改要變的欄位，**保留其他欄位**一起送 `..._tasks_by_id_put` / `..._subtasks_by_id_put`（整筆覆寫）。
3. 若只是改狀態，改用 `..._tasks_by_id_status` / `..._subtasks_by_id_status`。

### 刪除

- 直接呼叫 `..._tasks_by_id_delete` / `..._subtasks_by_id_delete`，id 走 path。
- **刪除前先向用戶確認**（顯示 task/subtask 標題），刪 task 會連帶其底下 subtask。

### 排序 / 看板拖移

- Task 重排：`..._tasks_reorder`，傳 `orderedTaskListIds`（完整的新順序 id 陣列）。
- 看板移動（換欄 + 換位）：`..._tasks_kanban_move`，傳 `taskListId` + 目標 `status` + 該欄的 `orderedTaskListIds`。
- Subtask 重排：`..._subtasks_reorder`，傳 `taskListId` + `orderedTaskListSubTaskIds`。

---

## 回報結果

操作完成後，用簡潔摘要回報，例如：

```
✅ 已在「台北捷運工程」建立任務：「鋼筋查驗」(ID: 1203)
   狀態：Open　到期：2026-06-30
   指派：王小明、李大華
   子任務 2 筆：現場拍照 / 上傳查驗表
```

---

## 注意事項

- 一律透過 v2 MCP tools 操作，不使用 CLI、不用 v1（v1 的 delete 在目前 codegen 下不會帶 body，刪不掉）。
- **`Status` 不要送 `Overdue`**（衍生狀態）；要表達逾期改看 `isOverdue`。
- **PUT 是整筆覆寫**：更新前務必先讀現值再合併，否則沒帶到的欄位會被清掉。
- `AssignUserIds` / `NotifyUserIds` / `FileIds` 是 number[]；人名/檔名先解析成 id。
- task 用 `Notes`（複數）、subtask 用 `Note`（單數），別寫錯。
- `SubTasks` / `LinkItems` 物件陣列的 key 用 camelCase（`assignUserIds`、`linkItems`…），且需 server 已含 multipart 修正才會生效。
- 刪除任務前先跟用戶確認，刪 task 會一併刪掉其 subtask。
- API 呼叫失敗時，回報錯誤訊息並詢問是否重試，不要靜默吞掉。
  </content>
  </invoke>
