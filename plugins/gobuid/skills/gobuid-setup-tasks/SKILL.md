---
name: gobuid-setup-tasks
description: Batch-create tasks and sub-tasks in a GoBuid project and assign owners. Suited for setting up many work items at once, e.g. a construction schedule. Operates via MCP tools.
allowed-tools: mcp__gobuid-mcp-sse__*
---

## 使用時機

當用戶說「幫我建立任務」、「setup tasks」、「批次新增工作項目」時使用此 skill。

---

## 前置需求

### GoBuid MCP Server

此 skill 透過 GoBuid MCP Server（`gobuid-mcp-sse`）操作 API，執行前須確認：

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server
2. 已設定 context：呼叫 `mcp__gobuid-mcp-sse__set_context` 選擇 group

驗證：呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常。

---

## MCP Tool 參考

```
# 查看目前使用者（確認連線）
mcp__gobuid-mcp-sse__user_self

# 列出專案
mcp__gobuid-mcp-sse__project_get_self_projects

# 查看專案成員
mcp__gobuid-mcp-sse__user_query_user_photo_brief_by_project_id

# 建立單一 task
mcp__gobuid-mcp-sse__task_list_create_task_list

# 批次建立多個 task
mcp__gobuid-mcp-sse__task_list_create_multiple_task_lists

# 建立 sub task
mcp__gobuid-mcp-sse__task_list_create_task_list_sub_task

# 查看 task 詳情
mcp__gobuid-mcp-sse__task_list_get_task_list_detail

# 更新 task
mcp__gobuid-mcp-sse__task_list_update_task_list

# 刪除 task
mcp__gobuid-mcp-sse__task_list_delete_task_list
```

---

## 執行流程

### Step 1：確認 context

確認 groupId 與 projectId：

- 詢問用戶要在哪個 **project** 下建立任務
- 透過 MCP tool 查詢用戶的專案：
  呼叫 `mcp__gobuid-mcp-sse__project_get_self_projects`
- 記下 projectId，後續建立 task 時使用

### Step 2：收集主 task 清單

請用戶提供任務清單，格式可以是：

- 自然語言描述（「建立地基、搭鋼筋、灌漿」）
- 條列清單
- 貼上 Excel 複製的內容

每個 task 收集：

- 標題（必填）
- 負責人（**必填**，沒有指派負責人的 task 將無人看到）
- 開始 / 結束日期（選填）
- 備註（選填）

### Step 3：批次建立主 task

將任務清單組成結構化參數，透過 MCP tool 批次建立：

呼叫 `mcp__gobuid-mcp-sse__task_list_create_multiple_task_lists`，參數範例：

```json
{
  "projectId": "<id>",
  "taskLists": [
    {
      "name": "地基工程",
      "startDate": "2026-04-01T00:00:00",
      "endDate": "2026-04-15T00:00:00"
    },
    { "name": "鋼筋工程" }
  ]
}
```

或逐一建立（需要更細緻的控制時）：

呼叫 `mcp__gobuid-mcp-sse__task_list_create_task_list`，參數範例：

```json
{ "projectId": "<id>", "name": "地基工程" }
```

記錄回傳的 task ID。

### Step 4：詢問是否有 sub task

針對每個主 task，詢問：
「這個任務下是否有子任務？如果有，請提供清單。」

用戶可以選擇：

- 跳過（無 sub task）
- 提供 sub task 清單

### Step 5：批次建立 sub task

有 sub task 的話，透過 MCP tool 建立：

呼叫 `mcp__gobuid-mcp-sse__task_list_create_task_list_sub_task`，參數範例：

```json
{
  "taskListId": 123,
  "name": "測量放線"
}
```

### Step 6：指派負責人（必要步驟）

**負責人為必填**，沒有指派的 task 將無人看到，不可跳過。

- 如果 Step 2 已收集負責人資訊，直接指派
- 如果尚未提供，**強制詢問**每個 task 的負責人
- 透過 MCP tool 更新 task 指派：
  呼叫 `mcp__gobuid-mcp-sse__task_list_update_task_list`，參數範例：
  ```json
  { "id": 123, "assigneeId": 456 }
  ```

### Step 7：回報結果

最後以結構化的方式回報建立結果：

```
✅ 已建立 X 個主 task：
  - 地基工程（ID: 123）
    └ 測量放線（sub task）
    └ 開挖作業（sub task）
  - 鋼筋工程（ID: 124）
    └ 綁紮作業（sub task）

負責人已指派完成。
```

---

## 注意事項

- 每個 API 呼叫之間不要太急，確認上一步成功再繼續
- 如果某個 task 建立失敗，告知用戶並詢問是否重試
- 日期格式使用 ISO 8601（`2026-04-01T00:00:00`）
- 一律透過 MCP tool 操作，不使用 CLI
