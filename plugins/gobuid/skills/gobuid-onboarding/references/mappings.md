# name → id 對照與預設值

解析試算表時，把人類可讀的值換成 API 要的 id / 格式。除了 Budget Unit 之外，**全部都是寫死對照表，不需要 call API**。

## 目錄

- [Equipment Ownership → TypeId](#equipment-ownership--typeid)
- [Equipment Type → CategoryId（9 類）](#equipment-type--categoryid9-類)
- [Budget Category → CategoryId（7 類）](#budget-category--categoryid7-類)
- [Budget Unit → UnitId（15 預設 + 自建）](#budget-unit--unitid15-預設--自建)
- [Participant Role → roleId](#participant-role--roleid)
- [Seat Type → seatTypeId](#seat-type--seattypeid)
- [Group Role 預設](#group-role-預設)
- [Timesheet 星期陣列（星期日開頭）](#timesheet-星期陣列星期日開頭)
- [Project 固定預設 icon / color](#project-固定預設-icon--color)
- [日期格式](#日期格式)

---

## Equipment Ownership → TypeId

sheet 欄位「Ownership (Owned/Rental)」→ `TypeId`（整數）。

| Sheet 值 | TypeId |
| -------- | ------ |
| Owned    | 1      |
| Rental   | 2      |

沒填時預設 `1`（Owned）。

## Equipment Type → CategoryId（9 類）

sheet 欄位「Equipment Type」其實對應系統的 **CategoryId**（不是「種類」而是分類），寫死如下：

| Sheet 值                    | CategoryId |
| --------------------------- | ---------- |
| Aerial Working Platform     | 1          |
| Compaction Equipment        | 2          |
| Material Handling           | 3          |
| Light Equipment & Tools     | 4          |
| Site Preparation & Services | 5          |
| Lifting Equipment           | 6          |
| Earthmoving                 | 7          |
| Forklifts                   | 8          |
| Others                      | 9          |

`WorkingHours`：直接取 sheet「Working Hours/Day」的整數（1–24），沒填預設 8。

## Budget Category → CategoryId（7 類）

| Sheet 值                  | CategoryId |
| ------------------------- | ---------- |
| Concrete and Cement-based | 1          |
| Masonry                   | 2          |
| Structural                | 3          |
| Finishing                 | 4          |
| Plumbing and electrical   | 5          |
| Hardware and fasteners    | 6          |
| Others                    | 7          |

## Budget / Activity Unit → UnitId（15 預設 + 自建）

Budget 與 Activity **共用同一套單位**。系統預設單位（`isDefault=true`, `groupId=null`），可直接對照：

| name | UnitId |     | name | UnitId |
| ---- | ------ | --- | ---- | ------ |
| m    | 1      |     | ft³  | 9      |
| ft   | 2      |     | yd²  | 10     |
| in   | 3      |     | kg   | 11     |
| cm   | 4      |     | lb   | 12     |
| mm   | 5      |     | t    | 13     |
| m²   | 6      |     | no.  | 14     |
| ft²  | 7      |     | %    | 15     |
| m³   | 8      |     |      |        |

流程：

1. sheet 的 Unit 值先比對上表（注意 `m²`/`m³` 等上標字元）。
2. 不在表內 → 呼叫 `activity_query_activity_units_by_group_id`（帶 groupId）查該 group 是否已有同名單位，有就用它的 `activityUnitId`。
3. 還是沒有 → `activity_create_activity_unit`（groupId + name）建立，取回 `activityUnitId`。

> **`%` 特例**：UnitId `15` 代表百分比，選它時 `TotalQty` 必須 = 100（後端/前端都這樣處理）。若 sheet 填了別的數字，在計畫裡標出並改成 100。

## Participant Role → roleId

sheet 欄位「Role (Manager/Internal/External)」→ 專案角色 `roleId`（`ProjectRoleEnum`，寫死）：

| Sheet 值 | roleId |
| -------- | ------ |
| Manager  | 3      |
| Internal | 4      |
| External | 5      |

（Owner=1、CoOwner=2、LightMember=6 不在 CS 表選項內。）

## Seat Type → seatTypeId

sheet 欄位「Seat Type (Full/Light)」→ `seatTypeId`：

| Sheet 值 | seatTypeId |
| -------- | ---------- |
| Full     | 1          |
| Light    | 2          |

沒填時預設 `1`（Full）。選 Light 時，該成員的 `roleId` 通常應為 LightMember（6）；若 sheet 仍填 Manager/Internal/External，在計畫裡標出讓使用者決定。

## Group Role 預設

`groupRoleId`：邀請必填，CS 表沒有這欄，一律用預設 `6`（General）。

## Participant Team Tags → teamIds（每 group 查詢，id 不固定）

sheet「Team Tags」欄常見的、每個 group 預設就有的 team tag：

`Administrator`、`Client`、`Consultant`、`Document Controller`、`Project Coordinator`、`Project Manager`、`Safety Coordinator`、`Safety Supervisor`、`Safety Manager`、`Subcontractor`

> **⚠️ 與上面所有對照表最大的不同：team tag 的 id 不是固定的，每個 group 都不一樣。絕對不能寫死 teamId。** 上面只是名稱參考，不是 id 表。

流程（每個 group 都要重新查）：

1. `teams_get`（`GET /api/Teams`）撈出「當前 group」的所有 team，建立 name → id 對照。
2. sheet 的 Team Tags 以逗號分隔，逐個 name 去對照取 `id`，收集成 `teamIds` 陣列。
3. 對照不到的 name → `teams_post`（`{ "name": "<tag>" }`）建立，取回新 `id` 再加入陣列。

## 每週工時陣列（星期日開頭）

工時設定都在 **General 分頁**：設定區 `A1:C6`（含 Daily Start/End Time、Geofence Radius），每週工時表 `A8:C15`（Day / Working Hours / Non-working Day）。

每週工時表是 **週一→週日** 排列，但 API 陣列是 **星期日開頭**（index 0 = Sunday）。務必重排：

| API index | 星期      |
| --------- | --------- |
| 0         | Sunday    |
| 1         | Monday    |
| 2         | Tuesday   |
| 3         | Wednesday |
| 4         | Thursday  |
| 5         | Friday    |
| 6         | Saturday  |

- `weeklyUserAttendanceRecordHours`：`number[7]`，每格 = 該天工時（取 sheet「Working Hours」）。
- `weeklyEquipmentNonWorkingDays`：`boolean[7]`，`true` = 非工作日（sheet「Non-working Day」為 Yes）。
- `defaultWorkDayHours`：`{ startTime, endTime }`，格式 `"HH:mm:ss"`，取 sheet「Daily Start/End Time」（例如 `"09:00:00"` / `"18:00:00"`）。start 與 end 不可相同。

## Project 固定預設 icon / color

sheet 沒有這兩欄，一律套前端固定預設：

| 欄位  | 值            |
| ----- | ------------- |
| icon  | `trafficCone` |
| color | `#E9BF78`     |

## 日期格式

sheet 是 `YYYY/MM/DD`。送 API 前轉成 ISO 8601（UTC），例如 `2026/07/01` → `2026-07-01T00:00:00.000Z`。
