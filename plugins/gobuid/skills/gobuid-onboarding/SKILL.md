---
name: gobuid-onboarding
description: Use this to onboard a new GoBuid customer — take one filled-in setup/onboarding spreadsheet (Google Sheet or Excel, the kind Customer Success fills out) and build the customer's whole GoBuid group in one pass instead of creating each item by hand. It covers everything the sheet describes at once: group settings (帳號名稱、工時、地理圍籬/geofence), projects, budget codes, equipment, and member/participant invites. Trigger on intents like: onboard this customer / run project setup / 幫這個客戶做 onboarding、跑 project setup、照這份 sheet 把整個 group（帳號、專案、預算、設備、成員）設定起來、客戶初始設定表批次建到 GoBuid（不要一個一個開）、import the CS onboarding spreadsheet into GoBuid — or just a pasted setup-sheet link with "把資料建到系統/GoBuid". The tell: one spreadsheet → many GoBuid records across several categories. Not for a single project/activity/budget create-or-edit, nor for just reading/exporting a sheet. Always plans and confirms before writing.
allowed-tools: mcp__gobuid-mcp-sse__*, Bash(gws *)
---

## 使用時機

當使用者要把一份 GoBuid onboarding / project setup 試算表（Customer Success 填的那種）自動建進系統時使用。觸發語如「幫這個客戶做 onboarding」、「跑 project setup」、「照這份 sheet 建起來」、「匯入 CS 設定表」，或直接丟一個 GoBuid 設定表的 Google Sheet 連結。

這個 skill 把整份表拆成六個模組，逐一對應到後端 API，**先算出完整計畫給人確認，確認後才真的寫入**。

---

## 為什麼要先算計畫再寫入（最重要的原則）

> **⚠️ GoBuid 後端對這些 create API 幾乎不檢查必填欄位。** 你少帶欄位、把名稱對錯 id、或送出 `0,0` 座標，後端多半照樣「建立成功」，只是資料殘缺且很難回收。

所以把關責任在這個 skill，不在後端。流程一定是：**讀 → 解析 → 對照 name→id → 產生人類可讀的計畫 → 等使用者確認 → 才執行**。任何一列對不到 id、或座標查不到，就在計畫裡標成「需處理」，不要硬建。

---

## 前置條件

1. Claude Code 已連接 `gobuid-mcp-sse` MCP server（第一次呼叫 tool 會自動開瀏覽器 OAuth）。驗證：呼叫 `user_self` 回傳正常。
2. **已設定 group context**：`set_context` 選定要 onboard 的 group。所有 group 層級設定（名稱 / geofence / 工時）、equipment、單位、teams 都是 per-group；project 也建在這個 group 底下。沒設先用 `/gobuid-list-groups`。
3. **本機有 `gws`（Google Workspace CLI）** 且已登入，用來讀 Google Sheet。
4. **試算表已開放讀取權限**，且是本 skill 預期的結構化格式（六個分頁，見下）。使用者提供 spreadsheet URL 或 ID。

---

## 預期的試算表結構

六個資料分頁（欄位順序固定，第一列是標題，資料從第二列起）：

| 分頁           | 欄位                                                                                                                                                                                                                       |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `General`      | 兩個區塊：`A1:C6` 設定 key-value（Account Name、Account Owner Email、Geofence Radius、Daily Start Time、Daily End Time）；`A8:C15` 每週工時表（Day / Working Hours / Non-working Day，週一→週日）。全部是 group 層級設定。 |
| `Equipment`    | Equipment Name / Ownership / Equipment Type / Working Hours/Day                                                                                                                                                            |
| `Budget`       | Project Name / Budget Code Name / Total Quantity / Unit / Category                                                                                                                                                         |
| `Activity`     | Project Name / Activity Name / Total Quantity / Unit / Start Date / End Date / Team Tags(選填) / Notes(選填)                                                                                                               |
| `Projects`     | Project Name / Start Date / End Date / Location / Latitude(選填) / Longitude(選填)                                                                                                                                         |
| `Participants` | Project Name / Email / Role / Seat Type / Team Tags                                                                                                                                                                        |

> **Budget vs Activity**：兩者都用「TotalQty + 單位」，但 **Budget 追資源/預算**、**Activity 追進度**（常用 `%` 單位做 0–100% 進度）。是不同 API、不同分頁，別混。

**跳過範例列**：任何一列若主要欄位（Project Name / Equipment Name / Budget Code Name / Activity Name / Email）以 `e.g.` 開頭，是灰色示範列，略過不讀。空白列也略過。

`Budget` / `Activity` / `Participants` 用 `Project Name` 欄對應到 `Projects` 分頁的專案（join key），據此把 budget code、activity、成員歸到正確的 project。

---

## 執行流程

### Step 1 — 讀取試算表

用 `gws` 一次把所有需要的範圍抓下來（`gws` 的 stdout 前面會多一行 `Using keyring backend`，解析 JSON 前先濾掉）：

```bash
gws sheets spreadsheets values batchGet \
  --params '{"spreadsheetId":"<ID>","ranges":[
    "General!A1:C15",
    "Equipment!A2:D200","Budget!A2:E200","Activity!A2:H200","Projects!A2:F60","Participants!A2:E400"]}' \
  --format json 2>/dev/null | sed '/Using keyring/d'
```

### Step 2 — 解析並做 name→id 對照

把每一列轉成結構化資料，並用 `references/mappings.md` 把人類可讀的值換成 API 要的 id / 格式。**這是最容易靜默出錯的地方**，務必逐項對照：

- Equipment Ownership、Equipment Type(9類)、Budget Category(7類)、Role、Seat Type、Group Role → **全是寫死對照表**，直接查 `mappings.md`。
- **Budget / Activity Unit** → 先比對 `mappings.md` 的 15 個系統預設單位；不在表內的，呼叫 `activity_query_activity_units_by_group_id` 查該 group 是否已有；還是沒有才 `activity_create_activity_unit` 建立，取回 `activityUnitId`。（`%` = id 15，會鎖 `TotalQty=100`。）Budget 與 Activity 共用同一套單位。
- **Activity 起訖日** → sheet 留空時，沿用該 Activity 所屬 project 的起訖日；有填就用填的。
- **每週工時陣列**（General 分頁 `A8:C15` 的每週工時表）：sheet 是週一→週日，但 API 的 `weeklyUserAttendanceRecordHours` / `weeklyEquipmentNonWorkingDays` 都是**星期日開頭**，務必重排。工時的起訖時間與 Geofence 取自 General 設定區（`A1:C6`）。詳見 `mappings.md`。
- **icon / color** 不在 sheet 裡，套固定預設（見 `mappings.md`）。

對不到 id 的列先記下來，標成「需處理」，不要中止整個流程。

### Step 3 — 處理 Project 座標

每個 project：

1. 有填 Latitude/Longitude → 直接用。
2. 沒填 → 用 `geocoding_place_autocomplete` 找地址，取第一個有信心的結果，再用 `geocoding_address_lookup`／place 細節拿座標。
3. **geocode 失敗、模糊、或得到 `0,0`** → 標成「需處理：座標無法確定」，**不要建立**這個 project（連帶它底下的 budget / participants 也一起 hold）。

### Step 4 — 印出計畫並等待確認

用清楚的中文摘要列出「將建立什麼」，分模組呈現，並把所有「需處理」項目集中在最上面醒目列出。格式見下方「計畫輸出格式」。**印完後停下來，等使用者明確說「確認 / 執行」才進 Step 5。**

### Step 5 — 依相依順序執行

嚴格照這個順序（細節、tool 名稱、payload 形狀見 `references/api-sequence.md`）：

```
1. Group 設定（與 project 無關，可先做）
   - group_update_group_name（Account Name）
   - group_update_group_geofence_radius（含 isGeofenceRadiusCheck=true 才生效）
   - group_setting_working_time_put（星期日開頭陣列 + defaultWorkDayHours）
2. 每個 Project：
   a. project_create_project（驗座標非 0,0）→ 取回 projectId
   b. 該 project 的 Budget → budget_create_budget_code（帶 projectId + unitId + categoryId）
   b2. 該 project 的 Activity → activity_v2_activity_v2_activities（帶 projectId + activityUnitId + 起訖日；日期空則沿用 project 起訖；% 鎖 100；teamIds 選填）
   c. 該 project 的 Participants：
      - 先確保需要的 Team 存在：teams_get 比對；缺的 teams_post 建立 → 取 teamId
      - user_invite_user_to_project（emails / roleId / teamIds / seatTypeId / groupRoleId / projectIds）
3. Equipment（group 層級）：equipment_create_equipment
4. Account Owner：待對應成員存在後，admin_console_modify_member_as_co_owner（userId）
   - ⚠️ 剛邀請、尚未接受的成員可能還沒有可用的 userId；查不到就標「待成員接受後再設」，不要硬設。
```

每一步的結果（成功 / 失敗 + 回傳 id）即時記錄，最後產出總結報告。

---

## 計畫輸出格式（Step 4）

先列需處理，再分模組列將建立項目：

```
## ⚠️ 需先處理（這些不會被建立）
- Projects「XXX」：地址查不到座標，請補 Latitude/Longitude
- Budget「YYY」：Project Name「ZZZ」在 Projects 分頁找不到對應專案

## 將建立的設定
### Group 設定
- Account Name: ...
- Geofence: 100m（啟用）
- 工時: 週一~週五 8h（09:00–18:00），週六日為非工作日

### Projects（N 筆）
- 大樓 A ｜ 2026/07/01–2026/12/31 ｜ 台北市... ｜ 座標 25.03,121.56 ✓

### Budget（N 筆） / Equipment（N 筆） / Participants（N 筆）
...

### Account Owner
- owner@company.com（設為 co-owner）

確認後我就開始建立。要執行嗎？
```

---

## 總結報告（Step 5 之後）

執行完列出每個模組的成功 / 失敗數與失敗原因，並把 Step 4 標記的「需處理」項目再列一次提醒使用者手動補。

---

## 參考檔

- `references/mappings.md` — 所有 name→id 對照表、預設值、星期陣列規則、`%`/geofence 特例。**解析前必讀。**
- `references/api-sequence.md` — 每支 tool 的正確名稱、payload 欄位、相依順序與各欄位資料來源。**執行前必讀。**
