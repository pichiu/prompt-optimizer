# 更新計畫（Incremental Update Plan）

> 本次增量更新基於 commit 範圍 `a9cbcd4..3824b64`（2026-04-23 → 2026-04-30）。

---

## 變更摘要

- **Base Commit 範圍**: `a9cbcd4..3824b64`
- **變更檔案數**: 180（含 fixtures、release notes 等）
- **核心程式碼變更**: ~95 個檔案
- **變更幅度**: ~5.6%（輕量更新）
- **判定**: 繼續增量更新，**不需全量重跑**

## 主要主題

1. **Favorites Workspace 路由化** — 把原本內嵌的收藏管理區塊抽離成獨立的 routed page
2. **SOUL 結構化模板新增** — 為 OpenClaw 與 Hermes 系統新增 6 個專用 prompt 模板
3. **DeepSeek v4 升級**
4. **OpenAI image provider 模型清單刷新**
5. **Image-optimize 全部 13 個模板批次調整**
6. **Workspace clear content 工具新增**

---

## 受影響文件與更新策略

### 1. CODEBASE_MAP.md — 需要更新（小至中幅）

- **原因**：
  - 新增 `packages/ui/src/components/favorites/`（FavoritesPage + context）
  - 新增 `packages/ui/src/router/workspaceRoutes.ts`
  - 新增 `packages/ui/src/utils/`（external-data-loading, favorite-mode, favorite-reproducibility）
  - 新增 `packages/ui/src/components/common/WorkspaceUtilityMenu.vue`
  - 新增 `packages/ui/src/components/app-layout/workspaceRouteSwitch.ts`
  - 新增 6 個 SOUL templates：`optimize/soul-{hermes,openclaw}-compose{,_en}.ts`、`iterate/soul-iterate{,_en}.ts`
  - `FavoriteCard.vue` 已刪除（被 `FavoriteWorkspaceListItem.vue` 取代）
- **影響段落**: directory tree、「我想改 X 要看哪裡」速查表
- **更新策略**: **Main Agent**（局部新增/修改幾個段落）
- **依賴 context**: changelog.md

### 2. INDEX.md — 需要更新（小幅）

- **原因**：
  - 版本資訊：2.9.4 → 2.9.6
  - 新增 SOUL 模板可在「術語表」加一條（OpenClaw、Hermes、SOUL）
  - 新增 routed favorites page，可在「文件地圖」上方提及 favorites 是獨立路由
- **影響段落**: 一句話總結（版本）、術語表（新增 SOUL）、文件地圖（保持不變）
- **更新策略**: **Main Agent**（極少量改動）
- **依賴 context**: changelog.md

### 3. ARCHITECTURE.md — 需要更新（中幅）

- **原因**：
  - 路由表（第 342-348 行）需新增 `/favorites` route
  - 元件清單可加入 FavoritesPage、WorkspaceUtilityMenu
  - Vue Router 章節可記錄 routed favorites page 的設計
  - `workspaceRouteSwitch.ts` 是新的 helper，連結 router 與 session 切換
- **影響段落**: Vue Router 配置、元件清單、Workspace 路由表
- **更新策略**: **Main Agent**（局部修改既有表格與段落，不需重繪 Mermaid 圖；架構主結構未變）
- **依賴 context**: changelog.md, recon.md（Vue Router 段落）

### 4. DATA_MODEL.md — 不需更新

- **原因**: 沒有 schema/migration 變動。Favorite 的資料模型在 core 端 (`services/favorite/`) 並未變更，所有變化都在 UI 呈現層（拆元件、路由化）。

### 5. API_SURFACE.md — 需要更新（小幅）

- **原因**：
  - MCP 工具 API、Core service 介面、IPC channels 都未變動
  - 但前端 router 是「使用者可直接訪問的 URL 介面」，新增 `/favorites` 路由值得在前端 routes 段落補一行
  - 若文件目前有列出前端路由表，需新增 `/favorites`
- **影響段落**: 前端路由章節（如有）
- **更新策略**: **Main Agent**（檢查是否有前端 routes 段落，若有則加一行；若無則跳過）
- **依賴 context**: changelog.md

### 6. DEV_GUIDE.md — 需要更新（小幅）

- **原因**：
  - 新增 e2e 測試 `tests/e2e/session-persistence/workspace-clear-content.spec.ts`
  - 新增 `packages/core/tests/integration/llm/deepseek-live.integration.test.ts`
  - 新增多個 favorites 相關單元測試
  - `scripts/release-notes.js` + `.test.mjs` 是 release 流程的工具
- **影響段落**: 測試策略、測試檔案分布概覽（如有）
- **更新策略**: **Main Agent**（可能只加一兩行；若 DEV_GUIDE 只談原則不列詳細路徑則可能不必動）
- **依賴 context**: changelog.md

### 7. LLM_INTEGRATION_FLOW.md — 需要更新（小幅）

- **原因**：
  - DeepSeek adapter 升級到 v4 模型支援
  - OpenAI image provider 預設模型清單刷新
  - Image adapters 7 個（dashscope, gemini, modelscope, openai, openrouter, seedream, siliconflow）有 JSON wrapper 處理修正
- **影響段落**: 文字 adapters（DeepSeek 條目）、image adapters（如有）
- **更新策略**: **Main Agent**（更新 DeepSeek 模型列表、加註 image adapter JSON wrapper 邊界處理改動，不重寫整章）
- **依賴 context**: changelog.md

### 8. TEMPLATE_CATALOG.md — 需要更新（中至大幅）

- **原因**：
  - **新增 6 個 SOUL 結構化模板**（這是本次最重要的 template 變更）：
    - `soul-iterate` + `soul-iterate_en`（iterate type）
    - `soul-hermes-compose` + `soul-hermes-compose_en`（optimize type）
    - `soul-openclaw-compose` + `soul-openclaw-compose_en`（optimize type）
  - 13 個 image-optimize 模板（中英版）內容修訂 — 不影響 ID/結構，但若文件中引用了具體文案需檢查
- **影響段落**: 「optimize」與「iterate」類別清單；可能需新增「SOUL 模板」子類別
- **更新策略**: **Sub-Agent**（需要新增多個模板說明，且要寫清楚 OpenClaw / Hermes 的用途與差異；非機械式 append，建議 sub-agent 處理）
- **依賴 context**: changelog.md、實際讀取 6 個 SOUL template 檔案

### 9. DISCOVERY_LOG.md — 需要更新（追加段落）

- **原因**: 紀律性追加本次更新的發現與決策
- **更新策略**: **Main Agent**（追加 2026-04-30 更新段落）
- **依賴 context**: 完成其他文件更新後再寫，總結本次的非顯而易見發現

---

## 執行順序

依照依賴與「上游 → 下游」原則：

1. **CODEBASE_MAP.md**（目錄結構是基礎）
2. **INDEX.md**（術語/版本）
3. **ARCHITECTURE.md**（路由表 + 元件清單）
4. **API_SURFACE.md**（前端路由補充，若適用）
5. **LLM_INTEGRATION_FLOW.md**（DeepSeek + image adapters）
6. **TEMPLATE_CATALOG.md**（SOUL 模板 — Sub-Agent）
7. **DEV_GUIDE.md**（測試策略小註）
8. **DISCOVERY_LOG.md**（最後追加）

> 註：DATA_MODEL.md 不在本次更新範圍。

---

## 不更新的文件

| 文件 | 原因 |
|------|------|
| DATA_MODEL.md | 沒有 schema/migration/entity 變動 |

---

## Web Search 判定

**不執行 web search**：
- DeepSeek v4 是既有 provider 的 model 列表更新，無新技術
- SOUL 模板是專案內部新增（Asus 內部 OpenClaw / Hermes 框架專用），延用現有 template 註冊機制
- 無框架/runtime 重大升級（package.json 只是 patch 版號 bump）
