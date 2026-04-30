# Prompt Optimizer — 專案總覽與速查

## 一句話總結

**Prompt Optimizer** 是一個 AI prompt 優化工具，讓使用者透過 LLM 迭代改進自己的 prompt，支援 Web App、Desktop (Electron)、Chrome Extension、Docker + MCP Server 五種部署模式，所有資料留存在使用者端，不流經第三方伺服器。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Node.js | ^22.0.0 | 服務器端執行環境 |
| 套件管理 | pnpm | 10.6.1 | Monorepo 套件管理 |
| 建置 | Vite | 最新 | Web/Extension 前端建置 |
| 建置 | tsup | ^8.5.1 | core 套件建置 (CJS + ESM) |
| 框架 | Vue 3 | 最新 | 前端 UI 框架 |
| UI Library | Naive UI | 最新 | Vue UI 組件庫 |
| 狀態管理 | Pinia | 最新 | Vue 狀態管理 |
| 模板引擎 | Mustache | ^4.2.0 | Prompt template 變數替換 |
| 桌面應用 | Electron | 最新 | 跨平台桌面應用 |
| MCP Protocol | @modelcontextprotocol/sdk | 最新 | MCP stdio/HTTP transport |
| LLM (OpenAI-compatible) | openai | ^6.33.0 | OpenAI 及 11 個兼容 Provider |
| LLM (Anthropic) | @anthropic-ai/sdk | ^0.80.0 | Claude 模型 |
| LLM (Gemini) | @google/genai | ^1.46.0 | Google Gemini |
| Storage (瀏覽器) | Dexie / IndexedDB | ^4.4.1 | 主要持久化儲存 |
| JSON 修復 | jsonrepair | ^3.13.3 | 修復 LLM 輸出的 JSON |
| i18n | vue-i18n | 最新 | zh-CN / zh-TW / en-US |
| 測試 | Vitest | ^4.1.2 | 單元測試 |
| E2E 測試 | Playwright | ^1.58.2 | E2E 測試 |
| 容器化 | Docker + nginx + supervisord | 最新 | Docker 部署 |
| 型別系統 | TypeScript | ^5.9.3 | 全專案型別安全 |
| Schema 驗證 | Zod | ^4.3.6 | LLM 輸出驗證 |

---

## 關鍵指令速查

```bash
# 安裝相依套件
pnpm install

# 本地開發 (Web)
pnpm dev          # 建置 core+ui，啟動 web dev server

# 本地開發 (Desktop)
pnpm dev:desktop  # 建置 core+ui，同時啟動 web + Electron

# Chrome Extension 開發
pnpm dev:ext      # 啟動 extension dev server

# 建置所有套件
pnpm build        # core → ui → web + extension (並行)

# 測試
pnpm test         # 所有單元測試
pnpm test:gate    # Gate 測試（pre-commit 必過）
pnpm test:e2e     # Playwright E2E 測試

# Lint & 型別檢查
pnpm lint         # 全專案 lint + typecheck

# MCP Server
pnpm mcp:dev      # 啟動 MCP Server 開發模式
pnpm mcp:build    # 建置 MCP Server

# 版本管理
pnpm version:prepare  # 升版號（不含 git tag）
pnpm version:sync     # 同步所有 package 版本號

# Docker 建置
docker build -t linshen/prompt-optimizer:latest .
docker run -d -p 80:80 -e ACCESS_PASSWORD=xxx linshen/prompt-optimizer:latest
```

---

## 文件地圖

| 文件 | 說明 |
|------|------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | 系統架構、元件關係、Sequence diagram |
| [CODEBASE_MAP.md](CODEBASE_MAP.md) | 程式碼地圖、目錄說明、「我想改 X 看哪裡」 |
| [DATA_MODEL.md](DATA_MODEL.md) | 資料模型、ER diagram、儲存策略 |
| [API_SURFACE.md](API_SURFACE.md) | MCP 工具 API、Core 服務介面、IPC channels |
| [LLM_INTEGRATION_FLOW.md](LLM_INTEGRATION_FLOW.md) | LLM 呼叫深度解析、Adapter Pattern、端對端串流範例 |
| [TEMPLATE_CATALOG.md](TEMPLATE_CATALOG.md) | 50+ 內建 templates 完整目錄、八大類別、設計模式 |
| [DEV_GUIDE.md](DEV_GUIDE.md) | 開發環境設定、測試方式、Debugging 技巧 |
| [DISCOVERY_LOG.md](DISCOVERY_LOG.md) | 探索紀錄、文件落差、TODO/技術債 |
| [_context/](./\_context/) | Stage 1-2 中繼分析資料（可刪除） |

---

## 專案術語表

| 術語 | 說明 |
|------|------|
| **Template** | prompt 優化用的模板，包含 Mustache 變數佔位符，分 Simple（string）和 Advanced（message array）兩種格式 |
| **PromptChain** | 一次優化操作及其後續迭代的版本鏈，共用同一個 `chainId` |
| **PromptRecord** | chain 中的一個版本，包含 `originalPrompt`、`optimizedPrompt`、模型資訊等 |
| **Provider** | LLM 服務提供商，如 OpenAI、Gemini、Anthropic |
| **Adapter** | 對應特定 Provider 的 SDK 封裝層，實作 `ITextProviderAdapter` |
| **Registry** | Adapter 的集中管理與查找器（`TextAdapterRegistry`、`ImageAdapterRegistry`） |
| **contextMode** | 目前是 system 還是 user 優化模式（`'system' | 'user'`） |
| **functionMode** | 目前的功能模式（`'basic' | 'pro' | 'image'`），決定顯示哪個 Workspace |
| **Electron Proxy** | 在 Electron 渲染進程中替代實際服務的 IPC 代理類（如 `ElectronLLMProxy`） |
| **IPC** | Electron 的進程間通訊機制（`ipcMain.handle` / `ipcRenderer.invoke`） |
| **MCP** | Model Context Protocol，允許 AI 工具（如 Claude Desktop）調用本專案功能 |
| **Structured Compare** | LLM-as-judge 的多輪 prompt 比較評估模式，輸出排名和分析 |
| **runtime_config** | Docker 部署時由 nginx 動態注入的 `window.runtime_config`，儲存 API keys 等設定 |
| **TemplateType** | template 的用途分類，決定它在哪個下拉選單中顯示（如 `optimize`、`iterate`、`evaluation`） |
| **Session Store** | 每種功能子模式（basic-system、pro-variable 等）的 Pinia store，管理該模式的 UI 狀態 |

---

## 支援的 LLM Providers

**文字模型**: OpenAI、Anthropic Claude、Google Gemini、DeepSeek、SiliconFlow、Zhipu AI、DashScope、OpenRouter、ModelScope、Ollama、MiniMax、Cloudflare Workers AI、自定義 OpenAI-compatible

**圖像模型**: Gemini、Seedream/火山方舟、SiliconFlow、OpenAI (DALL-E)、Cloudflare、DashScope、ModelScope、Ollama、OpenRouter
