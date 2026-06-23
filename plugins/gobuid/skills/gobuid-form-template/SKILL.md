---
name: gobuid-form-template
description: GoBuid 表單範本的建立與更新工作流程。支援從截圖、PDF、xlsx 或文字描述解析欄位結構，引導用戶設定欄位類型與選項，最後透過 MCP tool 建立或更新 form template。
allowed-tools: mcp__gobuid-mcp-sse__*, Bash(uuidgen *), Bash(for *)
---

## 前置需求

### GoBuid MCP Server

此 skill 透過 GoBuid MCP Server（`gobuid-mcp-sse`）操作 API，執行前須確認：

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server
2. 已設定 context：呼叫 `mcp__gobuid-mcp-sse__set_context` 選擇 group

驗證：呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常。

### 其他（選用）

若需要從 PDF 或 Excel 建立表單，請確認已安裝對應的 skills：

```bash
npx skills add https://github.com/anthropics/skills/tree/main/skills/pdf
npx skills add https://github.com/anthropics/skills/tree/main/skills/xlsx
```

## 使用時機

當用戶說「建立表單」、「新增表單範本」、「更新表單」、「幫我做一個表單」時使用此 skill。

---

## 支援的欄位類型

| 類型                | 說明                        |
| ------------------- | --------------------------- |
| `shortText`         | 單行文字                    |
| `longText`          | 多行文字                    |
| `headerSection`     | 區塊標題（純展示，無輸入）  |
| `dateTime`          | 日期時間                    |
| `singleSelection`   | 單選（radio）               |
| `multipleSelection` | 多選（checkbox）            |
| `dropdown`          | 下拉選單                    |
| `signature`         | 簽名                        |
| `upload`            | 檔案上傳                    |
| `image`             | 圖片（純展示）              |
| `inputTable`        | 輸入表格（含 Yes/No 等欄）  |
| `table`             | 靜態表格                    |
| `subForm`           | 子表單                      |
| `richText`          | 富文字（HTML 內容，純展示） |
| `calculation`       | 計算欄位（公式自動計算）    |

---

## ID 產生規則

所有 id 一律使用真實 UUID，透過以下指令產生：

```bash
# 產生單個 UUID
uuidgen | tr '[:upper:]' '[:lower:]'

# 一次產生多個（例如 10 個）
for i in $(seq 10); do uuidgen | tr '[:upper:]' '[:lower:]'; done
```

**建立表單前，先估算所需 ID 總數（pages + rows + fields），一次產生足夠的 UUID，再逐一填入 JSON 結構。**

不要手刻 UUID、不要重複使用、不要使用可讀短 ID（如 `page-1`）。

---

## 欄位 JSON Schema 速查

> `update_form_body` 需要一次性傳入完整 `pages` 陣列，**沒有逐欄位新增的 API**。
> 每個 page 對應一個 section tab；每個 page 下有多個 row；每個 row 包含 1～多個 field。

### 外層結構

```json
{
  "templateId": 123,
  "pages": [
    {
      "id": "page-1",
      "type": "page",
      "class": "",
      "name": "Section 名稱",
      "childSetting": {},
      "components": [
        /* rows */
      ]
    }
  ]
}
```

> **headerSection 為選用。** 如果用戶需要表單標頭（顯示 Reference No.、Submittal Date、Project 等資訊），可在第一個 page 的第一個 row 加入 headerSection。不是每個表單都需要。

### Row 結構

```json
{
  "id": "row-1-1",
  "type": "row",
  "class": "",
  "childSetting": {},
  "components": [
    /* fields */
  ]
}
```

### headerSection（第一個 page 的第一個 field，選用）

`headerSection` 沒有用戶輸入，是純展示的表單標頭區塊。若需要，必須是第一個 row 的唯一 component。

```json
{
  "id": "field-1-1-1",
  "type": "headerSection",
  "class": "",
  "components": null,
  "readonly": false,
  "required": false,
  "visible": false,
  "value": "",
  "setting": {
    "items": {
      "referenceNumber": {
        "id": "referenceNumber",
        "label": "Reference No.",
        "order": 1,
        "enabled": true
      },
      "submittalDate": {
        "id": "submittalDate",
        "label": "Submittal Date",
        "order": 2,
        "enabled": true
      },
      "company": {
        "id": "company",
        "label": "Company",
        "order": 3,
        "enabled": false
      },
      "project": {
        "id": "project",
        "label": "Project",
        "order": 0,
        "enabled": true
      },
      "projectDuration": {
        "id": "projectDuration",
        "label": "Project Duration",
        "order": 4,
        "enabled": false
      },
      "projectLocation": {
        "id": "projectLocation",
        "label": "Project Location",
        "order": 5,
        "enabled": false
      },
      "projectDescription": {
        "id": "projectDescription",
        "label": "Project Description",
        "order": 6,
        "enabled": false
      },
      "validDate": {
        "id": "validDate",
        "label": "Valid Date",
        "order": 7,
        "enabled": false
      }
    }
  }
}
```

### shortText

```json
{
  "id": "field-1-1-1",
  "type": "shortText",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": { "label": "欄位名稱", "format": "text", "localAlignment": "top" }
}
```

`format` 可為 `"text"` / `"email"` / `"number"`

### longText

```json
{
  "id": "field-1-1-1",
  "type": "longText",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": { "label": "欄位名稱", "format": "text", "localAlignment": "top" }
}
```

### singleSelection

```json
{
  "id": "field-1-1-1",
  "type": "singleSelection",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": {
    "label": "欄位名稱",
    "col": 3,
    "choices": "custom",
    "options": ["選項A", "選項B", "選項C"],
    "other": false,
    "otherLabel": "Other",
    "localAlignment": "top"
  }
}
```

### dropdown

```json
{
  "id": "field-1-1-1",
  "type": "dropdown",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": {
    "label": "欄位名稱",
    "choices": "custom",
    "options": ["選項A", "選項B"],
    "multiple": false,
    "other": false,
    "otherLabel": "Other"
  }
}
```

### dateTime

> **⚠️ label 與 description 放在 `setting`，不是 `value`。**
> `value` 物件只存實際填入的日期值（date / startsDate / endsDate）。把 label 寫進 `value.label` 前端不會渲染。

```json
{
  "id": "field-1-1-1",
  "type": "dateTime",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": {
    "date": "",
    "endsDate": "",
    "dateLabel": "",
    "endsLabel": "Ends",
    "startsDate": "",
    "startsLabel": "Starts"
  },
  "inputValue": null,
  "components": null,
  "setting": {
    "label": "欄位名稱",
    "description": "",
    "fieldType": "date",
    "rangeType": "single",
    "dateFormat": "YYYY/MM/DD",
    "timeFormat": "24",
    "defaultValue": "none",
    "localAlignment": "top",
    "useDefaultAlignment": false
  }
}
```

> **`useDefaultAlignment`**：設 `true` 會走 form-level 預設 alignment，可能導致 label 不顯示。
> 若要確保 label 可見，設 `false` 並指定 `localAlignment`（`"top"` / `"left"`）。

### signature

```json
{
  "id": "field-1-1-1",
  "type": "signature",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": { "localAlignment": "top", "useDefaultAlignment": true }
}
```

### inputTable（Yes / No / Remarks 表格）

`components` 是一個二維陣列：第一個子陣列為 header row，其後每個子陣列為一筆資料 row。

```json
{
  "id": "field-1-3-1",
  "type": "inputTable",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": { "label": "表格標題" },
  "inputValue": null,
  "setting": { "defaultType": "InputTableSingleSelection" },
  "components": [
    [
      { "id": "th-1-1", "type": "InputTableSingleSelection", "width": 260 },
      {
        "id": "th-1-2",
        "type": "InputTableSingleSelection",
        "title": "Yes",
        "width": 70
      },
      {
        "id": "th-1-3",
        "type": "InputTableSingleSelection",
        "title": "No",
        "width": 70
      },
      {
        "id": "th-1-4",
        "type": "InputTableTextInput",
        "title": "Remarks",
        "width": 180
      }
    ],
    [
      {
        "id": "td-1-1-1",
        "type": "InputTableSingleSelection",
        "title": "問題內容"
      },
      { "id": "td-1-1-2", "type": "InputTableSingleSelection", "value": "" },
      { "id": "td-1-1-3", "type": "InputTableSingleSelection", "value": "" },
      { "id": "td-1-1-4", "type": "InputTableTextInput", "value": "" }
    ]
  ]
}
```

可用的 column 類型：

- `InputTableSingleSelection` — radio 選項欄
- `InputTableTextInput` — 文字輸入欄
- `InputTableNonEditableText` — 靜態文字欄
- `InputTableDropdown` — 下拉選單欄（需附 `options` 陣列）

> **⚠️ inputTable data row 的顯示屬性規則：**
>
> - **第一欄**（index 0）：使用 `title` 顯示文字，`value` 會被忽略
> - **其他欄位**（index 1+）：使用 `value` 顯示文字，`title` 會被忽略
>
> 此規則適用於所有 column 類型（`InputTableSingleSelection`、`InputTableNonEditableText` 等）。

> **⚠️ inputTable 不會自動為每筆 data row 加序號。**
> 若原始表單有 `1)`、`2)`、`3)` 序號（常見於 checklist），必須在每個 item 的 `title` 字串前手動加上前綴：
>
> ```json
> { "type": "InputTableSingleSelection", "title": "1) RA / SWP briefed to workers?" }
> { "type": "InputTableSingleSelection", "title": "2) Are the hoardings erected ...?" }
> ```
>
> 解析 xlsx / PDF 時，若原始資料有 `1)`、`(1)`、`①` 等序號，請保留並寫進 `title`。

### richText（富文字，純展示）

`richText` 沒有 setting，內容以 HTML 字串存在 `value` 中。

```json
{
  "id": "field-1-1-1",
  "type": "richText",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "<h1>標題</h1><p>內容</p>",
  "setting": null,
  "components": null
}
```

### calculation（計算欄位）

透過 `formula` 參照其他欄位進行數學運算，結果自動計算、唯讀。

```json
{
  "id": "field-1-1-1",
  "type": "calculation",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": {
    "label": "合計",
    "description": "單價 × 數量",
    "decimalPlaces": 2,
    "formula": [
      {
        "id": "<uuid>",
        "type": "field",
        "value": "單價",
        "fieldId": "<price-field-uuid>",
        "fieldType": "shortText",
        "label": "單價"
      },
      {
        "id": "<uuid>",
        "type": "operator",
        "value": "*"
      },
      {
        "id": "<uuid>",
        "type": "field",
        "value": "數量",
        "fieldId": "<qty-field-uuid>",
        "fieldType": "shortText",
        "label": "數量"
      }
    ]
  }
}
```

`formula` 陣列中每個項目的 `type`：

| type           | 說明                       | 範例 value      |
| -------------- | -------------------------- | --------------- |
| `field`        | 參照其他欄位（需 fieldId） | 欄位 label      |
| `number`       | 數值常數                   | `"100"`         |
| `operator`     | 運算子                     | `+` `-` `*` `/` |
| `leftBracket`  | 左括號                     | `"("`           |
| `rightBracket` | 右括號                     | `")"`           |

> **限制：** 小數位最多 2 位、整數部分最多 10 位。被參照的欄位必須存在，否則公式會失效。

---

## 常見 Pattern

### Pattern A：表單最上方 Logo + 標題 + 副標說明

紙本表單常見的「公司 logo + 表單名稱 + 一段免責聲明」區塊。GoBuid 沒有原生的「form header with logo」欄位類型，要組合 `image` + `richText` 達成。

放在第一個 page 的第一個 row，並排兩個 components：

```json
{
  "id": "<row-uuid>",
  "type": "row",
  "class": "",
  "childSetting": {},
  "components": [
    {
      "id": "<image-uuid>",
      "type": "image",
      "class": "",
      "visible": false,
      "required": false,
      "readonly": false,
      "value": "https://<uploaded-logo-url>",
      "setting": { "width": 120 },
      "components": null
    },
    {
      "id": "<richtext-uuid>",
      "type": "richText",
      "class": "",
      "visible": false,
      "required": false,
      "readonly": false,
      "value": "<h2 style=\"text-align:center\"><u>PERMIT TO WORK - DEMOLITION</u></h2><p>This permit shall be displayed for the duration of the approved tasks and removed only upon task completion or upon its expiry.</p>",
      "setting": null,
      "components": null
    }
  ]
}
```

> **logo 圖片來源**：先透過 `mcp__gobuid-mcp-sse__file_upload_file` 或 `photo_upload_photo` 上傳，取得 URL 後填入 `value`。或若用 base64 內嵌，可寫進 `richText` 的 `<img src="data:image/png;base64,...">`（檔案會變大，僅限小 logo）。

---

### Pattern B：Declaration Confirmation Checkbox

紙本表單常有「☐ I declare that the information provided is accurate...」這類確認框。GoBuid 沒有獨立的 boolean / checkbox 欄位類型，用 `multipleSelection` 配單一 option 達成：

```json
{
  "id": "<field-uuid>",
  "type": "multipleSelection",
  "class": "",
  "visible": false,
  "required": false,
  "readonly": false,
  "value": "",
  "inputValue": null,
  "components": null,
  "setting": {
    "label": "Declaration",
    "col": 1,
    "choices": "custom",
    "options": [
      "I declare that the information provided is accurate and the control measures listed above have been effectively implemented."
    ],
    "other": false,
    "otherLabel": "Other",
    "localAlignment": "top"
  }
}
```

UI 上呈現為一個可勾選的 checkbox + 旁邊顯示完整宣告文字。`required: true` 可強制使用者必須勾選才能提交。

---

## MCP Tool 參考

```
# 建立表單範本（空模板）
mcp__gobuid-mcp-sse__form_templates_post

# 列出表單範本
mcp__gobuid-mcp-sse__form_templates_get_form_templates_by_page_filter

# 查看表單範本欄位結構
mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id
mcp__gobuid-mcp-sse__form_template_setting_get_form_detail_by_id

# 更新表單欄位
mcp__gobuid-mcp-sse__form_template_setting_update_form_body

# 刪除表單範本
mcp__gobuid-mcp-sse__form_templates_delete_forever
```

---

## 多頁 JSON 格式

當表單需要分多個 section（對應 workflow 的不同 step），**必須使用 `pages` 格式**：

```json
{
  "templateId": 123,
  "pages": [
    {
      "id": "<page-uuid>",
      "type": "page",
      "class": "",
      "name": "Page 1 名稱",
      "childSetting": {},
      "components": [
        {
          "id": "<row-uuid>",
          "type": "row",
          "class": "",
          "childSetting": {},
          "components": [
            {
              /* field */
            }
          ]
        }
      ]
    }
  ]
}
```

> **⚠️ 如果之後要設定 workflow，一定要用多頁格式。** 每個 workflow step 需要透過 form body component 的 `setting.steps` 綁定顯示哪些 page（詳見 `/gobuid-form-workflow` skill 的「Page-Step 綁定機制」章節），單頁表單會導致所有 step 都看到全部欄位，無法區分。

### Row 分組（欄位並排）

若需要多個欄位並排在同一列，將它們放在同一個 row 的 `components` 中：

- 單一 field 在 row 中 → 獨立佔一行
- 多個 fields 在同一 row 中 → 同一列並排顯示

---

## 執行流程

### 前置

確認 MCP server 已連接（呼叫 `mcp__gobuid-mcp-sse__user_self` 確認回傳正常）。

---

### Step 1：解析表單來源

根據用戶提供的格式選擇對應的解析方式：

- **截圖** → 直接讀取圖片，識別欄位名稱與版面配置
- **PDF** → 呼叫 `pdf` skill 解析內容
- **xlsx** → 呼叫 `xlsx` skill 解析欄位結構
- **文字描述** → 直接根據描述整理欄位清單

解析時特別注意：

- 並排欄位不要拆成獨立行，保留同一列的空間關係
- 有無標題區塊（`headerSection`）
- **表單最上方有無 logo 圖片 / 大標題 / 副標說明** → 用 Pattern A（image + richText 組合，見上方「常見 Pattern」章節）
- **「I declare that ...」這類 declaration checkbox** → 用 Pattern B（multipleSelection 配單一 option）
- 是否有需要 options 的欄位（選擇類型）
- Yes/No 表格 → `inputTable`（若原始有 `1)`、`2)` 序號，必須在每筆 item 文字前手動加前綴）

---

### Step 2：列出欄位結構，請用戶確認

將解析結果整理成以下格式，請用戶確認後再繼續：

```
解析結果如下，請確認是否正確：

Page 1：基本資料
1. 工程名稱 → shortText（必填）
2. 施工日期 → dateTime
3. 施工類別 → dropdown
   options：地基 / 鋼筋 / 灌漿 / 其他

Page 2：檢查項目
4. 安全檢查 → inputTable（Yes / No / Remarks，5 筆）
5. 簽名 → signature

是否需要調整？
```

**針對選擇類型欄位（singleSelection / multipleSelection / dropdown）**，如果 options 尚未確認，**強制詢問**：「請提供這個欄位的選項清單。」

---

### Step 3：建立或更新 form template

**新建：**

1. 詢問範本名稱
2. 詢問是否需要設定 workflow → 決定使用多頁或單頁格式
3. 呼叫 `mcp__gobuid-mcp-sse__form_templates_post` 建立空模板，取得 `templateId`
4. 估算所需 UUID 總數（pages + rows + fields），一次產生：
   ```bash
   for i in $(seq <count>); do uuidgen | tr '[:upper:]' '[:lower:]'; done
   ```
5. 組裝完整 `pages` JSON 結構
6. 呼叫 `mcp__gobuid-mcp-sse__form_template_setting_update_form_body`，傳入 `templateId` 和完整 `pages` 陣列

> **⚠️ 禁止使用 `SaveAsNew` API。** 該 API 會回傳 400 驗證錯誤。必須使用上述兩步驟流程：先建立空模板，再更新欄位。

**更新：**

1. 詢問要更新哪個範本（名稱或 ID）
2. 呼叫 `mcp__gobuid-mcp-sse__form_template_setting_get_form_body_by_id` 取得現有結構，列出給用戶參考
3. 詢問：新增欄位、修改現有欄位，還是刪除欄位？
4. 在現有 pages JSON 上修改後，呼叫 `mcp__gobuid-mcp-sse__form_template_setting_update_form_body` 傳入完整更新後的 `templateId` 和 `pages`

---

### Step 4：詢問是否繼續設定 workflow

建立完 form template 後，詢問用戶：「是否要繼續設定這個表單的 workflow？」

- **是** → 執行 `/gobuid-form-workflow` skill，傳入剛建立的 template ID
- **否** → 進入 Step 5 回報結果

> 如果用戶要設定 workflow，表單**必須已使用多頁格式**建立。若當初用單頁建立，需先更新表單結構為多頁再設定 workflow。

---

### Step 5：回報結果

```
✅ 表單範本建立完成：「工程日報表」（ID: 456）

Page 1：基本資料
- 工程名稱（shortText）
- 施工日期（dateTime）
- 施工類別（dropdown）- 4 個選項

Page 2：檢查項目
- 安全檢查（inputTable）- 5 筆
- 簽名（signature）

如需設定審核流程，請執行 /gobuid-form-workflow
```

---

## 注意事項

- **一定要讓用戶確認欄位結構**後才建立，除非用戶明確說「直接建立」
- 選擇類型欄位（singleSelection / multipleSelection / dropdown）**必須確認 options**
- 並排欄位要詢問用戶是否要保留並排版面配置
- `update_form_body` 是**全量更新**，每次都要傳入完整 pages，不能只傳修改的部分
- 如果 API 呼叫失敗，告知用戶並詢問是否重試
- 一律透過 MCP tool 操作，不使用 CLI
