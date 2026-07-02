# API 呼叫順序與 payload

執行前必讀。所有 tool 都在 `gobuid-mcp-sse` 上。`set_context` 設定的 groupId 會自動帶進每個 request 的 `GroupId` header，所以多數 group 層級 tool 不用自己傳 groupId（但傳了也無妨）。

## 相依關係（為什麼是這個順序）

```
Group 設定 ── 無相依，先做
Project ──► 取得 projectId ──► Budget（需 projectId）
                          ├─► Activity（需 projectId；先建 Team 取 teamId，選填）
                          └─► Participants（需 projectId；先建 Team 取 teamId）
Participants 成為成員 ──► Account Owner（需該成員 userId）
Equipment ── group 層級，任何時候都可做
```

Budget、Activity、Participants 都靠 sheet 的 `Project Name` join 到某個 project；那個 project 必須先建成功、且座標有效，否則整組 hold。

---

## 1. Group 設定

> **⚠️ 這兩支必須在 body 帶 `groupId`，不能只靠 `set_context` 的 header。**
> 只靠 header 會回 403 `errorCode 5`「Only owner or co-owner can operate」——**即使呼叫者就是 Owner** 也一樣。帶上 body 的 `groupId` 才成功。
> （對比：`group_setting_working_time_put` 只靠 header 就 OK，別被它誤導以為全部都吃 header。）

### group_update_group_name

`PUT /api/Group/UpdateGroupName`

```json
{ "groupId": <groupId>, "name": "<Account Name>" }
```

送 `groupId` + `name`（不要連帶送 geofence 欄位）。

### group_update_group_geofence_radius

`PUT /api/Group/UpdateGroupGeofenceRadius`

```json
{ "groupId": <groupId>, "isGeofenceRadiusCheck": true, "geofenceRadius": <公尺整數> }
```

> `isGeofenceRadiusCheck` 沒設 true 的話，半徑填了也不會生效。

### group_setting_working_time_put

`PUT /api/GroupSetting/WorkingTime`

```json
{
  "weeklyUserAttendanceRecordHours": [0, 8, 8, 8, 8, 8, 0],
  "weeklyEquipmentNonWorkingDays": [
    true,
    false,
    false,
    false,
    false,
    false,
    true
  ],
  "defaultWorkDayHours": { "startTime": "09:00:00", "endTime": "18:00:00" }
}
```

陣列**星期日開頭**（見 `mappings.md`）。上面範例 = 週一~週五 8h、週末非工作日。

---

## 2. Project

### project_create_project

`POST /api/Project/CreateProject`

```json
{
  "name": "<Project Name>",
  "address": "<解析後的完整地址>",
  "latitude": 25.03,
  "longitude": 121.56,
  "startDate": "2026-07-01T00:00:00.000Z",
  "endDate": "2026-12-31T00:00:00.000Z",
  "icon": "trafficCone",
  "color": "#E9BF78"
}
```

- **建立前再確認 latitude/longitude 不是 0,0**；是就別建。
- 回傳裡取 `projectId` 供 budget / participants 用。

---

## 3. Budget（每個 project 底下）

### budget_create_budget_code

`POST /api/Budget/CreateBudgetCode`（multipart form-data）

```json
{
  "ProjectId": <projectId>,
  "Name": "<Budget Code Name>",
  "TotalQty": <數字>,
  "UnitId": <對照/查得的 unitId>,
  "CategoryId": <1-7>
}
```

`UnitId` 為 15（%）時 `TotalQty` 應為 100。

---

## 3b. Activity（每個 project 底下，進度追蹤）

### activity_v2_activity_v2_activities

`POST /api/Activity/v2/Activities`（multipart form-data）

```json
{
  "ProjectId": <projectId>,
  "Name": "<Activity Name>",
  "TotalQty": <數字>,
  "ActivityUnitId": <對照/查得的 unitId>,
  "StartDate": "2026-08-01T00:00:00.000Z",
  "EndDate": "2027-02-28T00:00:00.000Z",
  "TeamIds": [<teamId>, ...]
}
```

- 與 Budget 共用單位系統：`ActivityUnitId` = 15（%）時 `TotalQty` 應為 100（Activity 常用 % 做進度）。
- `StartDate` / `EndDate`：sheet 留空時，沿用該 project 的起訖日；有填則轉 ISO 8601 UTC。
- `TeamIds`（選填）：同 Participants，先 `teams_get` 比對、缺的 `teams_post` 建立取 id；sheet Team Tags 逗號分隔可多個。
- `Notes`（選填）、`Location`/`Latitude`/`Longitude`（選填）依 sheet 有值才帶。

---

## 4. Participants（每個 project 底下）

### 先建/對 Team

- `teams_get`（`GET /api/Teams`）拿現有 teams，用 name 比對取 `id`。
- 缺的 team → `teams_post`（`POST /api/Teams`，`{ "name": "<tag>" }`，`name` 是唯一被後端標為必填的欄位）取回新 `id`。
- 一個成員可有多個 team tag（sheet 以逗號分隔）→ 收集成 `teamIds` 陣列。

### user_invite_user_to_project

`POST /api/User/InviteUserToProject`

```json
{
  "emails": ["<email>"],
  "projectIds": [<projectId>],
  "roleId": <3|4|5|6>,
  "seatTypeId": <1|2>,
  "groupRoleId": <6|7>,
  "teamIds": [<teamId>, ...]
}
```

- **⚠️ `seatTypeId` / `groupRoleId` / `roleId` 三者的相容組合是後端硬規則，錯了直接 400**（見 `mappings.md` 的相容表）：
  - **Full（seatTypeId=1）** → `groupRoleId=6`（Member）、`roleId` = 3/4/5（Manager/Internal/External）。
  - **Light（seatTypeId=2）** → **強制** `groupRoleId=7`、`roleId=6`（Light Member），不管 sheet Role 欄填什麼。
  - 別再無腦帶 `groupRoleId=6`——那只對 Full 成立。
- `emails` 是陣列；同一 project、同 role/seat/team 的多人可合併一次邀。role/seat/team 不同的人分開邀。
- 同一人若要進多個 project，可用 `projectIds` 一次帶多個（但 role/team 要一致）；不一致就分次。

> **防重複邀請的盲點**：`user_query_pending_invitation_users` **只列「系統已有帳號、待接受」的使用者，查不到對尚未註冊 email 送出的邀請**。所以邀請 timeout 後別用它判斷「是否已送出」（會回空陣列而誤判為沒送成功）。要防重複，改看 email 邀請紀錄，或直接以該次呼叫的 `isSucceed` 為準。

---

## 5. Equipment（group 層級）

### equipment_create_equipment

`POST /api/Equipment/CreateEquipment`（multipart form-data）

```json
{
  "Name": "<Equipment Name>",
  "TypeId": "<1|2>",
  "CategoryId": "<1-9>",
  "WorkingHours": <1-24>
}
```

（欄位名首字大寫；TypeId/CategoryId 以字串帶。）

---

## 6. Account Owner

### admin_console_modify_member_as_co_owner

`POST /api/AdminConsole/ModifyMemberAsCoOwner`

```json
{ "userId": <該成員的 userId> }
```

- 只吃 `userId`（整數），groupId 走 token。
- **此人必須已是 group 成員**才有 userId。剛用 email 邀請、尚未接受的人可能查不到 userId → 標「待成員接受後再設」，不要硬設。
- 取 userId：邀請後從成員清單（如 `member_query_all_members` / group 成員查詢）用 email 比對。
