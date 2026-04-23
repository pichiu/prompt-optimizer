# Stage 2.3 核心領域邏輯分析

## 「心臟」在哪裡？

本專案有兩個核心 abstraction：
1. **Template System** — 決定「如何把使用者的 prompt 轉成 LLM 輸入」
2. **Adapter Pattern** — 決定「如何與各家 LLM Provider 通訊」

這兩者的組合，就是 prompt 優化的全部機制。

---

## 核心 Abstraction 1：Template System

### Template 資料結構

**路徑**: `packages/core/src/services/template/types.ts:49`

```typescript
interface Template {
  id: string;                              // 唯一識別碼，如 'general-optimize'
  name: string;                            // 顯示名稱
  content: string | MessageTemplate[];     // 兩種格式
  metadata: TemplateMetadata;
  isBuiltin?: boolean;
}

interface TemplateMetadata {
  version: string;
  templateType: 'optimize' | 'userOptimize' | 'iterate' | 'evaluation' | ...  // 16 種
  language?: 'zh' | 'en';
}
```

**兩種 content 格式**:
1. **Simple** (`string`): 純文字 system prompt，使用者 prompt 直接作為 user message
2. **Advanced** (`MessageTemplate[]`): message 陣列，支援 Mustache 變數替換

### TemplateProcessor

**路徑**: `packages/core/src/services/template/processor.ts:45`

設計模式：**靜態工廠方法**，所有方法為 `static`

```typescript
class TemplateProcessor {
  static processTemplate(template, context): Message[] {
    // Simple template → [{role:'system', content}, {role:'user', content:originalPrompt}]
    // Advanced template → Mustache.render() 每個 MessageTemplate
  }
  
  static createExtendedContext(baseContext, customVariables?, conversationMessages?): TemplateContext
  static formatConversationAsText(messages): string  // 格式化多輪對話
  static formatToolsAsText(tools): string             // 格式化 Function Calling tools
  static processConversationMessages(messages, variables?): Message[]
}
```

**Mustache 變數系統** (template 中可使用的佔位符):

| 變數 | 來源 | 說明 |
|------|------|------|
| `{{originalPrompt}}` | OptimizationRequest | 待優化的 prompt |
| `{{lastOptimizedPrompt}}` | 迭代請求 | 上一版本 |
| `{{iterateInput}}` | 迭代請求 | 使用者的迭代指令 |
| `{{conversationContext}}` | Pro mode | 格式化的對話歷史 |
| `{{toolsContext}}` | Pro mode | Function Calling 定義 |
| `{{#var}}...{{/var}}` | Mustache 條件 | 條件渲染 |
| `{{自定義變數名}}` | variables map | 使用者定義的批量替換變數 |

**Built-in helper** (`processor.ts:172`):
```
{{#helpers.toJson}}{{someVar}}{{/helpers.toJson}}
```
→ 將變數值 JSON.stringify，用於需要傳遞 JSON 給 LLM 的場景

### 內建 Template 目錄

**路徑**: `packages/core/src/services/template/default-templates/`

主要 template 類別：

| 目錄 | 用途 |
|------|------|
| `optimize/` | System prompt 優化（通用、分析型、輸出格式型） |
| `user-optimize/` | User prompt 優化（基礎、規劃、專業） |
| `iterate/` | 迭代優化 |
| `evaluation/` | 結果評估（basic/pro，支援多種評估類型） |
| `evaluation-structured-compare/` | 結構化比較評估（pair-judge + synthesis） |
| `evaluation-rewrite/` | 評估驅動的重寫 |
| `image-optimize/` | 圖像 prompt 優化（text2image/image2image） |
| `variable-extraction/` | 自動提取 prompt 中的變數 |
| `variable-value-generation/` | 生成測試用的變數值 |

每個 template 有中英文兩個版本（`xxx.ts` 和 `xxx_en.ts`）。

---

## 核心 Abstraction 2：Adapter Pattern

### 設計模式：Template Method + Registry

**Abstract Base**: `packages/core/src/services/llm/adapters/abstract-adapter.ts:24`

```typescript
abstract class AbstractTextProviderAdapter implements ITextProviderAdapter {
  // 公共邏輯（子類不需重複）
  sendMessage(messages, config): Promise<LLMResponse>
  sendMessageStream(messages, config, callbacks): Promise<void>
  validateMessages(messages): void
  processThinkTags(content: string): { content, reasoning }  // 處理 <think> 標籤
  
  // 子類必須實作
  abstract getProvider(): TextProvider
  abstract getModels(): TextModel[]
  protected abstract doSendMessage(messages, config): Promise<LLMResponse>
  protected abstract doSendMessageStream(messages, config, callbacks): Promise<void>
}
```

### Registry 機制

**路徑**: `packages/core/src/services/adapters/abstract-registry.ts`

`TextAdapterRegistry` 繼承 `AbstractAdapterRegistry<ITextProviderAdapter, TextProvider, TextModel, TextModelConfig>`

Registry 在建構時 (`initializeAdapters()`) 實例化所有 13 個 adapter 並 `register()` 到 `Map<string, adapter>`。

使用時：`registry.getAdapter(providerId)` → 取出對應 adapter，例如 `'openai'` → `OpenAIAdapter`。

### 各 Adapter 的 SDK 選擇

| Adapter | SDK | 說明 |
|---------|-----|------|
| `OpenAIAdapter` | `openai` npm SDK | OpenAI 官方 |
| `OpenAICompatibleAdapter` | `openai` npm SDK | Ollama、自定義服務，修改 baseURL |
| `DeepseekAdapter` | `openai` npm SDK | DeepSeek 兼容 OpenAI API |
| `SiliconflowAdapter` | `openai` npm SDK | SiliconFlow 兼容 OpenAI API |
| `AnthropicAdapter` | `@anthropic-ai/sdk` | Anthropic 官方 |
| `GeminiAdapter` | `@google/genai` | Google 官方 |
| `ZhipuAdapter` | `openai` npm SDK | 智谱 AI，部分兼容 |
| `DashScopeAdapter` | `openai` npm SDK | 阿里百炼，OpenAI 兼容路由 |
| `OllamaAdapter` | `openai` npm SDK | Ollama local，修改 baseURL |
| `CloudflareAdapter` | 原生 fetch | Cloudflare Workers AI |

**共同模式**：大多數 adapter 都使用 `openai` npm SDK 搭配自定義 `baseURL`，只有 Anthropic 和 Gemini 使用專屬 SDK，Cloudflare 使用 fetch。

---

## 核心 Abstraction 3：PromptService 作為協調者

**路徑**: `packages/core/src/services/prompt/service.ts`

`PromptService` 的角色是**協調者 (Orchestrator)**，自身不包含業務邏輯，而是：
1. 接受 UI 層的請求
2. 向 `TemplateManager` 取得 template
3. 用 `TemplateProcessor` 把 template + context 渲染成 messages
4. 把 messages 交給 `LLMService` 發送
5. 把結果回傳（streaming 透過 callbacks）

**依賴注入**: 所有依賴透過建構子注入，`PromptService` 不 new 任何東西。

---

## 核心 Abstraction 4：Storage 抽象層

**路徑**: `packages/core/src/services/storage/`

Strategy pattern：

```typescript
interface IStorageProvider {
  getItem(key: string): Promise<string | null>
  setItem(key: string, value: string): Promise<void>
  removeItem(key: string): Promise<void>
  clear(): Promise<void>
  keys(): Promise<string[]>
}

// 實作：
LocalStorageProvider  → window.localStorage
DexieStorageProvider  → IndexedDB (透過 Dexie ORM)
MemoryStorageProvider → Map<string, string> (測試用)
FileStorageProvider   → Node.js fs (Electron)
```

`StorageAdapter` 包裝 `IStorageProvider`，提供 JSON 序列化/反序列化和 CAS (Compare-And-Swap) 原子更新：

```typescript
class StorageAdapter {
  async updateData<T>(key: string, updater: (current: T | null) => T): Promise<T>
  // 使用 read-modify-write 模式防止並發衝突
}
```

---

## 核心 Abstraction 5：Electron IPC Proxy 模式

**設計動機**: Electron 安全模式下，渲染進程不能直接存取 Node.js API。

**實作方式**:
- 主進程：`ipcMain.handle('llm:sendMessage', async (event, ...args) => {...})`
- Preload：`contextBridge.exposeInMainWorld('electronAPI', { sendMessage: (...) => ipcRenderer.invoke(...) })`
- 渲染進程：`new ElectronLLMProxy()` 的方法透過 `window.electronAPI.xxx()` 呼叫

每個 core 服務都有對應的 `electron-proxy.ts`，實作完全相同的 interface，對 UI 層完全透明。

---

## HistoryManager 的鏈式版本管理

**路徑**: `packages/core/src/services/history/manager.ts`

設計：每次優化建立一個新的 `PromptChain`（有唯一 `chainId`）。每次迭代在同一個 chain 下新增一個 `version`。

```
Chain (chainId=abc)
  ├── version 1: 原始優化
  ├── version 2: 第一次迭代
  └── version 3: 第二次迭代（目前版本）
```

最大保留 50 條記錄 (`manager.ts:17`)，超過自動刪除最舊的。

---

## EvaluationService 的 Structured Compare

**路徑**: `packages/core/src/services/evaluation/service.ts`

**Structured Compare** = LLM-as-judge 多輪比較：

```
1. 準備測試案例（test cases）
2. For each pair of prompts (A vs B):
   → 呼叫 pair-judge template，LLM 判斷哪個更好
   → 輸出 JSON: { winner: 'A'|'B'|'tie', reasoning: '...' }
3. synthesis:
   → 彙整所有 pair judgements
   → LLM 輸出最終排名和洞察
4. jsonrepair() 修復 LLM 可能輸出的格式問題
```

使用 `Zod` 做 schema 驗證確保 LLM 輸出格式正確 (`evaluation/types.ts`)。
