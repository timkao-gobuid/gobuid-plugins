---
name: gobuid-create-project
description: Create a GoBuid project, ensuring the required fields (name, start/end date, address and real coordinates) are gathered and confirmed before creating. Uses geocoding place autocomplete to bind address and coordinates together. Operates via the gobuid-mcp-sse MCP server.
allowed-tools: mcp__gobuid-mcp-sse__project_create_project, mcp__gobuid-mcp-sse__geocoding_place_autocomplete, mcp__gobuid-mcp-sse__geocoding_address_lookup, mcp__gobuid-mcp-sse__user_self, mcp__gobuid-mcp-sse__set_context
---

## 使用時機

當用戶說「建立專案」、「新增 project」、「開一個新工地 / 工程」時使用此 skill。

---

## 前置條件

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server。
2. **已設定 group context**：`project_create_project` 沒有 `groupId` 參數，project 會建在 `set_context` 設定的 group 底下（透過 GroupId header）。
   - 未設定時，先用 `/gobuid-list-groups` 列出並 `set_context` 選定 group。

驗證：呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常。

---

## 核心概念

> **⚠️ 後端對 create-project 不檢查必填欄位。** 缺欄位也會「建立成功」但資料殘缺（例如座標 `0,0`）。**必填把關是這個 skill 的責任**，不是後端。

建立前必須湊齊並確認以下欄位：

| 欄位                     | 來源        | 說明                       |
| ------------------------ | ----------- | -------------------------- |
| `name`                   | 問用戶      | 專案名稱，必填             |
| `address`                | geocoding   | 地址字串                   |
| `latitude` / `longitude` | geocoding   | **真實座標，不可為 `0,0`** |
| `startDate`              | 預設 + 確認 | 起始日（ISO 8601）         |
| `endDate`                | 預設 + 確認 | 結束日（ISO 8601）         |
| `icon`                   | 固定預設    | `"trafficCone"`            |
| `color`                  | 固定預設    | `"#E9BF78"`                |
| `description`            | 選用        | 描述                       |

> 只送上述欄位即可（比照產品行為）。`city` / `postalCode` / `placeId` / `premise` / `landmarkName` 雖然 tool 支援，但**不送**。

### 地址與座標必須綁在一起

agent 沒有地圖 UI，使用者也不可能手報經緯度。**用 `geocoding_place_autocomplete` 把地址與座標一次綁定**：

- 傳入 `Address`（地址或地標名）→ 回傳**候選陣列**，每個候選已含座標：

  ```json
  {
    "address": "110台灣台北市信義區市府路45號",
    "city": "台北市",
    "latitude": 25.0375,
    "longitude": 121.5637,
    "name": "台北101",
    "placeId": "ChIJ...",
    "postalCode": "110",
    "premise": null
  }
  ```

- 讓使用者從候選**選一個**，即同時得到一致的 `address` + `latitude` + `longitude`，並能消除同名地點的歧義。

反向情境（使用者直接給經緯度）：用 `geocoding_address_lookup`（傳 `latitude` / `longitude`）反查地址，回傳 `{ address: { address, city, latitude, longitude, ... } }`。

---

## MCP Tool 參考

```
# 地址 → 候選清單（含座標）— 建立流程主力
mcp__gobuid-mcp-sse__geocoding_place_autocomplete   # 參數 Address

# 經緯度 → 地址（反查）
mcp__gobuid-mcp-sse__geocoding_address_lookup        # 參數 latitude, longitude

# 建立 project
mcp__gobuid-mcp-sse__project_create_project

# group context（前置）
mcp__gobuid-mcp-sse__set_context
mcp__gobuid-mcp-sse__user_self
```

---

## 執行流程

### Step 1：確認 group context

確認已 `set_context`。沒有就先用 `/gobuid-list-groups` 列出 groups、選定後 `set_context(groupId)`。

### Step 2：收集 name

詢問專案名稱。空白不放行。

### Step 3：取得 address + 座標

1. 請使用者提供地址或地標名稱。
2. 呼叫 `geocoding_place_autocomplete`，`Address` 帶使用者輸入。
3. 列出候選（顯示名稱 + 完整地址）給使用者選：

   ```
   找到這些地點，請選一個：
   1. 台北101（110台灣台北市信義區市府路45號）
   2. ...
   ```

4. 使用者選定後，取出該候選的 `address`、`latitude`、`longitude`。
5. 若使用者改為直接給經緯度，改用 `geocoding_address_lookup` 反查補 `address`。

> **座標為 `0,0` 或空 → 視為未取得，回到本步驟重新地理編碼，不可直接建立。**

### Step 4：收集 from / to（startDate / endDate）

- 預設 `startDate = 今天`、`endDate = 今天 + 1 個月`。
- 把預設值列給使用者確認，允許修改。
- 送出前轉成 ISO 8601（例如 `"2026-06-23T00:00:00Z"`）。
- **`startDate > endDate` 則擋下重問。**

### Step 5：建立前確認閘門

把組好的 payload 攤開，逐項確認都齊全才繼續：

```
即將建立 project，請確認：
  name      ✓ 台北捷運工程
  address   ✓ 110台灣台北市信義區市府路45號
  座標       ✓ 25.0375, 121.5637
  startDate ✓ 2026-06-23
  endDate   ✓ 2026-07-23
  icon/color  trafficCone / #E9BF78（預設）
  description -（無）

確認建立？
```

任何一項缺漏（尤其座標為 `0,0`），**不要呼叫 create**，回到對應步驟補齊。

### Step 6：呼叫 create

確認後呼叫 `project_create_project`，傳入：

```
name, address, latitude, longitude, startDate, endDate,
icon: "trafficCone", color: "#E9BF78", description
```

### Step 7：回報結果

```
✅ 已建立 project：「台北捷運工程」(projectId: 312)
   地址：110台灣台北市信義區市府路45號（25.0375, 121.5637）
   期間：2026-06-23 ～ 2026-07-23
```

取得 `projectId` 後可接續 `/gobuid-task-crud` 等需要 project 的操作。

---

## 注意事項

- **必填把關在 skill，不在後端**：缺 name / address / 真實座標 / from / to 一律不送。
- **座標一定要真實**：不可送 `0,0`（產品 UI 允許 0,0，但我們要比它嚴）。座標一律從 geocoding 取得，不要憑空填。
- 地址太籠統（如只給「台北」）時，座標會落在市中心，提醒使用者可給更精確的地址再 autocomplete。
- `groupId` 走 `set_context`（create tool 無此參數），建立前務必確認 group 正確。
- 只送建立所需欄位，地址細項（city/postalCode/placeId/premise/landmarkName）不送。
- 日期一律轉 ISO 8601 再送。
- API 呼叫失敗時，回報錯誤訊息並詢問是否重試，不要靜默吞掉。
- 一律透過 MCP tool 操作，不使用 CLI。
  </content>
