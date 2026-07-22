# 分散式 MCP Server 開發指南

> **版本**: 1.3.0
> **建立日期**: 2026-07-19
> **狀態**: 已實作
> **適用對象**：想把自己系統的功能開放給 Venus AI 對話模組呼叫的**外部系統開發者**（本文件假設讀者對 Venus 內部架構完全不熟悉）
> **架構設計文件**（Venus 內部實作細節）：[`DISTRIBUTED_MCP_DESIGN.md`](./DISTRIBUTED_MCP_DESIGN.md)
> **機器可讀規格**：`GET /api/ai/mcp-servers/dev-spec`（本文件的內容有一份對應的 JSON Schema 版本，供你自己的 AI 開發工具直接讀取比對，見 §9）

---

## 1. 這是什麼、我為什麼要接？

如果你的系統想讓 Venus AI（公司內部的 AI 對話助理）能夠呼叫你系統的功能——查資料、觸發動作、產生報表——你有兩個選擇：

1. **請 Venus 團隊幫你寫**（內建工具）：你要花時間跟 Venus 工程師解釋業務邏輯，之後每次改動都要等 Venus 排期上版
2. **你自己接分散式 MCP**（本文件的主題）：你自己開發、自己部署、自己維護。你的系統對外提供一個標準化的 HTTP 端點，Venus 呼叫這個端點來執行你的工具。改動不需要 Venus 上版，你自己重新註冊 manifest 就生效

分散式 MCP 的核心承諾：**你最懂自己系統的業務邏輯，中介處理就該由你來寫**，Venus 只負責「知道有你這個服務、知道你有哪些工具、把 AI 的呼叫轉給你、把你的回應轉給 AI」。

---

## 2. 五分鐘快速上手（Quick Start）

### 步驟一：請 Venus 管理員幫你建立服務紀錄

**你不能自己申請**——這是刻意設計（見 §8 安全模型）。請 Venus 系統管理員在「AI 對話模組管理後台 → 術式管理 → 分散式 MCP 服務」頁面幫你「註冊新服務」，你只需要提供：

| 欄位 | 範例 | 說明 |
|---|---|---|
| `serverKey` | `hmrb` | 全域唯一識別碼，小寫字母開頭，只能含小寫字母/數字/`-`/`_` |
| `displayName` | `HMRB ReguCheck` | 給管理員看的顯示名稱 |
| `description` | `法規檢核與 BOM 管理系統` | 一句話說明你的系統是做什麼的 |
| `baseUrl` | `http://10.x.x.x:8520` | Venus 呼叫你系統時使用的完整 base URL（內網位址即可） |
| `mcpEndpoint` | `/api/mcp`（預設值，可不填） | JSON-RPC 端點相對路徑 |
| `ownerEmpid` | `1605A0012` | 你的員工編號（負責人）。本系統一律用具身分關聯性的 empid 追責，不接受自由文字聯絡方式；Venus 管理員會在後台從員工名冊搜尋並指派給你，出問題時才追得到人 |

管理員送出後，會拿到一個**明文顯示一次的 `shared_secret`**——這是你之後所有出站呼叫（註冊、心跳）都要用的憑證，請安全存放（如放進你系統的環境變數 `.env`），Venus 之後查詢都只會顯示遮罩後的前綴，不會再給你看明文。

### 步驟二：在你的系統裡實作 JSON-RPC 端點

你需要對外提供一個 HTTP POST 端點（步驟一填的 `mcpEndpoint`，慣例 `/api/mcp`），能處理三個方法：`initialize` / `tools/list` / `tools/call`（詳見 §5）。

### 步驟三：開機時向 Venus 註冊你的工具清單

呼叫 `POST http://<venus-host>/api/ai/mcp-servers/<你的serverKey>/register`，帶上你的完整工具 manifest（詳見 §4）。

### 步驟四：定期心跳

每 5 分鐘呼叫一次 `POST .../heartbeat`，讓 Venus 知道你還活著（純顯示信號，非強制，但建議做）。

以上四步完成後，你的工具就會出現在 Venus AI 的工具清單中，可以被 AI 對話呼叫了。

---

## 3. 全自動化設計理念（懶人包）

好的分散式 MCP Server 應該做到：**你的系統開機時自動嘗試註冊，Venus 還沒啟動或網路還沒通都不會讓你的系統卡死**。參考 HMRB 的實作模式（`server/mcp/venusRegistrationService.js`）：

```javascript
// 概念示意（非逐字複製，請依你自己的框架調整）
async function startVenusRegistration() {
  if (!process.env.YOUR_SHARED_SECRET) {
    console.log('[MCP] 未設定 shared secret，跳過向 Venus 註冊');
    return; // 安全跳過，不影響你系統其他功能
  }

  let attempt = 0;
  const backoffSeconds = [5, 15, 30, 60, 120]; // 之後固定用最後一個值重試

  async function tryRegister() {
    try {
      await axios.post(`${VENUS_BASE_URL}/api/ai/mcp-servers/${SERVER_KEY}/register`, manifest, {
        headers: { Authorization: `Bearer ${process.env.YOUR_SHARED_SECRET}` },
      });
      console.log('[MCP] 註冊成功');
      startHeartbeat(); // 註冊成功才開始心跳排程
    } catch (err) {
      const delay = backoffSeconds[Math.min(attempt, backoffSeconds.length - 1)];
      attempt++;
      console.warn(`[MCP] 註冊失敗，${delay}s 後重試: ${err.message}`);
      setTimeout(tryRegister, delay * 1000);
    }
  }

  tryRegister(); // 不 await，不阻擋你自己的 app.listen()
}
```

**關鍵原則**：

- 沒設定憑證 → 安全跳過，不報錯、不阻擋你系統其他功能啟動
- Venus 還沒啟動 / 網路還沒通 → 指數退避重試，不放棄、不丟未捕捉例外
- 註冊本身是**冪等**的——重新呼叫、manifest 有變更時重新呼叫，都是安全的
- **你完全不需要請 Venus 管理員手動把你新註冊的工具「掛到主要對話領域」**：Venus 首次看到一個全新的 `tool_key` 時，會自動把它掛載到 primary agent（Venus 全能力主助理，也就是使用者一般對話會用到的主要回應領域），只要 `toolMode` 不是 `agent_only` 就會自動生效，開箱即用。管理員之後仍可在後台取消勾選（取消後不會被復活），但你不需要主動要求這一步

---

## 4. Manifest 完整參考

`register` 呼叫的請求 body 就是這份 manifest（`tools/list` 回應的 `result.tools` 也是同樣的 `tools[]` 陣列形狀）：

```json
{
  "protocolVersion": "tci-mcp/1.0",
  "server": {
    "name": "HMRB ReguCheck",
    "version": "2.3.0",
    "description": "法規檢核與 BOM 管理系統",
    "baseUrl": "http://10.x.x.x:8520",
    "mcpEndpoint": "/api/mcp",
    "docsUrl": "http://10.x.x.x:8520/docs"
  },
  "tools": [
    {
      "name": "bom_query",
      "displayName": "BOM Query",
      "displayNameZh": "BOM 查詢",
      "triggerDescription": "查詢指定料號的 BOM（用料清單）結構，包含子件用量與單位。當使用者問到某個料號的配方組成、用了哪些原料時使用。",
      "usageGuide": "參數 matnr 必須是完整料號字串（不可只給部分關鍵字）。若回傳 errorCategory='not_found'，代表該料號在 SAP 中查無 BOM，請誠實告知使用者，不要重試。",
      "inputSchema": {
        "type": "object",
        "properties": {
          "matnr": { "type": "string", "description": "料號，例如 1234567890" }
        },
        "required": ["matnr"]
      },
      "passMode": "search",
      "toolMode": "all",
      "platformFilter": "all",
      "requireVerification": false,
      "adminOnly": false,
      "recommendedTimeoutSeconds": 30
    }
  ]
}
```

### 4.1 欄位速查表

| 欄位 | 必填 | 型別 | 說明 |
|---|---|---|---|
| `name` | ✅ | string，`^[a-z][a-z0-9_]*$` | **全域唯一**的工具識別碼（snake_case），跨所有 MCP Server 不可重複 |
| `displayName` | 建議填 | string | 管理 UI 顯示的英文名稱 |
| `displayNameZh` | 選填 | string | 中文顯示名稱 |
| `triggerDescription` | ✅ | string | **AI 判斷「何時該呼叫這支工具」的唯一依據**，寫得越精準，AI 越不會誤用或漏用 |
| `usageGuide` | 強烈建議填 | string | 教 AI「呼叫失敗時怎麼自我修正」，見 §7 |
| `inputSchema` | 選填（無參數可省略） | object | OpenAI function-calling 風格的 JSON Schema |
| `passMode` | 選填，預設 `search` | `always` \| `search` | `always` 每輪對話都注入（貴，只留給幾乎每次都要用的工具）；`search` 依語意搜尋命中才注入（預設，建議） |
| `toolMode` | 選填，預設 `all` | `all` \| `general_only` \| `agent_only` | 限制此工具只在一般對話 / 個人代理人對話中出現 |
| `platformFilter` | 選填，預設 `all` | `all` \| `web_only` \| `teams_only` | 限制此工具只在特定平台可用 |
| `requireVerification` | 選填，預設 `false` | boolean | `true` = 執行前需使用者在對話中按下確認卡片（用於不可逆/高風險操作） |
| `adminOnly` | 選填，預設 `false` | boolean | `true` = Venus 會在派發前強制檢查呼叫者是否為系統管理員或 AI 模組管理員 |
| `recommendedTimeoutSeconds` | 選填 | 正整數 | 若你的工具需要較長執行時間（如報表生成），設定此值可覆蓋 Venus 預設逾時 |

> **改欄位安全嗎？** 完全安全。改完 `triggerDescription` / `usageGuide` / `inputSchema` / `passMode` / `toolMode` / `platformFilter` / `requireVerification` 等 manifest 欄位後，等你下次排程重新註冊，或請 Venus 管理員點「重新同步」即可生效——這些欄位**每次**註冊/同步都會套用你 manifest 的最新值（2026-07-22 起，不再有「只在首次建立生效」的限制）。唯一 Venus 管理員專屬、你的 manifest 永遠不會覆寫的欄位是 `allowed_permissions` / `allowed_empids`（你的 manifest 本來就沒有對應欄位可填）。詳見 [`DISTRIBUTED_MCP_DESIGN.md §5`](./DISTRIBUTED_MCP_DESIGN.md#5-欄位所有權模型layer-ownership)。

> **`name` 的合法字元比 `serverKey` 更嚴格**：`serverKey`（§2 步驟一）允許 `-`（如 `my-service`），但 **`name`（工具識別碼）不允許 `-`，只能用 `_`**（正則 `^[a-z][a-z0-9_]*$`）。這是最常見的註冊 400 錯誤來源之一——`bom-query` 會被拒絕，必須寫成 `bom_query`。

### 4.2 Manifest 驗證規則與常見踩雷

`register` 呼叫會先跑一層嚴格驗證（`services/ai/mcpManifestValidator.js`），驗證失敗回 `400`，body 形狀：

```json
{
  "success": false,
  "error": "Manifest validation failed",
  "errorCategory": "validation_failed",
  "nextActionForAi": "Fix the listed fields and re-register",
  "errors": [
    { "field": "tools[bom-query].name", "reason": "name must be a lowercase snake_case identifier", "suggestion": "e.g. \"bom_query\" (must match ^[a-z][a-z0-9_]*$)" }
  ],
  "warnings": [
    { "field": "tools[bom_query].usageGuide", "reason": "usageGuide is empty", "suggestion": "The AI relies on usageGuide for self-correction on errors; strongly recommended for any tool with parameters" }
  ]
}
```

- **`errors` 非空 → 整份 manifest 被拒絕**（不會部分生效），修正 `errors` 陣列列出的每一項後重新送出
- **`warnings` 不會擋註冊**（如 `usageGuide` 留空、`tools` 陣列為空），但強烈建議照建議修正——`usageGuide` 留空等於讓 AI 在你的工具失敗時盲猜怎麼自我修正
- 型別要求是**嚴格型別**，不是字串："true" 這種字串不會被當成 `true`：

| 欄位 | 驗證規則 |
|---|---|
| `tools[].name` | `^[a-z][a-z0-9_]*$`（同一份 manifest 內也不可重複） |
| `tools[].triggerDescription` | 必填、去除空白後不可為空字串 |
| `tools[].passMode` | 必須是 `"always"` 或 `"search"`（其他字串會被拒絕） |
| `tools[].toolMode` | 必須是 `"all"` / `"general_only"` / `"agent_only"` |
| `tools[].platformFilter` | 必須是 `"all"` / `"web_only"` / `"teams_only"` |
| `tools[].requireVerification` / `adminOnly` | 必須是 JSON boolean（`true`/`false`），**不可是字串** `"true"` |
| `tools[].recommendedTimeoutSeconds` | 必須是正整數（`> 0`），浮點數或 0 會被拒絕 |
| `tools[].inputSchema` | 選填；若提供，`type` 必須是 `"object"`，`properties` 必須是物件，`required` 必須是陣列 |
| `server.baseUrl` | 必須是完整 `http://` 或 `https://` URL |
| `server.mcpEndpoint` | 若提供，必須以 `/` 開頭 |

---

## 5. 實作你的 JSON-RPC 端點

你的端點是一個標準 JSON-RPC 2.0 dispatcher。以下是三個方法的請求/回應範例（基於 HMRB 的實際實作 `server/mcp/mcpRoutes.js`）：

### 5.1 驗證進站請求

Venus 對你的端點發出 `tools/call`（以及 `initialize`/`tools/list`）時，一律帶下列 HTTP headers（來源：`services/ai/mcpClientService.js` `buildHeaders`）：

| Header | 值 | 說明 |
|---|---|---|
| `Content-Type` | `application/json` | — |
| `MCP-Protocol-Version` | `tci-mcp/1.0` | 目前協定版本常數，未來若升版會變動 |
| `X-API-Key` | 你的 `shared_secret` | **你必須驗證此值**，見下方程式碼 |
| `X-Caller` | `venus-ai:<empid>` 或 `venus-ai:anonymous` | 純稽核用途，選填參考——目前 HMRB 的稽核記錄改用 `params.context.empid`，不強制你解析此 header |

每個請求都要驗證：

```javascript
// 概念示意
const protocolVersion = req.headers['mcp-protocol-version'];
const apiKey = req.headers['x-api-key'];
if (apiKey !== process.env.YOUR_SHARED_SECRET) {
  return res.status(401).json({ jsonrpc: '2.0', id: req.body.id, error: { code: -32001, message: 'Invalid API key' } });
}
```

### 5.2 `initialize`

```json
// Request
{ "jsonrpc": "2.0", "id": "venus-abc123", "method": "initialize", "params": { "protocolVersion": "tci-mcp/1.0" } }
// Response
{ "jsonrpc": "2.0", "id": "venus-abc123", "result": { "protocolVersion": "tci-mcp/1.0", "serverInfo": { "name": "HMRB ReguCheck", "version": "2.3.0" }, "capabilities": { "tools": true } } }
```

### 5.3 `tools/list`

```json
// Request
{ "jsonrpc": "2.0", "id": "venus-abc124", "method": "tools/list", "params": {} }
// Response
{ "jsonrpc": "2.0", "id": "venus-abc124", "result": { "tools": [ /* 與 §4 manifest 的 tools[] 完全同形狀 */ ] } }
```

建議你的實作把 manifest 集中存在單一權威清單（如 HMRB 的 `server/mcp/toolManifest.js`），`register` 呼叫、`tools/list` 回應都從這個清單取值，避免兩處各存一份而漂移。

### 5.4 `tools/call`（核心）

```json
// Request
{
  "jsonrpc": "2.0", "id": "venus-abc125", "method": "tools/call",
  "params": {
    "name": "bom_query",
    "arguments": { "matnr": "1234567890" },
    "context": {
      "empid": "1605A0012", "english_name": "Alice Wang", "chinese_name": "王小明",
      "dep_name": "研發部", "preferred_language": "zh-TW",
      "platform": "web", "session_id": "sess_xxxxx"
    }
  }
}
```

你的 dispatcher 職責：

1. 依 `params.name` 找到對應的 handler
2. **驗證 `params.arguments` 是否符合你自己宣告的 `inputSchema`**（Venus 不會替你驗證必填參數！這是你 handler 自己的責任）
3. 執行業務邏輯
4. 回傳符合 §6 結果契約的 `result`

```json
// Response（業務成功範例）
{ "jsonrpc": "2.0", "id": "venus-abc125", "result": { "success": true, "matnr": "1234567890", "components": [ { "matnr": "9876543210", "qty": 2.5, "unit": "KG" } ] } }
```

### 5.5 `context` 欄位精確定義

`params.context` 的每個欄位都是**字串**，來源與 fallback 規則如下（來源：`services/ai/aiToolsManager.js` `_buildMcpContext`）——**所有欄位在無值時都是空字串 `""`，不會是 `null`／`undefined`**，你的 handler 可以放心用字串方法處理，不需要額外判斷 null：

| 欄位 | 來源 | 空值時 fallback |
|---|---|---|
| `empid` | 使用者工號 | `""`（代表系統/背景呼叫，無明確使用者） |
| `english_name` | 使用者英文名 | 退回 `empid`；兩者皆無則 `""` |
| `chinese_name` | 使用者中文名 | `""` |
| `dep_name` | 使用者部門名稱 | `""` |
| `preferred_language` | 使用者個人偏好語系設定（如 `zh-TW`/`en-US`） | `""`（**不是** `en-US`——代表「使用者沒設定偏好」，你的系統若需要一個明確預設值，請自行 fallback 到 `en-US`，對齊全系統 i18n 慣例，不要 fallback 到中文） |
| `platform` | 呼叫來源 | **只會是 `"web"` 或 `"teams"` 這兩個字串之一，沒有第三種值**（即使是透過 API 金鑰觸發的個人代理人對話，這裡也會落在 `"web"`——這不是「使用者實際用網頁」的保證，只是預設分支） |
| `session_id` | 對話 session 識別碼 | `""`（代表無 session 上下文的背景呼叫） |

> **逾時樓層（agent 對話會拿到更長的執行時間）**：Venus 解析你這支工具的逾時秒數規則是：先用你 manifest 的 `recommendedTimeoutSeconds`（若為正整數），否則用全域預設 30 秒；**若這次呼叫來自個人代理人對話情境，Venus 會再取 `max(上述秒數, 180)`**（180 為預設值，Venus 管理員可調整）。這個判斷依 Venus 內部的呼叫來源旗標，不由你的 manifest 控制，`context` 物件裡也看不到這個旗標。**建議**：不要仰賴這個樓層機制去猜你的工具會拿到多長時間——如果你的工具本身就需要長時間執行（如報表生成、外部系統呼叫），直接把 `recommendedTimeoutSeconds` 設高一點最保險。

### 5.6 業務權限標籤查詢（Permission Tags）— 需要資料範圍篩選時才用

> **先分清楚兩個完全不同的問題**：
> 1. 「這個呼叫者能不能叫我這支工具？」——這一層 Venus **已經**在呼叫你之前處理完了（§8 `adminOnly`，或一般工具的 `allowed_permissions`/`allowed_empids`/`tool_mode` 白名單）。你收到 `tools/call` 請求本身就代表這一層已經通過，**你不需要、也拿不到**「這個人是不是管理員」這種標籤——`context` 物件刻意不含這個資訊（§5.5 的資料邊界原則）。
> 2. 「這個呼叫者能不能看到*這一筆*資料？」——例如你的工具是「查詢原料文件」，**大部分文件所有人都能查**，但**少數特別敏感的文件只有 RA（法規遵循）或 RD（研發）身分才能看內容**。這是**同一支工具、依呼叫者身分回傳不同內容**的場景，Venus 沒辦法幫你做這個判斷（Venus 根本不知道你的業務資料哪些敏感），**必須由你自己的 handler 判斷**。

本節只解決第 2 種問題。如果你的工具回傳的資料對所有能呼叫它的人都一視同仁，不需要看本節。

#### 5.6.1 你已經有的線索：`context.empid`

`tools/call` 的 `context.empid`（§5.5）已經足夠當作查詢 key，你不需要 Venus 額外傳遞任何權限欄位進 `context`——原因見上方兩層區分：權限標籤是「有需要才查」的資訊，不該無條件塞進每一次呼叫的 context（多數工具根本用不到）。

#### 5.6.2 查詢端點

```
POST /api/ai/mcp-servers/employee-permissions
Authorization: Bearer <shared_secret>   # 與你 register/heartbeat 用的同一把金鑰，不需要另外申請
Content-Type: application/json

{ "empid": "1605A0012" }
```

成功回應：

```json
{ "success": true, "empid": "1605A0012", "permissions": ["RA", "RD"] }
```

`permissions` 是**這名員工目前擁有的業務權限標籤**，依組織架構即時計算（非靜態欄位），可能是空陣列 `[]`（代表「沒有任何特殊標籤」，這是正常結果，不是錯誤）：

| 標籤 | 業務意涵 |
|---|---|
| `PM` | 專案管理處及其下屬組織（含上級主管） |
| `BD` | 銷售/業務組織（含全球業務中心主管鏈） |
| `RD` | 研發設計中心及其下屬組織（含上級主管） |
| `AI` | AI 統籌服務組／戰略數據中心及其下屬組織（含上級主管） |
| `RA` | 法規遵循相關組織及其下屬組織（含上級主管） |
| `PD` | 產品設計相關組織及其下屬組織（含上級主管） |
| `PUR` | 全球策略採購處及其下屬組織（含上級主管） |

一名員工可以同時擁有 0 個、1 個或多個標籤（跨部門兼職身分會合併計算）。**這份標籤詞彙是全公司通用的組織權限概念，不是任何特定 MCP Server 的專屬設計**——不管你的系統是什麼業務領域，只要需要「依組織身分決定資料範圍」，都用這套標籤。

失敗回應：

| HTTP | `errorCategory` | 情境 | 你該怎麼處理 |
|---|---|---|---|
| 401 | `unauthorized` | Bearer token 錯誤或缺失 | 檢查你送出的 `Authorization` header 是否為你註冊時拿到的 `shared_secret` |
| 400 | `validation_failed` | 缺少或空白 `empid` | 檢查你是否正確從 `context.empid` 取值 |
| 503 | `service_unavailable` | Venus 端組織架構查詢暫時失敗（上游 HR 系統逾時等） | 重試一次；仍失敗就**視為該員工沒有任何標籤（fail-closed）**，不要無限重試、也不要預設「查不到就當作有權限」 |

#### 5.6.3 使用建議

- **Fail-closed**：查詢失敗或逾時時，把該員工當成「沒有任何標籤」處理，寧可讓有權限的人多按一次重試，也不要讓沒權限的人意外看到敏感資料。
- **做你自己的短 TTL 快取**：這是即時查組織架構的動態運算，不是免費的靜態查表。建議在你自己的服務內用 60 秒左右的記憶體快取 + in-flight 去重（同一 `empid` 短時間內重複查詢時，共用同一個進行中的 Promise，不要重複發送請求）——這不是 Venus 端的責任，是你自己服務該做的節流。
- **只查你需要判斷的那一刻**，不要在 list 型回應的每一筆資料迴圈裡各查一次——查一次、快取起來、迴圈內重複使用同一份結果。
- **這與 `adminOnly` 是互補而非取代關係**：如果你的整支工具本身就該限定管理員才能呼叫（而不是「工具內某些資料要限定」），請直接在 manifest 宣告 `adminOnly: true`（§8），不要繞道用本端點自己重新發明一套「呼叫者是不是管理員」的判斷。

---

## 6. 結果契約（Result Contract）

你的 `tools/call` 回應必須是下列三種形狀之一：

### 6.1 業務成功

```json
{ "success": true, "...你的業務欄位": "..." }
```

如果你的工具需要回傳檔案（PDF 報表、Excel 匯出等），用 `files[]`：

```json
{
  "success": true,
  "summary": "已產生法規檢核報告",
  "files": [
    { "filename": "regulatory_review_20260719.pdf", "mime_type": "application/pdf", "base64": "<base64 編碼內容>" }
  ]
}
```

Venus 會自動把 `files[]` 落地存檔並轉成使用者可下載的檔案卡片，**你不需要處理檔案儲存、過期、URL 產生**。

> **⚠️ 檔案大小上限：每個檔案（解碼後）50MB，超過會被靜默丟棄！** Venus 落地存檔用的是既有附件服務（`services/ai/programmaticToolFileExchange.js` `persistInboundFile`，預設上限 50MB），超過上限的檔案**不會**讓整個 `tools/call` 失敗，也**不會**在回應中出現任何錯誤訊息給 AI——只有 Venus 伺服器主控台會印一行 `console.warn`，AI 跟使用者看到的就是「回應成功但少了一個檔案」。如果你的工具可能產生大檔案（如高解析度掃描 PDF、大型 Excel 匯出），**務必自行控管輸出檔案大小在 50MB 以內**，或在你的 `success` 回應的文字說明中不要承諾「已附上檔案」以免與實際結果不一致。

### 6.2 業務失敗

```json
{
  "success": false,
  "error": "Material 1234567890 not found in SAP",
  "errorCategory": "not_found",
  "nextActionForAi": "Tell the user this material number does not exist; do not retry.",
  "validValues": null,
  "suggestion": null
}
```

`errorCategory` 是**你自己定義的詞彙**，選幾個清楚的分類即可，例如：

| errorCategory 範例 | 適用場景 | AI 該怎麼自我修正 |
|---|---|---|
| `validation_failed` | 參數格式不對、缺必填 | 修正參數後重試 |
| `invalid_region`（或任何 `invalid_xxx`） | 參數值不在合法集合內 | 附 `validValues` + `suggestion`，AI 直接挑一個重試 |
| `not_found` | 識別碼查無資料 | 誠實告知使用者，不要重試 |
| `business_rule_violation` | 業務規則拒絕（如庫存不足） | 誠實告知使用者具體原因 |
| `service_unavailable` | 你的後端服務或依賴的外部系統掛了 | 告知使用者稍後再試，AI 不應無限重試 |

> **重要**：`errorCategory` + `nextActionForAi` 是 AI 唯一能拿到的「怎麼修正」線索。寫得含糊（如只回 `"error": "失敗"`）等於讓 AI 盲猜，使用者體感就是 AI 卡住或亂猜。詳見 [`code-review-preferences.mdc §AI 工具的可操作性必檢項目`](../../../.cursor/rules/code-review-preferences.mdc)。

### 6.3 協定失敗（少用，僅限協定層問題）

```json
{ "jsonrpc": "2.0", "id": "venus-abc125", "error": { "code": -32601, "message": "Method not found" } }
```

**只在**「未知方法」「請求格式錯誤（連 JSON-RPC 信封都不對）」這類協定層問題使用。**正常的業務驗證失敗一律用 §6.2 的 `success:false` 形狀**，不要拿協定錯誤表達業務錯誤——Venus 會把協定失敗歸類為 `errorCategory:'protocol_error'`，AI 收到的自我修正線索會比業務失敗少得多。

---

## 7. `usageGuide` 撰寫建議

`usageGuide` 是 AI 唯一能「讀」到的除錯手冊，寫得好可以讓 AI 自己修正錯誤，不需要反問使用者或卡住。範例：

```
本工具查詢料號的 BOM 結構。

參數：
- matnr（必填）：完整料號字串，不可只給部分關鍵字。若不確定料號，請先用其他查詢工具搜尋。

錯誤自我修正：
- errorCategory='not_found'：該料號在 SAP 中查無 BOM，代表這個料號真的沒有配方資料。誠實告知使用者，不要重試。
- errorCategory='validation_failed'：matnr 格式不對（通常是給了關鍵字而非完整料號）。請先用搜尋工具取得完整料號後再呼叫本工具。
- errorCategory='service_unavailable'：SAP 連線暫時中斷。可以重試一次；若仍失敗，告知使用者稍後再試。
```

原則（詳見 [`project-conventions.mdc §8`](../../../.cursor/rules/project-conventions.mdc)）：寫**邏輯化的原則**而非窮舉所有情境；區分清楚「參數你自己填錯了該自己改」vs「資料本來就不存在該誠實告知」這兩種完全不同的失敗性質。

---

## 8. 安全模型摘要

- 你的 `shared_secret` 由 Venus 管理員核發，**雙向使用**：Venus 出站呼叫你時帶 `X-API-Key`，你入站呼叫 Venus 的 `register`/`heartbeat` 時帶 `Authorization: Bearer`
- 明文密鑰只會在 Venus 管理員建立/重生金鑰的那次操作顯示一次，之後查詢一律遮罩
- 本協定的信任模型是「內網、已授權的內部開發者」，**沒有** mTLS、沒有細粒度的 per-tool 簽名，密鑰洩漏等同該服務名下所有工具的完整控制權——請把 `shared_secret` 當成一般機密憑證妥善保管（環境變數，不要 commit 進版控）
- 若你的工具是高風險操作（如全域環境切換），在 manifest 宣告 `adminOnly: true`，Venus 會在派發前強制檢查呼叫者權限；但**你自己的 handler 仍應做防禦性檢查**，不要只依賴 Venus 這一層
  - 精確檢查邏輯（`services/ai/aiToolsManager.js` `_executeMcpTool`）：`isSystemAdmin(empid) || isModuleAdmin(empid, 'ai-chat')`——也就是 Venus 系統管理員或「AI 對話模組管理員」皆可通過，不是只有系統管理員一人
  - 被擋下時，AI 收到的回應是 `{ success:false, errorCategory:'forbidden', error:'Tool "..." is restricted to system or module administrators' }`——這是給 AI 誠實告知使用者用的訊息，不代表你的服務或工具本身有問題

---

## 9. 用 AI 輔助開發：機器可讀規格端點

如果你想讓你自己的 AI 開發工具（如 Cursor、Copilot）直接理解本協定規格，呼叫：

```
GET http://<venus-host>/api/ai/mcp-servers/dev-spec
Authorization: Bearer <你的shared_secret>   # 或帶已登入 Venus 的 session cookie
```

回應是一份完整的機器可讀 JSON（含 manifest 的 JSON Schema draft-07、三個方法的請求/回應形狀、context object 欄位表、逾時解析規則、結果契約、驗證錯誤格式），內容與本文件 §4-§8 的核心技術細節一一對應，但格式更適合貼進 AI 工具的 context 直接生成程式碼。此端點刻意放寬認證（已註冊服務的 Bearer token 或任何已登入使用者的 session 皆可），因為內容純粹是文件性質、不含機密資料——方便你在正式申請服務之前就先讓 AI 幫你看規格。

---

## 10. 服務生命週期：下架、停用、金鑰重生、刪除

這些操作大多由 Venus 管理員在後台按鈕觸發，但會直接影響你系統觀察到的行為，建議先讀過再上線：

| 操作 | 誰觸發 | 對你的影響 |
|---|---|---|
| **從 manifest 移除一支工具後重新註冊** | 你自己（改 manifest、重新呼叫 `register`） | 該工具在 Venus 端被標記**停用（`is_active=0`），不會被刪除**（保留稽核歷史）。停用的工具立刻從 AI 可呼叫的工具清單消失。若你之後把這支工具重新加回 manifest 再註冊，會被視為「既有工具重新啟用」，`is_active` 恢復為 1，且管理員先前對 `pass_mode`/`tool_mode` 等欄位的調整**不會**因為這次停用又復活而重置 |
| **管理員按「重新同步」** | Venus 管理員 | 主動呼叫你的 `tools/list`，走與開機自動註冊完全相同的 upsert 邏輯（不需要你額外處理任何差異） |
| **管理員按「測試連線」** | Venus 管理員 | 只呼叫你的 `initialize`，純健康檢查，**不會**寫入任何 DB 狀態、不影響你已註冊的工具 |
| **管理員重生 `shared_secret`** | Venus 管理員 | **舊金鑰立即失效**——如果你的服務還在用舊金鑰做心跳/註冊，會開始收到 `401`。新金鑰只在該次操作明文顯示一次，之後查詢一律遮罩，請立刻更新你系統的環境變數 |
| **管理員停用整個服務**（`isActive: false`） | Venus 管理員 | 你的 `register`/`heartbeat` 呼叫會收到 `403 forbidden`；**你已註冊的工具不會立刻從 AI 工具清單消失**（工具本身的 `is_active` 沒被連動改變），但任何呼叫都會在派發前被擋下，AI 收到 `errorCategory:'service_unavailable'` 的「暫時無法使用，最多重試一次」訊息，不會誤導使用者以為是你的服務出錯 |
| **管理員刪除整個服務** | Venus 管理員 | 刪除的是服務註冊本身（含金鑰，立即失效）。你的工具**不會**被清空——一律停用（`is_active=0`），且它們原本掛載在哪些 Agent／領域底下的綁定**完全保留**（2026-07-22 起改為軟刪除，見下方說明）。之後你的系統若還在心跳/重新註冊，會收到 `404`（`MCP server "xxx" is not registered on Venus`），代表需要請管理員重新走一次 §2 步驟一，拿新的 `shared_secret`，用新服務重新註冊同一批 `tool_key`——這時 Venus 會自動偵測到這些 `tool_key` 原本屬於一個已被刪除的服務，把它們**原地回收**（沿用原本的內部識別碼），你完全不需要、也**不應該**改用新的 `tool.name`，否則就真的變成全新工具、原本的 Agent 掛載與使用統計都接不回來 |

---

## 11. 疑難排解

| 現象 | 可能原因 | 排查方法 |
|---|---|---|
| 註冊回 `401` | `shared_secret` 錯誤或還沒被 Venus 管理員建立 | 確認 `serverKey` 拼字正確；請管理員確認服務紀錄已建立且金鑰未被重生過 |
| 註冊回 `400 validation_failed` | manifest 格式不符 §4.2 驗證規則 | 檢查回應中的逐項 `{ field, reason, suggestion }` 清單；常見錯誤：`name` 不是 snake_case（誤用 `-`）、`triggerDescription` 空白、`requireVerification`/`adminOnly` 傳成字串而非 boolean |
| 註冊仍回 `200`，但回應 `conflicts` 陣列非空，`reason` 是 `owned_by_another_mcp_server` 或 `internal_tool_not_in_takeover_allowlist` | 你宣告的 `tool.name` 與其他服務或既有 Venus 內建工具的 `tool_key` 撞名——該支工具會被跳過（不新增、不覆寫），但你 manifest 內其他沒衝突的工具仍會正常註冊成功 | 換一個全域唯一的名稱；工具名跨所有 MCP Server 共享同一個命名空間 |
| 註冊回 `200`，`conflicts[]` 出現 `reason: "owned_by_different_developer"` | 你的 `tool.name` 撞到一個既有的、由 Venus 前端手動建立的「程式型工具」，但那支工具的建立者跟你這個 MCP Server 登記的負責人（`owner_empid`）不是同一人——Venus 出於安全考量不會自動幫你把別人的工具吃下來，即使你的 shared_secret 認證通過也一樣 | 兩個選擇：(1) 換一個不撞名的 `tool.name`；(2) 若這確實是同一支功能、你想接手它的 MCP 化，請 Venus 管理員在「分散式 MCP 服務」面板的衝突清單裡點「確認接管」（回應中的 `existingOwnerName` 會告訴你目前是誰建立的，方便你確認要不要去跟對方溝通） |
| Venus 管理 UI 顯示你的服務為 `stale` | 超過 15 分鐘沒收到心跳 | 純顯示信號，不影響工具能否被呼叫；檢查你的心跳排程是否還在跑 |
| AI 呼叫你的工具但總是失敗 / 卡住 | `usageGuide` 沒教清楚怎麼自我修正，或 `errorCategory` 太模糊 | 參考 §7 補強 `usageGuide`；用具體的 `errorCategory` 取代單一模糊的 `"error"` 字串 |
| AI 幾乎不呼叫你的工具 | `triggerDescription` 寫得不夠精準，或 `passMode='search'` 但語意搜尋沒有命中你的關鍵情境 | 補強 `triggerDescription`，明確描述「使用者問什麼樣的問題時該用這支工具」；必要時考慮改 `passMode='always'`（但會增加每輪對話的 token 成本，謹慎使用） |
| 你的檔案回傳（`files[]`）沒有出現在對話中 | base64 編碼錯誤或 `mime_type` 遺漏 | 確認 `files[].base64` 是純 base64 字串（不含 `data:xxx;base64,` 前綴），且 `mime_type` 正確 |

---

## 12. Version History

### v1.5.0（2026-07-22）

- 使用者回報兩個問題：(1) 用 MCP 服務取代既有手動設定的工具時，原本掛載的領域（Agent）綁定全部跑掉，要求無縫轉移、甚至無感；(2) MCP 端註冊好的工具不會自動掛載到 Venus 主要回應領域（primary agent），且期望 `passMode` 沒特別設定時預設就是按需搜尋
- 問題 (1) 根因：Venus 端刪除 MCP Server 時（如 §10「管理員刪除整個服務」這個實際會發生的操作）過去會把該服務底下所有工具與其 Agent 綁定一併硬刪；重新註冊拿到全新內部識別碼，之前的領域掛載救不回來
- 修法：Venus 端刪除服務改為只停用工具（不刪工具紀錄，不動 Agent 綁定關聯）；工具重新用同一批 `tool.name` 註冊時（不論新服務是否為刪除後重建），Venus 會自動偵測並原地回收，Agent 掛載全數保留——你這邊完全不需要做任何改動，架構細節見 [`DISTRIBUTED_MCP_DESIGN.md §6.2.4`](./DISTRIBUTED_MCP_DESIGN.md#624-孤兒工具回收刪除-mcp-server-重建時的無縫轉移2026-07-22-新增)
- 問題 (2) 前半部：新增「全新工具自動掛載至 primary agent」機制（§3 新增說明），你完全不需要額外設定或請管理員手動處理，架構細節見 [`DISTRIBUTED_MCP_DESIGN.md §6.2.5`](./DISTRIBUTED_MCP_DESIGN.md#625-新建工具自動掛載主要領域primary-agent2026-07-22-新增)
- 問題 (2) 後半部（`passMode` 未設定時的預設值）：確認本來就已經是 `search`（§4.1 早已如此記載），未變更程式碼

### v1.4.0（2026-07-22）

- 使用者回報同事的 MCP Server 更新了工具的 `pass_mode` 設定後，重新註冊、按 Venus 管理員「重新同步」都沒有生效，只有刪除整個 MCP Server 重建才成功——追查後確認這是舊版 §4.1 描述的既有設計（`pass_mode`/`tool_mode`/`platform_filter`/`require_verification` 僅首次建立時採用 manifest 值），使用者認為既然註冊/同步本身就已經是 Venus 管理員授權過的行為，不需要再對這 4 個欄位額外凍結，要求改為 manifest 全權覆寫
- §4.1「改欄位安全嗎？」段落更新：`pass_mode` / `tool_mode` / `platform_filter` / `require_verification` 改為每次註冊/同步都套用你 manifest 的最新值（不再有「僅首次建立生效」的限制）；唯一仍是 Venus 管理員專屬、你的 manifest 永遠碰不到的欄位是 `allowed_permissions` / `allowed_empids`（manifest 本來就沒有對應欄位可填）
- 底層變更：`utils/aiChatDb.js` `upsertMcpTools()` 的既有 mcp 工具更新分支、接管既存 internal/programmatic 工具分支，都補上這 4 欄的覆寫；架構細節見 [`DISTRIBUTED_MCP_DESIGN.md §5`](./DISTRIBUTED_MCP_DESIGN.md#5-欄位所有權模型layer-ownership) 的 2026-07-22 決策放寬說明

### v1.3.0（2026-07-22）

- 目標：使用者要求釐清「你的 manifest 撞到既有的、由 Venus 前端手動建立的『程式型工具』`tool_key` 時」到底會發生什麼——尤其是「同一位開發者把自己手動建的工具改用 MCP 發布」（預期無條件成功）與「撞到別人建立的工具」（預期需要人工把關，不能無條件信任接管）這兩種情境必須有不同結果
- 修正：§11 疑難排解表新增 `owned_by_different_developer` 這個 `conflicts[]` reason 的專屬排查列，說明 Venus 端如何用「既有工具建立者」與「你這個 MCP Server 登記的 `owner_empid`」比對身分來決定自動接管還是需要管理員在前端手動核准；架構細節見 [`DISTRIBUTED_MCP_DESIGN.md §6.2.2`](./DISTRIBUTED_MCP_DESIGN.md)
- `/dev-spec` 機器可讀規格新增 `registration.toolKeyCollisionPolicy` 區塊，把這個判斷邏輯也用機器可讀的方式表達，方便你自己的 AI 開發工具在寫 manifest 前先確認 `tool.name` 是否可能撞名

### v1.2.0（2026-07-22）

- 目標：使用者明確要求本協定必須對「任何從零開始開發的 MCP 系統」（HMRB 只是示範用例，非設計目標本身）通用、完整、可擴展地支援「業務權限標籤」概念，而非只讓恰好知道 Venus 舊有內部端點的既有系統（如 HMRB、Treasure Island，這些系統因為歷史上就用共享 JWT Cookie 方式與 Venus 整合而「順便」知道 `/api/organizations/employee-permissions` 存在）用得上
- 發現的落差：先前協定完全沒有任何「資料範圍篩選（同一支工具依呼叫者身分回傳不同內容）」的官方能力或文件章節——`context` 物件只給身分（§5.5），不給權限標籤（刻意的資料邊界設計），但**沒有任何地方教外部開發者「那我該怎麼查權限標籤」**，也沒有任何一個對 MCP Server 開放、且有認證保護的官方查詢端點。舊有的 `/api/organizations/employee-permissions` 完全沒有任何身分驗證（任何連得到內網的人帶 empid/email 就能查到員工姓名/信箱/職稱/權限），且從未出現在本文件或 `/dev-spec`——把它非正式地當成「MCP 官方能力」推廣等於教新系統依賴一個未受保護、未受契約保證的端點
- 修正：新增 §5.6「業務權限標籤查詢（Permission Tags）」，並在 Venus 端新增專屬於 MCP 協定的認證端點 `POST /api/ai/mcp-servers/employee-permissions`（沿用你已經有的 `shared_secret`，不需要另外申請任何憑證），底層呼叫與舊端點相同的權威邏輯（`utils/employeePermissionService.js`），但走 MCP 既有的 Bearer 認證模型、回應遵循四件套錯誤契約（`errorCategory`/`nextActionForAi`）、且刻意精簡回應內容（只回 `permissions` 標籤陣列，不夾帶姓名/信箱等 PII——多一份不必要的 PII 外洩管道）。`/dev-spec` 機器可讀規格同步補上 `employeePermissions` 區塊，AI 輔助開發時可直接看到此能力
- 明確劃清兩層權限的邊界（呼叫本工具的資格 vs 工具內資料的可見範圍），避免開發者把「該不該讓某人呼叫這支工具」（`adminOnly`，Venus 呼叫前已擋)與「這支工具內某幾筆資料該不該讓某人看到」（本節，MCP Server 自己判斷）混為一談而重複造輪子

### v1.1.0（2026-07-21）

- 目標：讓本文件成為**單一自足文件**（使用者要求「只靠一份文件就做到正確的分散式 MCP 服務開發」），全面補齊下列先前缺漏的精確技術細節：
  - §4.1：修正「改欄位安全嗎」footnote 漏列 `pass_mode` 為管理員專屬欄位（先前只列 5 個，實際是 6 個）；新增 `name` 正則比 `serverKey` 更嚴格（不允許 `-`）的踩雷提示
  - §4.2（新增）：完整 manifest 驗證規則表 + 驗證失敗/警告的精確 JSON 回應形狀（`{ field, reason, suggestion }`）
  - §5.1：新增 Venus 出站呼叫的精確 HTTP headers 表（`MCP-Protocol-Version` / `X-API-Key` / `X-Caller`）
  - §5.5（新增）：`context` 物件逐欄位精確定義（來源、fallback、空值語意），並修正 `platform` 欄位只會是 `web`/`teams` 的限制說明；新增逾時樓層演算法說明（`recommendedTimeoutSeconds` → 全域 30 秒 → agent 對話樓層 180 秒）
  - §6.1：新增 `files[]` 每檔 50MB 上限 + 超限靜默丟棄（無錯誤回傳給 AI）的踩雷警告
  - §8：補上 `adminOnly` 精確權限檢查邏輯（`isSystemAdmin || isModuleAdmin('ai-chat')`，非僅系統管理員一人）與被擋下時 AI 收到的確切回應形狀
  - §10（新增）：服務生命週期表——manifest 移除工具（停用不刪）、重新同步/測試連線、金鑰重生、停用整個服務、刪除整個服務，各自對你系統的確切影響
- 連帶修正 `services/ai/mcpClientService.js` / `services/ai/aiToolsManager.js` 的**真實程式碼缺陷**：撰寫 §5.5 逾時樓層說明時發現 `_buildMcpContext` 收斂後的 context 物件遺失 `source` 欄位，導致 `resolveTimeoutMs` 的 agent 樓層判斷對 MCP 工具永遠不會生效（與程式型工具 `_executeProgrammaticTool` 行為不等價）。已修正：`callTool` 新增 `callerSource` 參數，僅用於本機逾時解析，不寫入送往 MCP server 的 wire payload

### v1.0.1（2026-07-20）

- 修正 §10 疑難排解表格「註冊回 `409`（`conflicts` 非空）」的錯誤描述：`handleRegistration()` 實際上**一律回 HTTP `200`**，不論 `conflicts` 陣列是否非空——衝突只讓對應的那支工具被跳過，manifest 內其他沒衝突的工具仍會正常註冊成功。同步修正 `DISTRIBUTED_MCP_DESIGN.md §6.2.1` 與 `services/ai/mcpRegistryService.js` 內對應的錯誤描述/註解

### v1.0.0（2026-07-19）

- 初版發布，配合 HMRB ReguCheck 首發範例（8 支工具全數遷移完成）撰寫
- 相關文件：[`DISTRIBUTED_MCP_DESIGN.md`](./DISTRIBUTED_MCP_DESIGN.md)（Venus 內部架構設計）

---

*文件結束*
