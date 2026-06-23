---
name: gobuid-create-activity
description: Create a GoBuid activity (a work item tracked by quantity and unit) under a project, ensuring the required fields (name, total quantity, unit, start/end date) are gathered and confirmed before creating. Handles unit selection and the percent-unit rule. Operates via the gobuid-mcp-sse MCP server.
allowed-tools: mcp__gobuid-mcp-sse__activity_v2_activity_v2_activities, mcp__gobuid-mcp-sse__activity_query_activity_units_by_group_id, mcp__gobuid-mcp-sse__activity_create_activity_unit, mcp__gobuid-mcp-sse__geocoding_place_autocomplete, mcp__gobuid-mcp-sse__geocoding_address_lookup, mcp__gobuid-mcp-sse__teams_get, mcp__gobuid-mcp-sse__user_self, mcp__gobuid-mcp-sse__set_context
---

## 使用時機

當用戶說「建立 activity」、「新增工項 / 工作項目」、「開一個施工活動」時使用此 skill。

---

## 前置條件

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server。
2. **已設定 group context**（`set_context`）。activity 的單位、team 都是 per-group；沒設先用 `/gobuid-list-groups`。
3. **已知 `ProjectId`**。activity 隸屬於 project；沒有就先用 `/gobuid-list-projects` 取得。

驗證：呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常。

---

## 核心概念

Activity 是 project 底下、以**數量（TotalQty）+ 單位（ActivityUnitId）**追蹤進度的工作項目（例如「外牆安裝　1200　m²」）。

> **⚠️ 後端對 create-activity 不檢查必填欄位。** 必填把關是這個 skill 的責任。建立前必須湊齊並確認：

| 欄位                                  | 必填 | 來源           | 說明                            |
| ------------------------------------- | ---- | -------------- | ------------------------------- |
| `ProjectId`                           | 是   | context / 詢問 | 所屬 project                    |
| `Name`                                | 是   | 問用戶         | 工項名稱                        |
| `TotalQty`                            | 是   | 問用戶         | 總數量，**不以 0 開頭的正整數** |
| `ActivityUnitId`                      | 是   | 列單位讓用戶選 | 計量單位                        |
| `StartDate`                           | 是   | 預設 + 確認    | 起始日（ISO 8601）              |
| `EndDate`                             | 是   | 預設 + 確認    | 結束日（ISO 8601）              |
| `Location` / `Latitude` / `Longitude` | 否   | geocoding      | **選用**，使用者有提地點才帶    |
| `TeamIds`                             | 否   | `teams_get`    | team tag（number[]）            |
| `Notes`                               | 否   | 問用戶         | 備註                            |

> **與 create-project 不同：activity 的「地點」是選用的**，不強制。其餘必填欄位（name / 數量 / 單位 / 起訖日）一定要齊。

### ⚠️ 百分比（%）單位的特殊規則

當使用者選的單位是 **`%`**（名稱為 `%`，產品中其 `activityUnitId` 為 `15`）時：

- **`TotalQty` 必須固定為 `100`**（代表以百分比追蹤進度，滿值 100%）。
- 使用者給其他數字時，改用 100 並告知原因。

### 日期規則

- 預設 `StartDate = 今天`、`EndDate = 今天 + 1 個月`，列出讓使用者確認或改。
- **`StartDate > EndDate` 時自動對調**（保持 start ≤ end），對調後告知使用者。
- 送出前轉 ISO 8601（例如 `"2026-06-23T00:00:00Z"`）。

---

## MCP Tool 參考

```
# 建立 activity（v2，multipart）
mcp__gobuid-mcp-sse__activity_v2_activity_v2_activities

# 列出 group 的單位（取 activityUnitId）
mcp__gobuid-mcp-sse__activity_query_activity_units_by_group_id   # 參數 GroupId
# 新建單位（清單沒有用戶要的單位時）
mcp__gobuid-mcp-sse__activity_create_activity_unit              # name, groupId

# 地點（選用）：地址→候選含座標 / 經緯度→反查地址
mcp__gobuid-mcp-sse__geocoding_place_autocomplete               # Address
mcp__gobuid-mcp-sse__geocoding_address_lookup                   # latitude, longitude

# team tag（選用）
mcp__gobuid-mcp-sse__teams_get

# 前置
mcp__gobuid-mcp-sse__set_context
mcp__gobuid-mcp-sse__user_self
```

> create 走 v2 multipart；`TeamIds`、`LinkItems` 等陣列欄位需 server 已含 `appendForm` 修正才會正確序列化（同 `/gobuid-task-crud` 的注意事項）。

---

## 執行流程

### Step 1：確認 group context 與 ProjectId

- 確認已 `set_context`；沒有就 `/gobuid-list-groups`。
- 確認 `ProjectId`；沒有就 `/gobuid-list-projects`。

### Step 2：收集 Name

詢問工項名稱。空白不放行。

### Step 3：選單位（ActivityUnitId）

1. 呼叫 `activity_query_activity_units_by_group_id`（`GroupId` 帶當前 group），列出單位給使用者選：

   ```
   可用單位：
   - m²（id: 3）
   - m³（id: 4）
   - %（id: 15）
   - ...
   ```

2. 使用者選定 → 取 `activityUnitId`。
3. 清單沒有想要的單位時，問是否新建 → `activity_create_activity_unit`（`name` + `groupId`），再用回傳的 id。

### Step 4：收集 TotalQty

- 要求不以 0 開頭的正整數。
- **若 Step 3 選的是 `%` 單位 → 直接設為 `100`**，不另外問（或覆蓋使用者輸入並說明）。

### Step 5：日期 StartDate / EndDate

- 預設今天 ～ +1 個月，列出確認/修改。
- `start > end` 自動對調並告知；轉 ISO 8601。

### Step 6（選用）：地點

使用者有提到地點才做：

- 給地址/地標 → `geocoding_place_autocomplete(Address=…)` → 列候選（含座標）→ 選一個 → 取 `Location`(地址字串) + `Latitude` + `Longitude`。
- 給經緯度 → `geocoding_address_lookup` 反查補地址。
- 沒提地點就整組省略。

### Step 7：建立前確認閘門

攤開 payload 逐項確認必填齊全才繼續：

```
即將建立 activity，請確認：
  Project    ✓ 台北捷運工程 (301)
  Name       ✓ 外牆安裝
  TotalQty   ✓ 1200
  單位        ✓ m²（id: 3）
  StartDate  ✓ 2026-06-23
  EndDate    ✓ 2026-07-23
  地點        - （未指定，選用）
  Notes      - （無）

確認建立？
```

任一必填缺漏，**不要呼叫 create**，回對應步驟補齊。

### Step 8：呼叫 create

確認後呼叫 `activity_v2_activity_v2_activities`，傳入：

```
ProjectId, Name, TotalQty, ActivityUnitId, StartDate, EndDate
（選用）Location, Latitude, Longitude, TeamIds, Notes
```

### Step 9：回報結果

```
✅ 已建立 activity：「外牆安裝」(activityId: 882)
   進度單位：1200 m²　期間：2026-06-23 ～ 2026-07-23
   專案：台北捷運工程
```

---

## 注意事項

- **必填把關在 skill，不在後端**：缺 Name / TotalQty / ActivityUnitId / StartDate / EndDate 一律不送。
- **`%` 單位 → TotalQty 固定 100**（產品中該單位 id 為 15，但以名稱 `%` 判斷較穩）。
- `TotalQty` 是正整數且不以 0 開頭；送出前轉 number。
- `StartDate > EndDate` 自動對調，不要直接送出反向區間。
- 地點是**選用**（與 project 不同）；要帶就一律從 geocoding 取得真實座標，不要憑空填 `0,0`。
- 單位 / team 都是 per-group，操作前確認 group context 正確。
- `TeamIds` / `LinkItems` 等陣列欄位依賴 server 的 multipart 修正才會生效。
- API 呼叫失敗時，回報錯誤訊息並詢問是否重試，不要靜默吞掉。
- 一律透過 MCP tool 操作，不使用 CLI。
  </content>
