# Stage 2.6 設定與環境分析

## 環境變數載入機制

### 優先順序（高到低）

```
runtime_config (Docker, window.runtime_config)
  > process.env (Node.js / Electron)
  > import.meta.env (Vite build-time)
```

**實作**: `packages/core/src/utils/environment.ts:262`  
函數 `getEnvVar(key)` 按上述優先順序查找。

---

## 環境變數類別

### 1. LLM API 金鑰（VITE_ 前綴）

| 變數名 | 用途 |
|--------|------|
| `VITE_OPENAI_API_KEY` | OpenAI |
| `VITE_GEMINI_API_KEY` | Google Gemini（含圖像）|
| `VITE_ANTHROPIC_API_KEY` | Anthropic Claude |
| `VITE_DEEPSEEK_API_KEY` | DeepSeek |
| `VITE_SILICONFLOW_API_KEY` | SiliconFlow |
| `VITE_ZHIPU_API_KEY` | 智谱 AI |
| `VITE_DASHSCOPE_API_KEY` | 阿里百炼 DashScope |
| `VITE_OPENROUTER_API_KEY` | OpenRouter |
| `VITE_MODELSCOPE_API_KEY` | 魔搭 ModelScope |
| `VITE_MINIMAX_API_KEY` | MiniMax |
| `VITE_CF_API_TOKEN` + `VITE_CF_ACCOUNT_ID` | Cloudflare Workers AI |
| `VITE_SEEDREAM_API_KEY` / `VITE_ARK_API_KEY` | Seedream / 火山方舟 |

### 2. 自定義模型（無限數量）

Pattern: `VITE_CUSTOM_API_{TYPE}_{suffix}`

```
VITE_CUSTOM_API_KEY_qwen3=...      # 必填
VITE_CUSTOM_API_BASE_URL_qwen3=... # 必填
VITE_CUSTOM_API_MODEL_qwen3=...    # 必填
VITE_CUSTOM_API_PARAMS_qwen3=...   # 選填，JSON 物件字串
```

`suffix` 規則：只允許 `[a-zA-Z0-9_-]`，最長 50 字元，不可用點號

掃描邏輯：`scanCustomModelEnvVars()` (`environment.ts:300`) 動態掃描所有 `VITE_CUSTOM_API_*` 環境變數，驗證後建立 model config。

### 3. Docker 部署存取控制

| 變數名 | 預設值 | 說明 |
|--------|--------|------|
| `ACCESS_USERNAME` | `admin` | HTTP Basic Auth 用戶名 |
| `ACCESS_PASSWORD` | 無 | 設定後啟用密碼保護 |

Docker 啟動時由 `docker/generate-auth.sh` 腳本生成 nginx `.htpasswd` 檔案。

### 4. MCP Server 設定

| 變數名 | 預設值 | 說明 |
|--------|--------|------|
| `MCP_DEFAULT_MODEL_PROVIDER` | 第一個可用 | 首選 LLM Provider |
| `MCP_HTTP_PORT` | `3000` | HTTP mode port |
| `MCP_LOG_LEVEL` | `debug` | 日誌等級 |
| `MCP_DEFAULT_LANGUAGE` | `zh` | 模板語言 |

### 5. Docker / Nginx 設定

| 變數名 | 預設值 | 說明 |
|--------|--------|------|
| `NGINX_PORT` | `80` | nginx 監聽 port |

### 6. 平台識別

| 變數名 | 說明 |
|--------|------|
| `VITE_APP_PLATFORM` | `electron` 代表桌面應用，未設定則自動偵測 |
| `VITE_LOCAL_DEV` | `true` 代表本地開發環境 |
| `VITE_VERCEL_DEPLOYMENT` | `true` 代表 Vercel 部署（啟用 Analytics）|

### 7. Electron 更新設定

| 變數名 | 說明 |
|--------|------|
| `GITHUB_REPOSITORY` | `owner/repo` 格式，指定更新檢查的 repo |
| `DEV_REPO_OWNER` + `DEV_REPO_NAME` | 分開設定 owner 和 repo |

---

## Docker Runtime Config

**路徑**: `packages/web/public/config.js`（被 nginx 動態生成覆蓋）

在 Docker 部署時，`docker/generate-config.sh` 腳本在容器啟動時讀取環境變數，生成 `config.js`：
```javascript
window.runtime_config = {
  OPENAI_API_KEY: "...",   // 注意：無 VITE_ 前綴
  GEMINI_API_KEY: "...",
  ...
}
```

`getEnvVar('VITE_OPENAI_API_KEY')` 先查 `window.runtime_config['OPENAI_API_KEY']`（去掉 `VITE_` 前綴），這樣 Docker 環境就可以在不重建鏡像的情況下動態注入 API 金鑰。

---

## 使用者設定儲存 (PreferenceService)

**路徑**: `packages/core/src/services/preference/service.ts`

儲存在 Dexie/localStorage（Web）或 FileStorage（Electron）中，key 為常數（`packages/core/src/constants/storage-keys.ts`）：

| Storage Key | 用途 |
|-------------|------|
| `prompt_history` | 優化歷史記錄 |
| `model_configs` | 模型設定 |
| `template_list` | 自訂模板 |
| `preferences` | 使用者偏好 |
| `context_list` | context 清單 |
| `favorite_list` | 收藏清單 |
| `startup_repair_report` | 啟動修復報告 |

`PreferenceService` 支援 `get/set/delete` 操作，以 JSON 格式儲存任意資料。

---

## Model 設定載入流程

**路徑**: `packages/core/src/services/model/defaults.ts:60`

`getDefaultTextModels()` 在 ModelManager 初始化時呼叫：
1. 遍歷 `PROVIDER_ENV_KEYS` 中的所有 Provider
2. 對每個 Provider，呼叫 `registry.getAdapter(providerId).getProvider()` 取得 metadata
3. 讀取對應環境變數（如 `VITE_OPENAI_API_KEY`）
4. 若 API key 存在，建立一個 `TextModelConfig` 並啟用 (`enabled: true`)
5. 額外掃描 `VITE_CUSTOM_API_*` 環境變數，動態建立自定義模型

**注意**: 環境變數中的 API key 只是預設值，使用者可在 UI 的「模型管理」中覆蓋或新增模型。使用者設定儲存在 Dexie，優先於環境變數。

---

## Feature Flags

**此專案無正式 feature flag 機制**。功能開關透過：
1. 環境變數（如 `VITE_VERCEL_DEPLOYMENT`）
2. 服務是否初始化（如 `evaluationService?` 可選服務）
3. 平台偵測（如 `isRunningInElectron()`）

---

## Secrets 管理

| 環境 | 管理方式 |
|------|----------|
| 本地開發 | `.env.local` 檔案（git ignored） |
| Vercel | Vercel Dashboard 環境變數 |
| Docker | `docker run -e` 或 `docker-compose.yml` environment section |
| Electron | `.env.local` 檔案（放在 exe 旁邊）或應用內設定 |

**注意**: `VITE_` 前綴的環境變數在 Vite build-time 被靜態替換到 JS bundle 中。若使用 Docker runtime config，環境變數在容器啟動時注入，**不**需要重建鏡像。

---

## 設定驗證

自定義模型設定有完整的驗證邏輯 (`environment.ts:66`)：
- suffix 格式驗證（正則）
- API key 長度警告
- baseURL 格式驗證（URL 物件解析）
- 禁止在 `PARAMS` 中覆蓋 `model`、`messages`、`stream`

Zod schema 用於評估服務的 LLM 輸出驗證（`evaluation/types.ts`）。
