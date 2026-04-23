# Stage 2.2 Data Flow 分析

## 代表性用例：「優化 System Prompt（流式）」

**使用者操作**: 在 Basic System 模式輸入一段 prompt → 選擇模型 → 點擊「優化」按鈕

---

## 完整流程追蹤

```
[User] 點擊「優化」按鈕
  ↓
[UI Layer - Component]
  PromptPanel.vue 或 Workspace 組件
  觸發 handleOptimizePrompt()

[UI Layer - Composable]
  packages/ui/src/composables/prompt/usePromptOptimizer.ts:104
  handleOptimizePrompt()
    1. 驗證 state.prompt 不為空
    2. 驗證 optimizeModel.value 存在
    3. isOptimizing = true (防止重複點擊)
    4. 清空 state.optimizedPrompt
    5. 構建 OptimizationRequest:
       {
         optimizationMode: 'system',
         targetPrompt: state.prompt,
         templateId: currentTemplate.id,
         modelKey: optimizeModel.value,
         contextMode: contextMode.value
       }
    6. 呼叫 promptService.optimizePromptStream(request, callbacks)

[Core Layer - PromptService]
  packages/core/src/services/prompt/service.ts:494
  optimizePromptStream()
    1. validateOptimizationRequest(request) — 驗證 targetPrompt, modelKey
    2. modelManager.getModel(modelKey) — 取得模型設定
    3. resolveOptimizationMessages(request):
       a. templateManager.getTemplate(templateId || defaultId)
       b. 構建 TemplateContext:
          { originalPrompt, optimizationMode, contextMode, ... }
       c. TemplateProcessor.processTemplate(template, context)
          → Mustache.render() 渲染 template 中的 {{originalPrompt}} 變數
          → 返回 Message[] 陣列 [{role:'system', content:...}, {role:'user', content:...}]
    4. llmService.sendMessageStream(messages, modelKey, callbacks)

[Core Layer - LLMService]
  packages/core/src/services/llm/service.ts:125
  sendMessageStream()
    1. validateMessages(messages)
    2. modelManager.getModel(modelKey) — 再次確認模型存在
    3. validateModelConfig(modelConfig) — 確認已啟用
    4. registry.getAdapter(modelConfig.providerMeta.id)
       → 例如 OpenAIAdapter、GeminiAdapter、AnthropicAdapter
    5. prepareRuntimeConfig(modelConfig) — 合併 API key、base URL、參數覆蓋
    6. adapter.sendMessageStream(messages, runtimeConfig, callbacks)

[External - LLM Provider SDK]
  (以 OpenAI 為例) packages/core/src/services/llm/adapters/openai-adapter.ts
  sendMessageStream()
    → openai.chat.completions.create({stream: true, ...})
    → 迭代 stream chunks
    → callbacks.onToken(chunk.delta.content)    ← 每個 token 回傳
    → callbacks.onReasoningToken(...)            ← 推理 token（若有）
    → callbacks.onComplete(LLMResponse)          ← 完成時

[回到 PromptService callbacks]
  onToken: (token) → 直接 pass-through 到 UI callbacks
  onComplete: (response)
    → validateResponse(response.content, prompt)  # 確認不為空
    → 呼叫 UI 傳入的 onComplete

[回到 UI Composable callbacks]
  onToken: (token) → state.optimizedPrompt += token  ← 實現串流顯示
  onReasoningToken: (token) → state.optimizedReasoning += token
  onComplete: async () →
    1. historyManager.createNewChain({
         id: uuidv4(),
         originalPrompt: state.prompt,
         optimizedPrompt: state.optimizedPrompt,
         type: 'optimize',
         modelKey, templateId, timestamp
       })
       → 儲存到 Dexie (IndexedDB) / FileStorage (Electron)
    2. state.currentChainId = newRecord.chainId
       state.currentVersions = newRecord.versions
       state.currentVersionId = newRecord.currentRecord.id
    3. toast.success('優化成功')
  onError: (error) →
    toast.error(error.message)
    isOptimizing = false
```

---

## 各層職責摘要

| 層 | 職責 | 關鍵檔案 |
|----|------|----------|
| Component / Composable (UI) | 驗證輸入、管理 loading 狀態、token 累積顯示、onComplete 後存歷史 | `usePromptOptimizer.ts:104` |
| PromptService (Core) | 選擇 template、建構 messages、協調 LLMService | `prompt/service.ts:494` |
| TemplateProcessor (Core) | Mustache 渲染，將 template + context → Message[] | `template/processor.ts:49` |
| LLMService (Core) | 解析模型設定、取得 adapter、發送請求 | `llm/service.ts:125` |
| Provider Adapter (Core) | 呼叫對應 SDK（openai/anthropic/genai）實作 streaming | `llm/adapters/*.ts` |
| Storage (Core) | 優化完成後透過 historyManager 持久化到 IndexedDB/File | `history/manager.ts` |

---

## 歷史記錄資料結構

```typescript
// packages/core/src/services/history/types.ts
interface PromptRecord {
  id: string         // uuid
  chainId: string    // 同一次優化的所有迭代版本共用
  version: number    // 1-based，chain 內版本號
  type: PromptRecordType  // 'optimize' | 'userOptimize' | 'iterate' | ...
  originalPrompt: string
  optimizedPrompt: string
  modelKey: string
  modelName?: string
  templateId?: string
  timestamp: number
  metadata?: Record<string, unknown>
}
```

**PromptChain** = 一次優化的版本鏈（同 chainId），記錄：原始 + 每次迭代的結果

---

## 迭代 Prompt 流程差異

**迭代** = 在已優化的 prompt 上繼續改進

```
handleIteratePrompt({ originalPrompt, optimizedPrompt, iterateInput })
  → promptService.iteratePromptStream(...)
    → templateManager.getTemplate('iterate')
    → TemplateProcessor.processTemplate:
        {{lastOptimizedPrompt}} = optimizedPrompt
        {{iterateInput}} = 使用者的迭代指令
    → llmService.sendMessageStream(...)
  → onComplete: historyManager.addIteration({ chainId, version: n+1, ... })
```

迭代模板**強制要求** message array 格式（不支援 simple string template），目的是確保 `{{lastOptimizedPrompt}}` 和 `{{iterateInput}}` 變數能被正確替換（`prompt/service.ts:323-330`）。

---

## 測試 Prompt 流程

```
testPromptStream(systemPrompt, userPrompt, modelKey, callbacks)
  → messages = [{ role:'system', content:systemPrompt }, { role:'user', content:userPrompt }]
  → llmService.sendMessageStream(messages, modelKey, callbacks)
```

注意：測試不儲存歷史記錄（設計決策，`prompt/service.ts:1031-1033`）。

---

## 評估流程 (簡述)

```
EvaluationService.evaluate(request)
  → 根據 evaluationType 選擇 template:
      'result'      → evaluation-result template
      'compare'     → evaluation-compare template
      'structured-compare' → 多輪 pair-judge + synthesis
  → LLM 評估 prompt 質量，輸出 JSON 格式的評分
  → jsonrepair() 修復可能的 JSON 格式問題
  → 返回 EvaluationScore
```

Structured Compare 是最複雜的評估模式：使用 LLM-as-judge 方式，逐對比較多個 prompt 測試結果，最後 synthesis 得出排名。
