---
name: gobuid-form-workflow
description: GoBuid 表單的 workflow 設定流程。建構 React Flow 節點圖（step/button/close 等 nodes + edges），設定 team tag，並確認相關成員已加入對應的 team tag。透過 MCP tool 操作。
allowed-tools: mcp__gobuid-mcp-sse__*, Bash(uuidgen *), Bash(for *)
---

## 使用時機

當用戶說「設定表單流程」、「設定 workflow」、「幫我設定審核步驟」時使用此 skill。

---

## 前置條件

### GoBuid MCP Server

此 skill 透過 GoBuid MCP Server（`gobuid-mcp-sse`）操作 API，執行前須確認：

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server
2. 已設定 context：呼叫 `mcp__gobuid-mcp-sse__set_context` 選擇 group

驗證：呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常。

### 其他前置

- form template ID：
  - 如果是從 `/gobuid-form-template` skill 串接過來，templateId 已自動帶入，**不需再詢問**
  - 如果是獨立執行，詢問用戶要設定哪個表單的 workflow（名稱或 ID）

---

## 核心概念

Workflow 是一個 **React Flow 節點圖**，由 nodes 和 edges 組成：

```
[Step] → [Button] → [Step] → [Button] → [Close]
```

- **Node 類型**：step、button、close、sendEmail、backToPrevious、createNewVersion
- **Edge**：連接 nodes 的有向邊，定義流程走向
- **API 是全量替換**：每次都送整個 graph（nodes + edges + validation），不支援單獨新增/刪除 node

### Page-Step 綁定機制（重要）

> **⚠️ 前端不會讀取 workflow node data 中的 `pageIds` 來顯示 sections。**
>
> Sections 的顯示是由 **form body 中每個 component 的 `setting.steps`** 決定的。
> 前端 CardStep 元件會掃描 `pagesData`，找出每個 component 的 `setting.steps` 陣列中包含該 step node ID 的 page，才會顯示在該 step 下。
>
> 因此，設定 workflow 時必須執行 **兩個步驟**：
>
> 1. 更新 workflow graph（nodes + edges）— 透過 `mcp__gobuid-mcp-sse__form_template_setting_update_form_workflow`
> 2. 更新 form body，在相關 page 的 components 中加入 `setting.steps` — 透過 `mcp__gobuid-mcp-sse__form_template_setting_update_form_body`
>
> 只做第 1 步而不做第 2 步，UI 上的 step 會顯示「Add Sections」但沒有任何 page。

`setting.steps` 格式：

```json
{
  "setting": {
    "label": "欄位名稱",
    "format": "text",
    "localAlignment": "top",
    "steps": [{ "id": "<step-node-uuid>", "name": "<step-name>" }]
  }
}
```

- 每個 field component 的 `setting.steps` 陣列存放該欄位所屬的 step 資訊
- 一個 page 下所有 components 都需要加上相同的 steps 綁定
- 一個 component 可以屬於多個 steps（例如 step 1 和 step 2 都能看到該欄位）

---

## MCP Tool 參考

```
# 取得 workflow
mcp__gobuid-mcp-sse__form_template_setting_get_form_workflow_by_id

# 更新 workflow
mcp__gobuid-mcp-sse__form_template_setting_update_form_workflow

# 列出 team tags
mcp__gobuid-mcp-sse__teams_get
mcp__gobuid-mcp-sse__teams_get_teams_by_page_filter

# 指派成員到 team tag
mcp__gobuid-mcp-sse__teams_assign_to_user_project

# 批次指派
mcp__gobuid-mcp-sse__teams_assign_to_multiple_users_and_projects

# 取得表單 body（pages）
mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id

# 更新表單 body
mcp__gobuid-mcp-sse__form_template_setting_update_form_body
```

---

## 執行流程

### Step 1：並行取得 Form Body + Team Tags

同時呼叫兩個 MCP tool：

- `mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id`（傳入 templateId）
- `mcp__gobuid-mcp-sse__teams_get`

從 form body 中提取 pages 陣列（每個 page 有 `id` 和 `name`）。
整理後展示給用戶：

```
此表單有以下 Pages：
- "基本資料" (ID: abc-123)
- "檢查項目" (ID: def-456)

此 project 可用的 Team Tags：
- 土木工程組（ID: 1）
- 品管組（ID: 2）
```

> Pages 的 ID 後續用於：
>
> 1. step node 的 `pageIds` 欄位（API 儲存參考）
> 2. form body component 的 `setting.steps` 綁定（**前端實際用來顯示 sections 的機制**）

---

### Step 2：收集 Workflow 結構

提示用戶描述 workflow 結構，提供常見模板作為參考：

```
最常見的 workflow 結構：
  Step 1 → Submit → Step 2 → Submit → Close

請描述你的 workflow：
- 每個 step 的名稱、負責的 team tag、包含的 pages
- 每個 step 上的按鈕（預設為一個 "Submit" 按鈕）
- 每個按鈕的目標（下一個 step / 結案 / 寄信 / 退回 / 建立新版本）
```

每個 **step** 需收集：

| 欄位      | 必填 | 說明                                             |
| --------- | ---- | ------------------------------------------------ |
| Step 名稱 | 是   | 顯示名稱                                         |
| Team tag  | 是   | 從 Step 1 清單選擇，可多選 → `teamIds: number[]` |
| Pages     | 是   | 從 Step 1 的 pages 選擇 → `pageIds: string[]`    |

> **⚠️ 每個 step 必須指定至少一個 page。** 分配 pages 後，除了在 workflow node data 中設定 `pageIds`，還必須更新 form body 的 `setting.steps`（見 Step 4b），否則 UI 上 "Add Sections" 會是空的。
> 建立 workflow 前，先用 `mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id` 取得所有 page 的 ID 和名稱，再逐一分配給各 step。

每個 **button** 需收集：

| 欄位        | 必填 | 說明                                                              |
| ----------- | ---- | ----------------------------------------------------------------- |
| Button 名稱 | 否   | 預設 "Submit"                                                     |
| 目標        | 是   | next step / close / sendEmail / backToPrevious / createNewVersion |

若目標為特殊類型，額外收集：

- **sendEmail**：subject、content、收件 teamIds、emails
- **backToPrevious**：是否清除記錄（isClearRecord）
- **createNewVersion**：從哪個 step 重新開始（startFromStepId）、是否清除記錄
- **close**：是否清除記錄

---

### Step 3：建構 Node Graph

#### 3a. 產生 UUID

計算需要的 node 和 edge 總數，一次產生：

```bash
for i in $(seq <count>); do uuidgen | tr '[:upper:]' '[:lower:]'; done
```

count = step 數 + button 數 + 終端 node 數（close/sendEmail/backToPrevious/createNewVersion）+ edge 數

#### 3b. 建構 Nodes

每個 node 的基本結構：

```json
{
  "id": "<uuid>",
  "type": "<node-type>",
  "position": { "x": 0, "y": 0 },
  "data": { ... }
}
```

**6 種 Node Type 的 data schema：**

**step**

```json
{
  "stepName": "Step 1",
  "triggerStep": true,
  "buttonsAlignment": "right",
  "teamIds": [1, 2],
  "pageIds": ["page-uuid-1"],
  "isLocationBasedSubmission": false,
  "geoFenceRadiusMeters": 0
}
```

> `triggerStep`：第一個 step 設為 `true`，其餘 step 設為 `false`。
> `pageIds`：僅用於 API 儲存參考，**前端不會讀取此欄位來顯示 sections**（詳見「Page-Step 綁定機制」章節）。

**button**

```json
{
  "name": "Submit",
  "color": "#0029A5",
  "order": 0
}
```

**close**

```json
{
  "isCreateNewVersion": false,
  "isClearRecord": false
}
```

**sendEmail**

```json
{
  "subject": "通知信主旨",
  "content": "通知信內容",
  "teamIds": [1],
  "emails": ["user@example.com"]
}
```

**backToPrevious**

```json
{
  "isClearRecord": false
}
```

**createNewVersion**

```json
{
  "startFromStepId": ["<step-node-id>"],
  "isClearRecord": false
}
```

> `startFromStepId` 為陣列，存放新版本要從哪個 step 開始的 step node ID（通常是 step1）。
> `isClearRecord: false` 保留原本資料給新版本參考；`true` 則清空。

#### 3c. 計算 Node Position

參考前端預設 workflow 的 position 模式（水平排列）：

| Node                            | x                    | y    |
| ------------------------------- | -------------------- | ---- |
| Step N                          | `N * 520`            | `0`  |
| Button（接在 Step N 後）        | `N * 520 + 350`      | `69` |
| Close（接在最後一個 Button 後） | `lastButton.x + 170` | `56` |

範例（2-step 線性 workflow）：

- Step 1: `{ x: 0, y: 0 }`
- Button 1: `{ x: 350, y: 69 }`
- Step 2: `{ x: 520, y: 0 }`
- Button 2: `{ x: 870, y: 69 }`
- Close: `{ x: 1040, y: 56 }`

#### 3d. 建構 Edges

> **⚠️ 關鍵限制：Button 的 edge 只能往後（forward），不能往前（backward）。**
> 如果需要退回到前一個 step，**必須使用 `backToPrevious` node**，不能用 button 直接連回前面的 step。
> 違反此規則會導致 workflow 圖形渲染異常（button 會脫離原本的 step 顯示在錯誤位置）。
>
> **但 `backToPrevious` 的 outgoing edge 必須接回前面的 step（往回連）。** 這是唯一允許往回連接的 node 類型。

每條 edge 的結構：

```json
{
  "id": "<uuid>",
  "type": "smoothstep",
  "source": "<source-node-id>",
  "target": "<target-node-id>",
  "sourceHandle": "<sourceType>Out",
  "targetHandle": "<targetType>In",
  "animated": true
}
```

**Handle 命名規則：**

| 連接方向                        | sourceHandle          | targetHandle         |
| ------------------------------- | --------------------- | -------------------- |
| step → button                   | `stepOut`             | `buttonIn`           |
| button → step（只能往後）       | `buttonOut`           | `stepIn`             |
| button → close                  | `buttonOut`           | `closeIn`            |
| button → sendEmail              | `buttonOut`           | `sendEmailIn`        |
| button → backToPrevious         | `buttonOut`           | `backToPreviousIn`   |
| button → createNewVersion       | `buttonOut`           | `createNewVersionIn` |
| sendEmail → step                | `sendEmailOut`        | `stepIn`             |
| sendEmail → close               | `sendEmailOut`        | `closeIn`            |
| backToPrevious → step（往回）   | `backToPreviousOut`   | `stepIn`             |
| createNewVersion → step（往回） | `createNewVersionOut` | `stepIn`             |

> **⚠️ `backToPrevious` 需要兩條 edge：**
>
> 1. `button → backToPrevious`（buttonOut → backToPreviousIn）
> 2. `backToPrevious → 前面的 step`（backToPreviousOut → stepIn）
>
> 如果只有第一條沒有第二條，backToPrevious node 會呈現未連接狀態，workflow 不完整。

> **⚠️ `backToPrevious` 只能接回一個 step（擇一）。**
> 可接回自己所在的 step 或任何前面的 step。例如 3-step workflow 中：Step 2 的 backToPrevious 可以接 Step 1 或 Step 2（擇一），Step 3 的 backToPrevious 可以接 Step 1、Step 2 或 Step 3（擇一）。

> **⚠️ `createNewVersion` 同樣需要兩條 edge：**
>
> 1. `button → createNewVersion`（buttonOut → createNewVersionIn）
> 2. `createNewVersion → 前面的 step`（createNewVersionOut → stepIn）
>
> 雖然 node data 內已用 `startFromStepId` 指明從哪個 step 重啟，但 visual graph 上仍須以 edge 連回該 step，否則 createNewVersion node 會懸空、workflow 不完整。
>
> 連回的 step 必須與 `startFromStepId[0]` 一致。

**常見錯誤：**

- ❌ `button → 前面的 step`：button edge 不能往回連，會導致渲染異常
- ❌ `button → backToPrevious` 但沒有 `backToPrevious → step`：node 會懸空未連接
- ❌ `button → createNewVersion` 但沒有 `createNewVersion → step`：visual graph 不完整
- ❌ `backToPrevious → close`：backToPrevious 應該接回前面的 step，不是 close
- ❌ `backToPrevious` 同時接多個 step：只能接一個 step
- ✅ 正確做法：`button → backToPrevious → 前面的某一個 step`
- ✅ 正確做法：`button → createNewVersion → startFromStepId 指定的 step`

#### 3e. 設定 rootNodeId

```
rootNodeId = nodes[0].id
```

即第一個 step node 的 ID。確保 nodes 陣列中第一個元素是起始 step。

#### 3f. 計算 Validation

對於**線性 workflow**（所有路徑最終都到達 close node），使用以下值：

```json
{
  "isHasTargetStep": true,
  "isAllSendEmail": true,
  "isAllButtonClose": true,
  "isAllBackToPreviousClose": true,
  "isAllCreateNewVersionClose": true,
  "isAllClose": true,
  "detail": {
    "isAllButtonClose": [],
    "isAllBackToPreviousClose": [],
    "isAllCreateNewVersionClose": [],
    "isAllClose": [],
    "stepsWithoutTeamTags": []
  }
}
```

若有 step 沒設 teamIds，將該 step ID 加入 `stepsWithoutTeamTags`。

---

### Step 4：呼叫 MCP Tool 更新 Workflow

呼叫 `mcp__gobuid-mcp-sse__form_template_setting_update_form_workflow`，直接傳入完整 JSON 結構（rootNodeId、nodes、edges、validation）作為參數。

成功後繼續下一步。失敗則告知用戶並詢問是否重試。

---

### Step 4b：綁定 Pages 到 Steps（更新 Form Body）

> **⚠️ 此步驟必須執行，否則 UI 不會顯示 sections。**

更新 workflow graph 後，需要在 form body 的每個 component 的 `setting` 中加入 `steps` 陣列，將 page 綁定到對應的 workflow step。

**操作流程：**

1. 呼叫 `mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id` 取得當前 form body

2. 根據 Step 2 收集的 page-step 對應關係，遞迴更新每個 page 下所有 component 的 `setting.steps`：

   ```python
   # 概念：對每個 page 的每個 component，加入對應的 step 資訊
   for page in pages:
       for row in page['components']:
           for component in row['components']:
               component['setting']['steps'] = [
                   { "id": "<step-node-id>", "name": "<step-name>" },
                   ...  # 該 page 對應的所有 steps
               ]
   ```

   - 若一個 page 屬於多個 steps，`steps` 陣列包含多個 step 物件
   - 同一個 page 下的所有 components 都要加上相同的 `steps`

3. 呼叫 `mcp__gobuid-mcp-sse__form_template_setting_update_form_body` 傳入完整更新後的 `templateId` 和 `pages`

4. 驗證結果：重新呼叫 `mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id`，確認每個 component 的 `setting.steps` 已正確設定。

---

### Step 5：檢查 Team Tag 成員

針對 workflow 中用到的每個 team tag，檢查：

1. **當前用戶是否在該 team tag 中**
   - 如果不在，提醒：
     ```
     你目前不在「品管組」這個 team tag 中，
     建立 submission 時將無法 assign 給自己。
     是否要將自己加入？
     ```

2. **詢問是否有其他成員需要加入**
   ```
   是否有其他成員需要加入以下 team tag？
   請指定：
     - 哪個成員
     - 加入哪個 team tag
     - 在哪個 project 下
   ```

根據確認結果指派：

- 單一成員：呼叫 `mcp__gobuid-mcp-sse__teams_assign_to_user_project`
- 多個成員：呼叫 `mcp__gobuid-mcp-sse__teams_assign_to_multiple_users_and_projects`

---

### Step 6：回報結果

```
Workflow 設定完成：「工程日報表審核流程」

Graph：
  Step 1「填寫申請」(土木工程組) [Pages: 基本資料, 施工資訊]
    → Submit →
  Step 2「主管審核」(品管組) [Pages: 審核意見]
    → Submit → Close

Team tag 成員更新：
  - 王小明 已加入「品管組」（專案：台北捷運工程）
```

---

## 注意事項

- **page-step 綁定有兩層**：workflow node data 的 `pageIds`（API 儲存用）+ form body component 的 `setting.steps`（前端顯示用）。兩者都要設定。
- 每個 page 可以屬於多個 steps（例如審核者需要看到所有 page）。若用戶未指定，可平均分配或詢問。
- **Team tag 只能從當前 project 的清單中選擇**，不可手動輸入新的。
- **每個 step 至少需要一個 team tag**，否則 submission 無法 assign。
- **修改既有 workflow**：先呼叫 `mcp__gobuid-mcp-sse__form_template_setting_get_form_workflow_by_id` 取得現有 graph，展示給用戶，確認要修改的部分後，重建整個 nodes/edges/validation 再更新（全量替換）。
- Button `color` 只能使用以下色碼（預設 `"#0029A5"`）：
  - `#0029A5`（深藍，預設 Submit）
  - `#1BC05D`（綠，適合 Approve）
  - `#CC0954`（紅，適合 Return / Reject）
  - `#BA7D09`（深黃）、`#D4A149`（金）、`#D1BE0D`（亮黃）
  - `#20AC8B`（青綠）、`#23A0CB`（天藍）
  - `#8C51B1`（紫）、`#B196F5`（淡紫）
  - `#D67676`（粉紅）、`#98A3C2`（灰藍）
  - `#6D6245`（棕）、`#000821`（黑）
- 一律透過 MCP tool 操作，不使用 CLI。

---

## 預設 Workflow 範本

### 範本 A：2-Step 線性（Submit → Review → Close）

```json
{
  "rootNodeId": "<step1-id>",
  "nodes": [
    {
      "id": "<step1-id>",
      "type": "step",
      "position": { "x": 0, "y": 0 },
      "data": {
        "stepName": "Step 1",
        "triggerStep": true,
        "buttonsAlignment": "right",
        "teamIds": [20],
        "pageIds": ["<page1-id>"],
        "isLocationBasedSubmission": false,
        "geoFenceRadiusMeters": 0
      }
    },
    {
      "id": "<button1-id>",
      "type": "button",
      "position": { "x": 350, "y": 69 },
      "data": { "name": "Submit", "color": "#0029A5", "order": 0 }
    },
    {
      "id": "<step2-id>",
      "type": "step",
      "position": { "x": 520, "y": 0 },
      "data": {
        "stepName": "Step 2",
        "triggerStep": false,
        "buttonsAlignment": "right",
        "teamIds": [7],
        "pageIds": ["<page1-id>", "<page2-id>"],
        "isLocationBasedSubmission": false,
        "geoFenceRadiusMeters": 0
      }
    },
    {
      "id": "<button2-id>",
      "type": "button",
      "position": { "x": 870, "y": 69 },
      "data": { "name": "Submit", "color": "#0029A5", "order": 0 }
    },
    {
      "id": "<close-id>",
      "type": "close",
      "position": { "x": 1040, "y": 56 },
      "data": { "isCreateNewVersion": false, "isClearRecord": false }
    }
  ],
  "edges": [
    {
      "id": "<e1>",
      "type": "smoothstep",
      "source": "<step1-id>",
      "target": "<button1-id>",
      "sourceHandle": "stepOut",
      "targetHandle": "buttonIn",
      "animated": true
    },
    {
      "id": "<e2>",
      "type": "smoothstep",
      "source": "<button1-id>",
      "target": "<step2-id>",
      "sourceHandle": "buttonOut",
      "targetHandle": "stepIn",
      "animated": true
    },
    {
      "id": "<e3>",
      "type": "smoothstep",
      "source": "<step2-id>",
      "target": "<button2-id>",
      "sourceHandle": "stepOut",
      "targetHandle": "buttonIn",
      "animated": true
    },
    {
      "id": "<e4>",
      "type": "smoothstep",
      "source": "<button2-id>",
      "target": "<close-id>",
      "sourceHandle": "buttonOut",
      "targetHandle": "closeIn",
      "animated": true
    }
  ],
  "validation": {
    "isHasTargetStep": true,
    "isAllSendEmail": true,
    "isAllButtonClose": true,
    "isAllBackToPreviousClose": true,
    "isAllCreateNewVersionClose": true,
    "isAllClose": true,
    "detail": {
      "isAllButtonClose": [],
      "isAllBackToPreviousClose": [],
      "isAllCreateNewVersionClose": [],
      "isAllClose": [],
      "stepsWithoutTeamTags": []
    }
  }
}
```

### 範本 B：2-Step + Approve / Return（含 backToPrevious）

```json
{
  "rootNodeId": "<step1-id>",
  "nodes": [
    {
      "id": "<step1-id>",
      "type": "step",
      "position": { "x": 0, "y": 0 },
      "data": {
        "stepName": "Submit",
        "triggerStep": true,
        "buttonsAlignment": "right",
        "teamIds": [20],
        "pageIds": ["<page1-id>"],
        "isLocationBasedSubmission": false,
        "geoFenceRadiusMeters": 0
      }
    },
    {
      "id": "<btn-submit-id>",
      "type": "button",
      "position": { "x": 350, "y": 69 },
      "data": { "name": "Submit", "color": "#0029A5", "order": 0 }
    },
    {
      "id": "<step2-id>",
      "type": "step",
      "position": { "x": 520, "y": 0 },
      "data": {
        "stepName": "Review",
        "triggerStep": false,
        "buttonsAlignment": "right",
        "teamIds": [7],
        "pageIds": ["<page1-id>", "<page2-id>"],
        "isLocationBasedSubmission": false,
        "geoFenceRadiusMeters": 0
      }
    },
    {
      "id": "<btn-approve-id>",
      "type": "button",
      "position": { "x": 870, "y": 39 },
      "data": { "name": "Approve", "color": "#1BC05D", "order": 0 }
    },
    {
      "id": "<btn-return-id>",
      "type": "button",
      "position": { "x": 870, "y": 99 },
      "data": { "name": "Return", "color": "#CC0954", "order": 1 }
    },
    {
      "id": "<close-id>",
      "type": "close",
      "position": { "x": 1040, "y": 26 },
      "data": { "isCreateNewVersion": false, "isClearRecord": false }
    },
    {
      "id": "<back-id>",
      "type": "backToPrevious",
      "position": { "x": 1040, "y": 86 },
      "data": { "isClearRecord": false }
    }
  ],
  "edges": [
    {
      "id": "<e1>",
      "type": "smoothstep",
      "source": "<step1-id>",
      "target": "<btn-submit-id>",
      "sourceHandle": "stepOut",
      "targetHandle": "buttonIn",
      "animated": true
    },
    {
      "id": "<e2>",
      "type": "smoothstep",
      "source": "<btn-submit-id>",
      "target": "<step2-id>",
      "sourceHandle": "buttonOut",
      "targetHandle": "stepIn",
      "animated": true
    },
    {
      "id": "<e3>",
      "type": "smoothstep",
      "source": "<step2-id>",
      "target": "<btn-approve-id>",
      "sourceHandle": "stepOut",
      "targetHandle": "buttonIn",
      "animated": true
    },
    {
      "id": "<e4>",
      "type": "smoothstep",
      "source": "<step2-id>",
      "target": "<btn-return-id>",
      "sourceHandle": "stepOut",
      "targetHandle": "buttonIn",
      "animated": true
    },
    {
      "id": "<e5>",
      "type": "smoothstep",
      "source": "<btn-approve-id>",
      "target": "<close-id>",
      "sourceHandle": "buttonOut",
      "targetHandle": "closeIn",
      "animated": true
    },
    {
      "id": "<e6>",
      "type": "smoothstep",
      "source": "<btn-return-id>",
      "target": "<back-id>",
      "sourceHandle": "buttonOut",
      "targetHandle": "backToPreviousIn",
      "animated": true
    },
    {
      "id": "<e7>",
      "type": "smoothstep",
      "source": "<back-id>",
      "target": "<step1-id>",
      "sourceHandle": "backToPreviousOut",
      "targetHandle": "stepIn",
      "animated": true
    }
  ],
  "validation": {
    "isHasTargetStep": true,
    "isAllSendEmail": true,
    "isAllButtonClose": true,
    "isAllBackToPreviousClose": true,
    "isAllCreateNewVersionClose": true,
    "isAllClose": true,
    "detail": {
      "isAllButtonClose": [],
      "isAllBackToPreviousClose": [],
      "isAllCreateNewVersionClose": [],
      "isAllClose": [],
      "stepsWithoutTeamTags": []
    }
  }
}
```

> **注意範本 B 的 edge `<e7>`**：`backToPrevious → step1`，將表單退回到 Step 1 或是 Step 2。backToPrevious 只能接回當前 Step 或是前面任一個 Step。

> 使用時將所有 `<...-id>` 替換為實際產生的 UUID，並填入正確的 teamIds 和 pageIds。
> 呼叫 `mcp__gobuid-mcp-sse__form_template_setting_update_form_workflow` 傳入完整 JSON 結構。
> **別忘了執行 Step 4b，更新 form body 的 `setting.steps` 以綁定 pages 到 steps。**
