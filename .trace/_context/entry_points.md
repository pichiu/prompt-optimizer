# Stage 2.1 Entry Points 分析

## 各平台的啟動入口

---

### 1. Web App (`packages/web`)

**主入口**: `packages/web/src/main.ts`

啟動流程：
```
packages/web/src/main.ts (createApp)
  → installI18nOnly(app)    # 安裝 vue-i18n plugin
  → installPinia(app)        # 安裝 Pinia plugin
  → app.use(router)          # 安裝 Vue Router
  → router.isReady()         # 等待首次路由解析完成
  → app.mount('#app')        # 掛載到 DOM
```

App 組件：`packages/web/src/App.vue` — 純殼，僅渲染 `<PromptOptimizerApp />`

---

### 2. Chrome Extension (`packages/extension`)

**主入口**: `packages/extension/src/main.ts`

啟動流程與 Web 相同（也使用 `@prompt-optimizer/ui`），差異在於打包成 Chrome Extension 格式，有獨立的 `public/background.js`。

---

### 3. Electron Desktop (`packages/desktop`)

**主進程入口**: `packages/desktop/main.js`

主進程啟動流程：
```
main.js 頂部 (require 階段)
  → ConsoleLogger 初始化（捕獲全局錯誤）
  → dotenv 載入 .env.local / .env
  → 從 @prompt-optimizer/core import 所有服務 factory
app.whenReady()
  → 創建所有 core 服務實例（不通過 proxy，直接在主進程中）：
      storageProvider = new FileStorageProvider(userData)
      modelManager   = createModelManager(storageProvider)
      templateManager = createTemplateManager(storageProvider, languageService)
      historyManager  = createHistoryManager(storageProvider, modelManager)
      llmService      = createLLMService(modelManager)
      promptService   = createPromptService(...)
      imageService    = createImageService(...)
      favoriteManager = new FavoriteManager(storageProvider)
  → 設置 IPC handlers（ipcMain.handle）
  → createWindow() → 創建 BrowserWindow，載入 web/dist/index.html
```

**Preload Script**: `packages/desktop/preload.js`
- 使用 `contextBridge.exposeInMainWorld('electronAPI', {...})` 暴露 IPC 方法給渲染進程
- 渲染進程透過 `window.electronAPI` 呼叫主進程功能

---

### 4. MCP Server (`packages/mcp-server`)

**主入口**: `packages/mcp-server/src/index.ts`

啟動流程：
```
index.ts
  → loadConfig()                 # 從環境變數載入設定
  → createServerInstance(config)
      → new Server(...)          # 創建 MCP Server 實例
      → CoreServicesManager.getInstance()
      → coreServices.initialize(config)  # 初始化 core 服務
  → setupServerHandlers(server, coreServices)
      → getTemplateOptions(...)  # 載入 template 選項（給工具定義用）
      → server.setRequestHandler(ListToolsRequestSchema, ...)  # 工具清單
      → server.setRequestHandler(CallToolRequestSchema, ...)   # 工具呼叫
  → 根據 config.transport 選擇:
      stdio: new StdioServerTransport() + server.connect()
      http:  express app + StreamableHTTPServerTransport
```

**MCP 工具清單** (實際 3 個):
1. `optimize-user-prompt` — 優化 user prompt
2. `optimize-system-prompt` — 優化 system prompt
3. `iterate-prompt` — 迭代優化現有 prompt

---

### 5. 核心服務初始化 (`useAppInitializer`)

**路徑**: `packages/ui/src/composables/system/useAppInitializer.ts:94`

這是 Web/Extension 環境的服務初始化器，在 `PromptOptimizerApp.vue` 的 `onMounted` 鉤子中執行。

**環境判斷分支**:

```
isRunningInElectron()
├── true (Electron)
│   └── 所有服務使用 IPC Proxy 類：
│       ElectronModelManagerProxy, ElectronTemplateManagerProxy,
│       ElectronHistoryManagerProxy, ElectronLLMProxy,
│       ElectronPromptServiceProxy, ElectronPreferenceServiceProxy,
│       ElectronContextRepoProxy, ElectronImageModelManagerProxy,
│       ElectronImageServiceProxy, ElectronDataManagerProxy,
│       FavoriteManagerElectronProxy
│       （以上 proxy 方法通過 window.electronAPI IPC 橋接到主進程）
└── false (Web/Extension)
    └── 建立真實服務實例：
        storageProvider = StorageFactory.create('dexie')
        preferenceService = createPreferenceService(storageProvider)
        languageService = createTemplateLanguageService(preferenceService)
        modelManager = createModelManager(storageProvider)
        textAdapterRegistry = createTextAdapterRegistry()
        imageAdapterRegistry = createImageAdapterRegistry()
        imageModelManager = createImageModelManager(storageProvider, imageAdapterRegistry)
        historyManager = createHistoryManager(storageProvider, modelManager)
        await languageService.initialize()
        await modelManager.ensureInitialized()
        llmService = createLLMService(modelManager, textAdapterRegistry)
        promptService = createPromptService(modelManager, llmService, templateManager, historyManager)
        evaluationService = createEvaluationService(...)
        variableExtractionService = createVariableExtractionService(...)
        favoriteManager = new FavoriteManager(storageProvider)
```

初始化完成後：
1. `services.value = { ...所有服務 }` (shallowRef)
2. `setPiniaServices(services.value)` — 注入到 Pinia，供所有 stores/composables 使用
3. `scheduleImageStorageGc(...)` — 排程圖像 GC

---

### 6. Vue Router 路由

**路徑**: `packages/ui/src/router/index.ts`

路由結構（推測）：
```
/                    → redirect to /basic/system
/basic/system        → BasicSystemWorkspace
/basic/user          → BasicUserWorkspace
/pro/multi           → ContextSystemWorkspace (多輪對話模式)
/pro/variable        → ContextUserWorkspace  (變數模式)
/image/text2image    → ImageText2ImageWorkspace
/image/image2image   → ImageImage2ImageWorkspace
```

（路由在 `PromptOptimizerApp.vue` 的 `<RouterView>` 中渲染）

---

### 7. 服務注入模式

專案採用**模組級單例 + shallowRef** 方式注入服務（非標準 Vue provide/inject）：

```typescript
// packages/ui/src/plugins/pinia.ts
const servicesRef = shallowRef<AppServices | null>(null)

export function setPiniaServices(services: AppServices | null) {
  servicesRef.value = services
}

export function getPiniaServices(): AppServices | null {
  return servicesRef.value
}
```

所有 Pinia stores 和 composables 透過 `getPiniaServices()` 取得服務實例，而非依賴注入。

---

### 8. Vercel Edge Function

**路徑**: `api/auth.js`

在 Vercel 部署時，若設置了 `ACCESS_PASSWORD` 環境變數，`middleware.js` 會攔截所有請求，要求輸入密碼後才可訪問 Web UI。

---

### 關鍵依賴順序 (Web 環境)

```
dexie storage
  → preferenceService
    → languageService (templateLanguageService)
      → templateManager (需要 languageService 確定語言)
  → modelManager (ensureInitialized → 載入 env vars)
  → historyManager (依賴 modelManager 查詢 modelName)
modelManager + textAdapterRegistry
  → llmService
modelManager + llmService + templateManager + historyManager
  → promptService
```
