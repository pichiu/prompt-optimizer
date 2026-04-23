# Stage 2.5 外部整合分析

## 外部 API / SDK / 服務

---

## 1. OpenAI / OpenAI-compatible (openai npm SDK)

**路徑**: `packages/core/src/services/llm/adapters/openai-adapter.ts`  
**SDK**: `openai` ^6.33.0

**呼叫方式**:
```typescript
const openai = new OpenAI({ apiKey, baseURL })
// 非流式
const response = await openai.chat.completions.create({ model, messages })
// 流式
const stream = await openai.chat.completions.create({ model, messages, stream: true })
for await (const chunk of stream) { ... }
```

**Provider 包含**: OpenAI, OpenAICompatible, DeepSeek, SiliconFlow, Zhipu, DashScope, Ollama, OpenRouter, ModelScope

**失敗處理**: SDK 拋出原生錯誤，由 `AbstractTextProviderAdapter` 的公共方法捕獲並轉換為 `APIError`

---

## 2. Anthropic Claude (@anthropic-ai/sdk)

**路徑**: `packages/core/src/services/llm/adapters/anthropic-adapter.ts`  
**SDK**: `@anthropic-ai/sdk` ^0.80.0

**特殊處理**:
- Anthropic 的 `thinking` 功能輸出 `<think>` 標籤
- `processThinkTags(content)` 方法在 `abstract-adapter.ts` 中處理，分離 content 與 reasoning
- Anthropic 的 API 格式略不同（system prompt 是獨立參數而非 messages 第一條）

---

## 3. Google Gemini (@google/genai)

**路徑**: `packages/core/src/services/llm/adapters/gemini-adapter.ts`  
**SDK**: `@google/genai` ^1.46.0

**特殊處理**:
- Gemini 支援圖像生成（text2image），走 image adapter 路徑
- 圖像生成 API 與文字生成 API 完全不同
- `GeminiAdapter` 同時被文字和圖像 registry 使用（部分功能）

---

## 4. LLM Provider 失敗處理模式

**統一策略**: 無 retry、無 circuit breaker，依賴 SDK 內建的 timeout 和錯誤傳播

```typescript
// abstract-adapter.ts (簡化)
async sendMessage(messages, config): Promise<LLMResponse> {
  try {
    return await this.doSendMessage(messages, config)
  } catch (error) {
    // SDK 原始錯誤直接往上拋（保留完整堆栈）
    throw error
  }
}
```

UI 層在 `onError` callback 中顯示 toast 通知使用者。

---

## 5. Dexie (IndexedDB ORM)

**路徑**: `packages/core/src/services/storage/dexieStorageProvider.ts`  
**SDK**: `dexie` ^4.4.1

**資料庫名稱**: `PromptOptimizerDB`

Dexie 用作 key-value store（`setItem/getItem` interface），不使用其 relational query 功能。

**啟動安全檢查** (`packages/core/src/services/storage/startup-safety-check.ts`):
- 每次啟動時驗證 IndexedDB 資料完整性
- 若發現損壞資料，嘗試修復並記錄 `StartupRepairReport`
- Repair report 存入 preference，下次啟動後顯示給使用者

**圖像儲存** (獨立資料庫):
- `PromptOptimizerImageDB` — session 期間生成的圖像（50MB 上限，7天過期，100張）
- `PromptOptimizerFavoriteImageDB` — 收藏快照圖像（200MB，無過期）

---

## 6. Electron 自動更新 (electron-updater)

**路徑**: `packages/desktop/main.js`

使用 `electron-updater` 檢查 GitHub Releases，支援：
- 自動下載並在重啟時安裝
- `dev-app-update.yml` 設定開發環境的更新伺服器
- IPC events: `update-available`, `update-downloaded`, `install-update`

---

## 7. Model Context Protocol (MCP)

**路徑**: `packages/mcp-server/src/index.ts`  
**SDK**: `@modelcontextprotocol/sdk`

**兩種 transport**:
1. **stdio** — 直接被 Claude Desktop 等工具 fork 為子進程，透過 stdin/stdout 通訊
2. **HTTP (Streamable HTTP)** — 使用 express + `StreamableHTTPServerTransport`，透過 `/mcp` 路徑提供服務

Docker 部署時兩者並存：nginx 在 `/mcp` 路徑代理到 MCP HTTP server（port 3000）。

---

## 8. Vercel Analytics

**路徑**: `packages/web/src/main.ts:51`

條件載入：只在 `VITE_VERCEL_DEPLOYMENT=true` 時才動態插入 `/_vercel/insights/script.js`。

---

## 9. Nginx（Docker 部署）

**路徑**: `docker/nginx.conf`

職責：
- 靜態文件服務（`packages/web/dist/` → `/`）
- `/mcp` 路徑反代到 MCP HTTP server (`localhost:3000`)
- HTTP Basic Auth（若設置了 `ACCESS_PASSWORD`）

---

## 10. MiniMax API

**路徑**: `packages/core/src/services/llm/adapters/minimax-adapter.ts`  

MiniMax 有自己的 API 格式（非 OpenAI 兼容），但與聊天補全類似，使用原生 fetch。

---

## 11. Cloudflare Workers AI

**路徑**: `packages/core/src/services/llm/adapters/cloudflare-adapter.ts`

同時支援文字和圖像生成。API endpoint 格式：  
`https://api.cloudflare.com/client/v4/accounts/{accountId}/ai/run/{model}`

需要 `VITE_CF_API_TOKEN` 和 `VITE_CF_ACCOUNT_ID` 兩個環境變數。

---

## 12. Seedream / 火山方舟 ARK (ByteDance)

**路徑**: `packages/core/src/services/image/adapters/seedream.ts`

只支援圖像生成，使用 ARK API，需要 `VITE_SEEDREAM_API_KEY` 或 `VITE_ARK_API_KEY`。

---

## 外部依賴安全性考量

**資安設計原則**: 所有 API key 均在 client-side 使用（純前端架構），不通過中繼伺服器。

這意味著：
- ✅ 使用者資料不經過第三方伺服器
- ⚠️ API key 在瀏覽器 localStorage/IndexedDB 中儲存（相對安全，不能跨域讀取）
- ⚠️ Desktop App 的 API key 存於本機 FileStorage
- ⚠️ Docker/Vercel 環境的 API key 可透過環境變數預置，會被編譯進 JS bundle（`VITE_` 前綴環境變數在建置時替換）
