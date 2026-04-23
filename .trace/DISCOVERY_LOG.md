# DISCOVERY_LOG — 探索紀錄與待解問題

**版本**: 2.9.4 | **探索日期**: 2026-04-23 | **範圍**: 全 monorepo 靜態分析 + 線上資料

---

## 1. Web Search 發現摘要

| 資源 | 連結 | 關鍵 Takeaway |
|------|------|--------------|
| GitHub 主倉庫 | https://github.com/linshenkx/prompt-optimizer | AGPL-3.0，活躍維護（v2.5.4 ~ v2.9.4） |
| Live Demo | https://prompt.always200.com | 可公開試用 |
| DeepWiki 自動文件 | https://deepwiki.com/linshenkx/prompt-optimizer | 最後索引 2025-10-19；記載 MCP 工具數有誤（說 2 個，實際 3 個） |
| ZRead 文件站 | https://zread.ai/linshenkx/prompt-optimizer | 含 Docker/Vercel 部署指南 |
| Chrome Extension | https://chromewebstore.google.com/detail/prompt-optimizer/cakkkhboolfnadechdlgdcnjammejlna | 可從 Web Store 取得 |
| Docker Hub | https://hub.docker.com/r/linshen/prompt-optimizer | 官方映像檔，含版本 tag |

**關鍵 Takeaway**:
- 專案在 GitHub 社群有 Trendshift 熱度指標，社群活躍度可透過 badge 追蹤
- 學術研究（arXiv:2511.20836）顯示結構化 prompting 平均提升約 6% 效能，與本專案 Structured Compare 的 LLM-as-judge 設計吻合
- 無官方 Discord/Slack，無 ADR 或設計決策文件，Issue 討論需直接存取 GitHub

---

## 2. 既有文件與程式碼的落差清單

| # | 文件說 | 程式碼實際 | 位置 |
|---|--------|-----------|------|
| 1 | `dev.md` 第 76-79 行：Dockerfile 有 `base` → `builder` → `production` **3 個**建置階段 | Dockerfile 實際只有 **2 個** `FROM`：`node:22-slim AS base` 和 `FROM base AS build`，第三個 `FROM node:22-alpine` 是 production stage，但 `dev.md` 將其命名為 `builder` 與 `production`，命名不一致 | `/home/user/prompt-optimizer/dev.md:76-79` vs `/home/user/prompt-optimizer/Dockerfile:1-3` |
| 2 | `dev.md` 第 16 行：`pnpm >= 8`（寬鬆版本要求） | `package.json` 第 5 行：`"packageManager": "pnpm@10.6.1"`（強制鎖定版本） | `/home/user/prompt-optimizer/dev.md:16` vs `/home/user/prompt-optimizer/package.json:5` |
| 3 | DeepWiki 文件（外部）：MCP Server 提供 **2 個**工具 | `mcp-server/src/index.ts` 實際宣告 **3 個**工具：`optimize-user-prompt`、`optimize-system-prompt`、`iterate-prompt` | `packages/mcp-server/src/index.ts` |
| 4 | `dev.md` Dockerfile 說明中的階段命名為 `builder`（第 78 行） | Dockerfile 中第二個階段命名為 `build`（非 `builder`） | `/home/user/prompt-optimizer/dev.md:78` vs `/home/user/prompt-optimizer/Dockerfile:6` |

> **注意**: `docs/user/mcp-server_en.md` 已正確記載 3 個工具，與程式碼一致。

---

## 3. TODO / FIXME / HACK / @deprecated 彙整

### 3.1 TODO 標記

| 檔案 | 行號 | 內容摘要 | 優先級 |
|------|------|----------|--------|
| `packages/core/src/services/preference/service.ts` | 63 | `TODO: 確認無舊資料後可安全移除（預計 v3.0）` — 廢棄的 storage keys：`app:selected-optimize-model`、`app:selected-test-model` | 高（v3.0 計畫） |
| `packages/ui/src/composables/ui/useTagSuggestions.ts` | 99 | `TODO: 未來可基於收藏更新時間優化 getRecentTags 邏輯`（目前 `getRecentTags` 回傳的就是 `getPopularTags`，未依時間排序） | 低 |
| `packages/ui/tests/unit/components/ImageModelManager.spec.ts` | 132 | `TODO: 添加測試用例` — `ImageModelManager` 元件測試完全空白（僅有 `it.todo()`） | 中 |

### 3.2 @deprecated 標記

| 檔案 | 行號 | 廢棄內容 | 替換方案 | 移除計畫 |
|------|------|----------|----------|----------|
| `packages/core/src/services/model/types.ts` | 56, 75 | `TextModelConfig.customParamOverrides` 及 `StoredTextModelConfig.customParamOverrides` | 使用 `paramOverrides`（已統一合併） | **v3.0 移除** |
| `packages/core/src/services/image/types.ts` | 60 | `ImageModelConfig.customParamOverrides` | 使用 `paramOverrides` | **v3.0 移除** |
| `packages/core/src/services/preference/service.ts` | 63 | Storage keys `app:selected-optimize-model`、`app:selected-test-model` | Session store 管理模型選擇 | **v3.0 移除** |
| `packages/core/src/services/llm/types.ts` | 193 | `ILLMService.sendMessage()` — 傳回合併字串 | 改用 `sendMessageStructured()` 取得完整 `LLMResponse`（含 reasoning） | 無明確移除時程 |
| `packages/core/src/services/llm/errors.ts` | 94 | `ERROR_MESSAGES` 常數（重新匯出舊錯誤碼）| 改用 `LLM_ERROR_CODES` from `@prompt-optimizer/core/constants/error-codes` | 無明確移除時程 |
| `packages/core/src/types/advanced.ts` | 145 | `ContextEditorState.variables` 欄位 | 改用 `useTemporaryVariables()` 和 `useVariableManager()` | 無明確移除時程 |
| `packages/ui/src/composables/prompt/usePromptOptimizer.ts` | 41 | `optimizationMode` 參數建議傳入 computed 值 | 從 `basicSubMode`/`proSubMode` 動態計算 | 文件警告，非強制 |
| `packages/ui/src/composables/app/useAppPromptGardenImport.ts` | 1196 | `getImageStorageService` 介面方法 | 改用 `getFavoriteImageStorageService` | 無明確移除時程 |

---

## 4. 未解答的疑問與模糊地帶

### 4.1 架構層面

- **Ollama 和 openai-compatible 的環境變數缺失**: `PROVIDER_ENV_KEYS`（`defaults.ts:11-21`）列出 11 個 provider，但 `TextAdapterRegistry.initializeAdapters()` 實際註冊 13 個 adapter（多了 `ollama` 和 `openai-compatible`）。這兩個 provider 沒有對應的環境變數 key，導致 `getBuiltinModelIds()` 的回傳清單**不包含** `ollama` 和 `openai-compatible`，是設計決策（Ollama 不需 API key，透過 custom 模型設定）還是疏漏？⚠️ 未驗證

- **`StorageAdapter.updateData()` 的 CAS 機制**: 文件說「防止並發衝突」，但瀏覽器單執行緒的 JavaScript 環境是否真的存在需要 CAS 的並發場景？Electron 多視窗情境是否有實際測試？⚠️ 未驗證

- **`HistoryManager` 上限 50 筆的依據**: `manager.ts:17` 硬編碼 `MAX_HISTORY = 50`，是否有效能測試或 IndexedDB 空間考量支撐此數字？⚠️ 未驗證

### 4.2 資料遷移層面

- **`customParamOverrides` 向後相容保留策略**: `manager.ts:257` 的註解說「保留 `customParamOverrides` 欄位以防版本回退，但新程式碼不再使用」。在儲存時（`manager.ts:426-429`）已主動清除此欄位設為 `undefined`。若使用者降版，資料是否完全遺失自定義參數？

- **Preference 舊 storage keys 的移除條件**: `preference/service.ts:63` 的 TODO 說「確認無舊資料後」才能移除，但沒有明確的「確認機制」——如何判斷「無舊資料」？

### 4.3 測試覆蓋層面

- **`ImageModelManager.spec.ts` 完全空白**: 唯一一個測試檔案中只有 `it.todo('應該正確初始化組件')`，圖像模型管理功能目前無單元測試覆蓋。

- **E2E 測試範圍**: Playwright 測試（`tests/e2e/`）的實際覆蓋率未知，是否涵蓋多 provider 情境？⚠️ 未驗證

---

## 5. 已知技術債

### 5.1 `customParamOverrides` 廢棄欄位（高優先級）

**位置**: `packages/core/src/services/model/types.ts:56,75`、`packages/core/src/services/image/types.ts:60`

**狀況**: 欄位已廢棄，功能已遷移到 `paramOverrides`。Migration 邏輯散佈於：
- `packages/core/src/services/model/manager.ts`（約 15 處）
- `packages/core/src/services/image-model/manager.ts`（約 10 處）
- `packages/core/src/services/llm/service.ts:291,362-363`
- `packages/core/src/services/image/service.ts:345-350`

**計畫**: v3.0 移除（需同步清理所有 Migration 邏輯、`converter.ts`、`defaults.ts` 中的殘留賦值）

### 5.2 `getBuiltinModelIds()` 的硬編碼 Provider 清單

**位置**: `packages/core/src/services/model/defaults.ts:47-49`

```typescript
export function getBuiltinModelIds(): string[] {
  return [...Object.keys(PROVIDER_ENV_KEYS), 'custom'];
}
```

**問題**: 此函數的輸出完全取決於 `PROVIDER_ENV_KEYS` 的靜態定義（11 個 provider）。但實際 `TextAdapterRegistry` 中已有 13 個 adapter（另有 `ollama`、`openai-compatible`）。若未來新增 adapter 但忘記更新 `PROVIDER_ENV_KEYS`，`isBuiltinModel()` 的判斷會靜默出錯，導致使用者無法正確識別內建 vs 自訂模型。

### 5.3 `ILLMService.sendMessage()` 廢棄介面

**位置**: `packages/core/src/services/llm/types.ts:193`

**問題**: 舊介面只回傳 `string`，丟失推理 token（reasoning）資訊。現有程式碼已有 `sendMessageStructured()` 替代，但沒有明確的移除時程計畫。

### 5.4 `ERROR_MESSAGES` 常數重複匯出

**位置**: `packages/core/src/services/llm/errors.ts:94`

**問題**: `ERROR_MESSAGES` 只是 `LLM_ERROR_CODES` 的代理（重新匯出），保留兩套 API 增加混淆，且無移除時程。

### 5.5 `useTagSuggestions.ts` 的排序邏輯缺失

**位置**: `packages/ui/src/composables/ui/useTagSuggestions.ts:99`

**問題**: `getRecentTags` 函數的語意是「最近使用的標籤」，但實際實作直接代理 `getPopularTags`（熱門標籤），未依時間排序。UI 上的顯示標籤可能與使用者預期不符。

---

## 6. 需要更深入調查的區域

以下區域在本次探索中觸及但未深入，建議開發者自行補充：

| 區域 | 原因 | 建議調查方向 |
|------|------|-------------|
| `packages/core/src/services/evaluation/` | Structured Compare 的 LLM-as-judge 流程複雜，未驗證錯誤處理路徑 | 驗證 `jsonrepair()` 失敗時的 fallback 行為；Zod schema 驗證失敗的使用者體驗 |
| `packages/core/src/services/storage/startup-safety-check.ts` | IndexedDB 損壞時的修復機制完整性 | 確認 `StartupRepairReport` 在 UI 層的顯示邏輯；確認 Electron FileStorage 是否有類似保護 |
| `packages/desktop/` IPC channel 完整清單 | IPC channel 名稱硬編碼，未列出全部 | 確認 `main.js` 與各 `electron-proxy.ts` 的 channel 名稱完全同步 |
| `tests/e2e/` Playwright 測試 | E2E 覆蓋範圍未知 | 確認是否有 multi-provider、Electron、MCP server 的 E2E 測試 |
| `packages/core/src/services/variable-extraction/` 和 `variable-value-generation/` | 這兩個服務在探索過程中未深入分析 | 確認 LLM 提取變數的 prompt 品質與錯誤容忍度 |
| Docker `generate-config.sh` 的安全性 | 環境變數動態注入到 `window.runtime_config` | 確認是否有 XSS 防護：若 API key 包含特殊字元，生成的 `config.js` 是否正確 escape |

---

## 7. 與維護者確認的問題清單

| # | 問題 | 背景 | 影響範圍 |
|---|------|------|----------|
| 1 | **`ollama` 和 `openai-compatible` 刻意不列入 `PROVIDER_ENV_KEYS`？** 這兩個 adapter 已在 `TextAdapterRegistry` 中但不在 `PROVIDER_ENV_KEYS`，導致 `getBuiltinModelIds()` 不包含它們。設計決策還是疏漏？ | `defaults.ts:11` vs `registry.ts:64,78` | 模型管理 UI 的「內建/自訂」判斷邏輯 |
| 2 | **`customParamOverrides` 的 v3.0 移除里程碑具體時間？** 遷移邏輯目前散佈廣泛，何時進行最終清理？是否有 migration script 計畫？ | `model/types.ts:56,75`、`image/types.ts:60` | 資料相容性、程式碼複雜度 |
| 3 | **`preference/service.ts` 中廢棄 storage keys 的移除條件如何確認？** `TODO: 確認無舊資料後可安全移除`——沒有具體的確認機制，何時可以安全刪除？ | `preference/service.ts:63` | import/export 功能的向後相容性 |
| 4 | **`HistoryManager` MAX_HISTORY = 50 的依據？** 是否有空間或效能測試支撐此上限？使用者有無途徑調整？ | `history/manager.ts:17` | 重度使用者的體驗 |
| 5 | **`ILLMService.sendMessage()` 的移除計畫？** 目前標記 `@deprecated` 但無明確時程，是否有外部整合（如 MCP server）依賴此介面？ | `llm/types.ts:193`、`llm/service.ts:114` | API 相容性 |
| 6 | **`generate-config.sh` 是否對 API key 中的特殊字元進行 escape？** 若 API key 包含單引號或 HTML 特殊字元，生成的 `window.runtime_config` 是否有 XSS 風險？ | `docker/generate-config.sh` | Docker 部署安全性 |
| 7 | **服務注入採用模組級單例（`shallowRef`）而非 Vue provide/inject 的設計原因？** 此模式對 SSR 或多 Vue app 實例是否有隱患？ | `packages/ui/src/plugins/pinia.ts` | 可測試性、框架相容性 |

---

*本文件由靜態程式碼分析 + 線上資料搜尋綜合產出，標注 ⚠️ 未驗證 的項目需實際執行程式碼或詢問維護者確認。*
