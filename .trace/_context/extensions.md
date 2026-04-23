# Stage 2.4 Extension Points 分析

## 擴展點總覽

本專案有多個可擴展的接縫，開發者可在不修改核心的情況下擴充功能。

---

## 1. 新增 LLM Provider Adapter（最常見的擴展）

**接縫**: `packages/core/src/services/llm/adapters/`

**步驟**:
1. 建立 `my-provider-adapter.ts`，繼承 `AbstractTextProviderAdapter`:
   ```typescript
   export class MyProviderAdapter extends AbstractTextProviderAdapter {
     getProvider(): TextProvider { return { id: 'myprovider', name: 'My Provider', ... } }
     getModels(): TextModel[] { return [...] }
     protected async doSendMessage(messages, config): Promise<LLMResponse> { ... }
     protected async doSendMessageStream(messages, config, callbacks): Promise<void> { ... }
   }
   ```
2. 在 `packages/core/src/services/llm/adapters/registry.ts:52` 的 `initializeAdapters()` 中 `this.register(new MyProviderAdapter())`
3. 在 `packages/core/src/services/model/defaults.ts:11` 的 `PROVIDER_ENV_KEYS` 中新增環境變數對應
4. 在 `env.local.example` 補充環境變數說明

**現有 Adapter 數量**: 13 個文字 adapter + 9 個圖像 adapter

---

## 2. 新增圖像生成 Provider

**接縫**: `packages/core/src/services/image/adapters/`

類似文字 adapter，繼承 `AbstractImageProviderAdapter`，在 `ImageAdapterRegistry.initializeAdapters()` 中 register。

---

## 3. 新增內建 Template（優化策略）

**接縫**: `packages/core/src/services/template/default-templates/`

**步驟**:
1. 在對應目錄下新增 `my-template.ts` 和 `my-template_en.ts`
2. 在同目錄的 `content.ts` 中匯入並加入模板陣列
3. 在 `builders.ts` 中新增對應的 template builder

Template 中可使用的 Mustache 變數請參閱 `core_logic.md` 的「Mustache 變數系統」。

Template 的 `metadata.templateType` 決定它出現在哪個選擇器中（`TemplateSelect.vue`）。

---

## 4. 新增 Storage Provider

**接縫**: `packages/core/src/services/storage/types.ts`

實作 `IStorageProvider` interface，在 `StorageFactory.create()` 中新增分支。

目前 `StorageFactory.create('file')` 故意拋出錯誤（要求直接 new `FileStorageProvider`），因為 file storage 需要路徑參數。

---

## 5. 新增 Optimization 功能模式

**接縫**: `packages/ui/src/stores/session/`

每種子模式都有獨立的 Pinia session store。新增模式步驟：
1. 建立 `packages/ui/src/stores/session/useMyNewSession.ts`
2. 在 `packages/ui/src/stores/index.ts` 中 export
3. 在 `packages/ui/src/router/index.ts` 中新增路由
4. 在 `PromptOptimizerApp.vue` 的 `<RouterView>` 中它會自動渲染
5. 建立對應的 Workspace 組件

---

## 6. MCP 工具擴展

**接縫**: `packages/mcp-server/src/index.ts` 的 `setupServerHandlers()`

在 `ListToolsRequestSchema` handler 的工具陣列中新增工具定義，在 `CallToolRequestSchema` handler 中新增對應的處理邏輯。

MCP Server 使用 `CoreServicesManager` 取得所有 core 服務，可直接調用 `promptService`、`templateManager` 等。

---

## 7. TemplateLanguageService（模板語言切換）

**接縫**: `packages/core/src/services/template/languageService.ts`

支援切換內建 template 的語言（目前支援 `zh` 和 `en`）。  
實作 `ITemplateLanguageService` 可插入自訂的語言來源。

---

## 8. Composable / Hook 擴展（UI 層）

**模式**: Vue Composition API + Pinia

UI 層的功能透過 composable 組合（`packages/ui/src/composables/`），每個 composable 負責單一職責：

| Composable | 職責 |
|------------|------|
| `usePromptOptimizer` | 優化/迭代 prompt 的核心邏輯 |
| `useTemplateManager` | 模板選擇與管理 |
| `useModelManager` | 模型選擇與設定 |
| `useEvaluation` | 評估流程 |
| `useVariableExtractor` | 變數提取 |
| `useAppInitializer` | 服務初始化 |

新功能可新增 composable，並在對應的 Workspace 組件中 `use()` 即可。

---

## 9. 已知可替換點

| 可替換點 | 目前實作 | 替換方式 |
|----------|----------|----------|
| Template engine | Mustache | 修改 `TemplateProcessor.buildMessages()` |
| JSON repair | `jsonrepair` 庫 | 修改 `EvaluationService` 中的 repair 邏輯 |
| Storage (瀏覽器) | Dexie/IndexedDB | 實作 `IStorageProvider` |
| UI component library | Naive UI | 需修改所有 `.vue` 組件 |
| i18n | vue-i18n | 修改 `packages/ui/src/plugins/i18n.ts` |

---

## 限制與不建議修改的部分

- **`packages/core/`** 的公開 API（`index.ts` 的 export）盡量保持穩定，因為 desktop 主進程直接 `require` core
- **IPC channel 名稱**（`packages/desktop/main.js` 中的字串）與 proxy 類別中的名稱必須完全一致
- **Template ID** (`general-optimize`, `iterate` 等) 被硬編碼在 `PromptService.DEFAULT_TEMPLATES` 中，修改需同步更新
