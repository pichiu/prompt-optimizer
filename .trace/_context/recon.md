# Stage 1 偵察報告

## 1. 專案概覽

**名稱**: Prompt Optimizer  
**版本**: 2.9.4  
**授權**: AGPL-3.0-only  
**作者**: linshenkx  
**GitHub**: https://github.com/linshenkx/prompt-optimizer  
**Live Demo**: https://prompt.always200.com  

**一句話描述**: 一個幫助使用者撰寫更好 AI prompt 的優化工具，支援 Web、Desktop (Electron)、Chrome Extension、Docker、MCP Server 五種部署模式，純 client-side 架構，資料不經第三方伺服器。

---

## 2. 技術棧

| 類別 | 技術 | 版本 | 備註 |
|------|------|------|------|
| Runtime | Node.js | ^22.0.0 | 強制要求 |
| 套件管理 | pnpm | 10.6.1 | 強制，不可用 npm/yarn |
| 框架 | Vue 3 | 最新 | Composition API |
| UI 組件庫 | Naive UI | - | `@prompt-optimizer/ui` 封裝 |
| 狀態管理 | Pinia | - | Session stores |
| 建置工具 | Vite / tsup | - | Web/Extension 用 Vite，core 用 tsup |
| 桌面應用 | Electron | - | `packages/desktop` |
| 模板引擎 | Mustache | ^4.2.0 | prompt template 變數替換 |
| 資料庫/Storage | Dexie (IndexedDB) | ^4.4.1 | 瀏覽器端 |
| ORM/Storage | localStorage | - | 備用/輕量方案 |
| 桌面 Storage | FileStorageProvider | - | Electron file system |
| i18n | vue-i18n | - | zh-CN / zh-TW / en-US |
| 測試 | Vitest | ^4.1.2 | 單元測試 |
| E2E 測試 | Playwright | ^1.58.2 | |
| MCP Protocol | @modelcontextprotocol/sdk | - | HTTP + stdio transport |
| HTTP Server (MCP) | express | - | MCP HTTP 模式 |
| 容器化 | Docker + nginx | - | nginx 反代靜態資源+MCP |
| 進程管理 (Docker) | supervisord | - | 管理 nginx + MCP |
| Linting | ESLint | - | |
| 型別檢查 | TypeScript | ^5.9.3 | |
| CSS | Tailwind CSS | - | (部分使用) |
| 布局 | PostCSS | - | |

### LLM Provider SDK
- `openai` ^6.33.0 — OpenAI, DeepSeek, SiliconFlow, Zhipu, Ollama (OpenAI compatible)
- `@anthropic-ai/sdk` ^0.80.0 — Anthropic Claude
- `@google/genai` ^1.46.0 — Google Gemini
- `zod` ^4.3.6 — 參數 schema 驗證
- `jsonrepair` ^3.13.3 — LLM 輸出 JSON 修復
- `diff` ^8.0.4 — prompt diff 比對

---

## 3. 架構模式

**Monorepo** (pnpm workspace)，架構分層：

```
packages/
├── core/          # 純邏輯層，Platform-agnostic，無 UI 依賴
├── ui/            # Vue 3 + Naive UI 組件庫，依賴 core
├── web/           # Vite SPA，依賴 ui
├── extension/     # Chrome Extension，依賴 ui
└── desktop/       # Electron App，依賴 web/ui/core (透過 IPC proxy)
packages/mcp-server/  # MCP Server (Node.js)，依賴 core
```

**設計原則**:
1. `core` 層完全平台無關，可在瀏覽器/Node/Electron 執行
2. `ui` 層封裝所有 Vue 組件，web/extension 共用
3. Electron 使用 IPC 代理模式（`electron-proxy.ts`），主進程執行 core 服務，渲染進程透過 IPC 呼叫
4. MCP server 在 Node.js 環境直接實例化 core 服務

---

## 4. 目錄結構 (3層深)

```
prompt-optimizer/
├── packages/
│   ├── core/                  # 核心邏輯 (TypeScript, tsup 建置)
│   │   └── src/
│   │       ├── constants/     # 錯誤碼、storage keys
│   │       ├── interfaces/    # import-export 介面
│   │       ├── services/      # 所有核心服務
│   │       │   ├── adapters/  # 抽象 adapter registry
│   │       │   ├── compare/   # 比較服務
│   │       │   ├── context/   # 上下文模式 (system/user)
│   │       │   ├── data/      # 資料管理
│   │       │   ├── evaluation/ # AI 評估服務
│   │       │   ├── favorite/  # 收藏管理
│   │       │   ├── history/   # 優化歷史
│   │       │   ├── image/     # 圖像生成服務
│   │       │   ├── image-model/ # 圖像模型管理
│   │       │   ├── image-understanding/ # 圖像理解服務
│   │       │   ├── llm/       # LLM 服務 + 多 provider adapters
│   │       │   ├── model/     # 文字模型管理
│   │       │   ├── preference/ # 使用者偏好
│   │       │   ├── prompt/    # 核心 prompt 優化服務
│   │       │   ├── storage/   # 儲存抽象層 (多實現)
│   │       │   ├── template/  # template 管理+處理器+內建模板
│   │       │   ├── variable-extraction/ # 變數提取服務
│   │       │   └── variable-value-generation/ # 變數值生成服務
│   │       ├── types/         # 全局型別定義
│   │       └── utils/         # 工具函式
│   ├── ui/                    # Vue UI Library
│   │   └── src/
│   │       ├── components/    # Vue 組件 (~60個)
│   │       ├── composables/   # Composition API hooks
│   │       ├── stores/        # Pinia stores (session-based)
│   │       ├── i18n/          # 多語言 (zh-CN/zh-TW/en-US)
│   │       ├── plugins/       # Vue plugins (i18n, pinia)
│   │       ├── router/        # Vue Router
│   │       ├── config/        # Naive UI 主題設定
│   │       └── types/         # UI 型別定義
│   ├── web/                   # Vite Web SPA
│   │   └── src/
│   │       ├── App.vue        # 純殼，渲染 PromptOptimizerApp
│   │       └── main.ts        # 入口，安裝 i18n/Pinia/Router
│   ├── extension/             # Chrome Extension
│   │   └── src/
│   │       ├── App.vue        # Extension 主組件
│   │       └── main.ts        # 入口
│   ├── desktop/               # Electron App
│   │   ├── main.js            # 主進程，IPC handler 宣告
│   │   ├── preload.js         # Preload script，暴露 electronAPI
│   │   └── config/            # proxy-dispatcher, update-config, etc.
│   └── mcp-server/            # MCP Protocol Server
│       └── src/
│           ├── index.ts       # 伺服器入口，工具註冊
│           ├── start.ts       # 啟動邏輯
│           └── adapters/      # core services 初始化
├── api/
│   └── auth.js                # Vercel Edge Function，密碼保護 middleware
├── docker/                    # Docker 相關設定
├── docs/                      # 使用者文件
├── mkdocs/                    # MkDocs 文件設定
├── scripts/                   # 建置/測試/版本管理腳本
├── tests/
│   └── e2e/                   # Playwright E2E 測試
├── releases/                  # 各版本發布說明
├── vercel.json                # Vercel 部署設定
├── docker-compose.yml         # Docker Compose
├── middleware.js              # Vercel middleware (密碼保護)
├── env.local.example          # 環境變數範例
└── package.json               # Root monorepo，協調建置
```

---

## 5. 核心服務清單 (packages/core/src/services/)

| 服務 | 類別 | 主要職責 |
|------|------|----------|
| `PromptService` | 核心業務 | prompt 優化/迭代/測試，協調 LLM + Template |
| `LLMService` | 基礎設施 | 透過 Adapter registry 調用 LLM |
| `TemplateManager` | 模板管理 | CRUD 自定義/內建 template |
| `TemplateProcessor` | 模板處理 | Mustache 渲染，訊息陣列組裝 |
| `ModelManager` | 模型管理 | 文字模型設定 CRUD |
| `HistoryManager` | 歷史記錄 | 優化鏈 (chain) 管理，max 50 筆 |
| `EvaluationService` | 評估 | LLM 評估、結構化比較評估 |
| `CompareService` | 比較 | 多版本 prompt 比較 |
| `ImageService` | 圖像生成 | T2I/I2I 請求協調 |
| `ImageModelManager` | 圖像模型 | 圖像模型設定管理 |
| `FavoriteManager` | 收藏 | 收藏 prompt 管理 |
| `DataManager` | 資料管理 | import/export 所有資料 |
| `PreferenceService` | 偏好設定 | 使用者個人設定 |
| `VariableExtractionService` | 變數提取 | 從 prompt 中自動提取變數 |
| `VariableValueGenerationService` | 變數值生成 | LLM 生成測試變數值 |
| `StorageFactory` | 儲存 | 創建 localStorage/Dexie/Memory/File 實例 |

---

## 6. 支援的 LLM Providers

**文字模型 (TextAdapterRegistry)**:
- OpenAI (openai-adapter)
- OpenAI Compatible (openai-compatible-adapter) — Ollama、自定義
- Anthropic Claude (anthropic-adapter)
- Google Gemini (gemini-adapter)
- DeepSeek (deepseek-adapter)
- SiliconFlow (siliconflow-adapter)
- Zhipu AI (zhipu-adapter)
- DashScope / 阿里百炼 (dashscope-adapter)
- OpenRouter (openrouter-adapter)
- ModelScope (modelscope-adapter)
- Ollama (ollama-adapter)
- MiniMax (minimax-adapter)
- Cloudflare Workers AI (cloudflare-adapter)

**圖像模型 (ImageAdapterRegistry)**:
- Gemini (gemini image)
- Seedream / 火山方舟 ARK
- SiliconFlow (image)
- OpenAI (DALL-E)
- Cloudflare AI (image)
- DashScope (image)
- ModelScope (image)
- Ollama (image)
- OpenRouter (image)

---

## 7. 資料儲存策略

| 環境 | 儲存 Provider | 備註 |
|------|---------------|------|
| 瀏覽器 (Web/Extension) | Dexie (IndexedDB) | 主要，支援大量資料 |
| 瀏覽器 fallback | localStorage | 舊瀏覽器/設定資料 |
| Electron (Desktop) | FileStorageProvider | JSON 文件，存於 userData 目錄 |
| 測試/SSR | MemoryStorageProvider | 記憶體，不持久化 |

---

## 8. 部署模式

| 模式 | 說明 |
|------|------|
| Web (Vercel) | 靜態 SPA，透過 vercel.json + middleware.js 加密碼保護 |
| Web (Docker) | nginx 靜態服務 + supervisord 管理 MCP server，port 80 |
| Desktop | Electron 應用，跨平台，支援自動更新 |
| Chrome Extension | 瀏覽器插件，Chrome Web Store 可取得 |
| MCP Server | stdio 或 HTTP transport，與 Claude Desktop 等工具整合 |

---

## 9. 既有文件概覽

| 文件路徑 | 內容摘要 |
|----------|----------|
| `README.md` | 完整英文 README，功能介紹、部署方式 |
| `README.zh-CN.md` | 簡體中文 README |
| `dev.md` | 開發環境設定、Docker 建置、常見問題 |
| `CHANGELOG.md` | 版本更新紀錄 |
| `AGENTS.md` | AI agent 開發指南 |
| `docs/image-mode.md` | 圖像模式使用說明 |
| `packages/ui/docs/` | UI 組件 API、無障礙指南 |
| `packages/extension/` | Chrome Extension 相關 md |
| `packages/desktop/README*.md` | Desktop 配置說明 |
| `mkdocs/` | MkDocs 文件站設定 |

**與程式碼的主要落差**:
- `dev.md` 提到 3 個建置階段，但 Dockerfile 實際只有 2 個（base/build）
- `dev.md` 的 pnpm 版本要求寫「>= 8」，但 `package.json` 實際指定 `pnpm@10.6.1`
- README 說「MCP 提供 2 個工具」，但 `mcp-server/src/index.ts` 實際宣告 3 個工具 (optimize-user-prompt, optimize-system-prompt, iterate-prompt)

---

## 10. 功能模式分類

專案的 UI 依「功能模式」組織：

| 模式 | 子模式 | 說明 |
|------|--------|------|
| **Basic** | System | 優化 system prompt |
| **Basic** | User | 優化 user prompt |
| **Pro** | Multi-message | 多輪對話模式，單條訊息優化 |
| **Pro** | Variable | 變數模式，批量替換測試 |
| **Image** | Text2Image (T2I) | 文生圖 prompt 優化 |
| **Image** | Image2Image (I2I) | 圖生圖 prompt 優化 |

每種模式有獨立的 Pinia session store (`packages/ui/src/stores/session/`)。
