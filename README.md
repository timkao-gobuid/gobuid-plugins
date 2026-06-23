# GoBuid Plugin Marketplace

GoBuid 內部的 Claude Code plugin marketplace。提供 `gobuid` plugin —— 包含 MCP server（500+ API tools 操作 GoBuid 平台）以及一組封裝常見流程的 skills（表單、任務、專案、工項、群組）。

服務 endpoint：`https://mcp.gobuid.com/mcp`（HTTP MCP server，OAuth 登入）

---

## 給使用者：怎麼裝來用

server 已部署成共用服務，你只要把 client 指過去即可。第一次用到 tool 時會自動開瀏覽器，用 GoBuid 帳密登入完成 OAuth。

### Claude Code

**方式 A — 透過 marketplace（推薦，含版本管理）**

```
/plugin marketplace add https://gitlab.gobuid.com/ai-ops/gobuid-mcp-marketplace.git
/plugin install gobuid-mcp@gobuid
```

之後更新：`/plugin update gobuid-mcp@gobuid`

**方式 B — 手動填設定**（不想用 marketplace 的話）

編輯 `~/.claude/settings.json`：

```json
{
  "mcpServers": {
    "gobuid": { "type": "http", "url": "https://mcp.gobuid.com/mcp" }
  }
}
```

### Codex

Codex 不支援 Claude Code 的 marketplace，改編輯 `~/.codex/config.toml`，用 `mcp-remote` 橋接（會自動處理 OAuth）：

```toml
[mcp_servers.gobuid]
command = "npx"
args = ["-y", "mcp-remote", "https://mcp.gobuid.com/mcp"]
```

### 用法

連上後第一步先設定要操作的 group：

```
set_context — 設定 groupId
```

之後即可呼叫任意 API tool（例如 project_get_self_projects）。Token 到期會自動 refresh。

---

## Skills

plugin 內附下列 skills，封裝常見流程；裝好 plugin 後在 Claude Code 直接用 `/<skill-name>` 觸發，或用自然語言描述需求自動帶出。全部透過上述 MCP server 操作，使用前需先 `set_context` 設定 group。

| Skill                    | 用途                                                                                  |
| ------------------------ | ------------------------------------------------------------------------------------- |
| `gobuid-list-groups`     | 列出登入者所屬的 groups，並可切換 group context（groupId 來源）                       |
| `gobuid-list-projects`   | 列出某 group 底下使用者參與的 projects（取 projectId）                                |
| `gobuid-create-project`  | 建立 project；用 geocoding 綁定地址與座標，建立前確認 name / 起訖日 / 座標 / 地址齊全 |
| `gobuid-create-activity` | 在 project 底下建立工項（activity），含數量 + 單位、% 單位規則、起訖日把關            |
| `gobuid-task-crud`       | task / subtask 的建立、查詢、更新、刪除、狀態與排序（v2 API）                         |
| `gobuid-form-template`   | 從截圖 / PDF / xlsx / 文字建立或更新表單範本                                          |
| `gobuid-form-workflow`   | 設定表單審核流程（React Flow 節點圖）、team tag 與成員                                |

> 常見串接：`gobuid-list-groups` → `gobuid-list-projects` → `gobuid-create-project` / `gobuid-create-activity` / `gobuid-task-crud`；建表單則 `gobuid-form-template` → `gobuid-form-workflow`。

---

## 給維護者：部署 server

server 原始碼在 [ai-ops/gobuid-mcp](https://gitlab.gobuid.com/ai-ops/gobuid-mcp)，部署檔（Dockerfile / docker-compose.yml）也在那個 repo。

### 部署限制（重要）

- **只能單一 instance**：OAuth client 註冊、authorization code、進行中的 MCP session 都存在記憶體，不可多 replica / autoscale。
- **需要持久化 volume**：token store 與 device id 寫到 `/data`。device id 若重生會導致 GoBuid refresh token 失效（device mismatch），全部人要重登 —— compose 已掛 `gobuid-mcp-data` volume 處理。
- **`BASE_URL` 必須是對外 URL**（`https://mcp.gobuid.com`），它是 OAuth issuer，設錯認證會失敗。

### 步驟

```bash
git clone https://gitlab.gobuid.com/ai-ops/gobuid-mcp.git
cd gobuid-mcp
docker compose up -d --build   # 跑在 127.0.0.1:3939
```

前面放反向代理做 TLS（DNS 先把 mcp.gobuid.com 指到這台機器）。Caddy 最省事：

```caddyfile
mcp.gobuid.com {
    reverse_proxy 127.0.0.1:3939
}
```

> 用 nginx 的話記得 `proxy_buffering off;`，否則 streaming 會卡。

### 驗證

```bash
curl https://mcp.gobuid.com/.well-known/oauth-authorization-server
```

有回 OAuth metadata 就代表 server + TLS 都通了。

### 發新版

server 更新後，記得同步調 [gobuid-mcp](https://gitlab.gobuid.com/ai-ops/gobuid-mcp) 的 `.claude-plugin/plugin.json` 與本 repo `marketplace.json` 的 `version`，使用者 `/plugin update` 才會抓到。
