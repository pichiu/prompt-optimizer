# Stage 1 線上搜尋結果

## 搜尋摘要

### 搜尋 1: 專案概覽
**查詢**: "linshenkx prompt-optimizer GitHub project overview features"

**關鍵發現**:
- GitHub 倉庫：https://github.com/linshenkx/prompt-optimizer
- Chrome Web Store 插件：https://chromewebstore.google.com/detail/prompt-optimizer/cakkkhboolfnadechdlgdcnjammejlna
- Docker Hub：https://hub.docker.com/r/linshen/prompt-optimizer
- 有獨立的 Trendshift trending 指標，顯示此專案在 GitHub 社群有一定熱度
- ZRead 文件站：https://zread.ai/linshenkx/prompt-optimizer

---

### 搜尋 2: MCP Server 架構
**查詢**: "prompt-optimizer linshenkx MCP server architecture deepwiki 2025"

**關鍵發現**:
- DeepWiki 有完整的自動生成技術文件：https://deepwiki.com/linshenkx/prompt-optimizer
- DeepWiki 最後索引時間：2025年10月19日（反映相對最新的版本）
- Docker 環境中，MCP Server 與 Web 應用在同一個 container 中執行
- MCP Server 可透過 `/mcp` 路徑訪問（與 Web 同 port）
- DeepWiki 頁面列出更詳細的子模組，包含：
  - Variable Extraction and CodeMirror Integration：https://deepwiki.com/linshenkx/prompt-optimizer/3.6-variable-extraction-and-codemirror-integration
  - Context Editor and Conversation Manager：https://deepwiki.com/linshenkx/prompt-optimizer/3.7-context-editor-and-conversation-manager
  - Docker and Vercel Deployment：https://zread.ai/linshenkx/prompt-optimizer/20-docker-and-vercel-deployment

**架構描述 (來自 DeepWiki)**:
1. **Core Layer** (`@prompt-optimizer/core`)：平台無關的 LLM 整合、template 處理、資料管理
2. **UI Layer** (`@prompt-optimizer/ui`)：可重用的 Vue 組件
3. **Application Layer**：4 個部署目標 (web, desktop, extension, docker)，共用同一份 core + ui

**MCP 工具清單 (DeepWiki 描述 vs 實際程式碼)**:
- DeepWiki 說 2 個工具：optimize-user-prompt, optimize-system-prompt
- 實際 `mcp-server/src/index.ts` 有 3 個工具（加上 iterate-prompt）

---

### 搜尋 3: 評估與比較功能
**查詢**: "prompt-optimizer evaluation compare structured pipeline 2025"

**關鍵發現**:
- 未找到本專案專屬的 evaluation 架構文件
- 學術界研究顯示結構化 prompting 平均提升 6% 效能（arXiv:2511.20836）
- 本專案的 Structured Compare 功能（`evaluation/structured-compare-prompts.ts`）是一種 LLM-as-Judge 實作，讓 LLM 對兩份 prompt 的測試結果進行評分與比較
- 此設計遵循業界 "LLM-as-evaluator" 的標準模式

---

## 重要連結整理

| 資源 | 連結 |
|------|------|
| GitHub 主倉庫 | https://github.com/linshenkx/prompt-optimizer |
| Live Demo | https://prompt.always200.com |
| DeepWiki 文件 | https://deepwiki.com/linshenkx/prompt-optimizer |
| ZRead 文件 | https://zread.ai/linshenkx/prompt-optimizer |
| Chrome Extension | https://chromewebstore.google.com/detail/prompt-optimizer/cakkkhboolfnadechdlgdcnjammejlna |
| Docker Hub | https://hub.docker.com/r/linshen/prompt-optimizer |
| Vercel 部署指南 | https://zread.ai/linshenkx/prompt-optimizer/20-docker-and-vercel-deployment |

---

## 社群活躍度觀察

- GitHub Stars 數目在 Trendshift 可見（有進入 trending 指標）
- Chrome Web Store 有真實使用者數目（可從 README badge 確認）
- 專案有 CHANGELOG.md，顯示版本 v2.5.4 ~ v2.9.4 的更新紀錄，維護積極
- 有 `releases/` 目錄，包含中英文雙語的版本發布說明

---

## ⚠️ 未能在線上找到的資訊

- 官方 Discord/Slack 社群（README 未提及）
- 架構設計 ADR (Architecture Decision Records)
- 開發者 blog posts 或 conference talks
- Issues/Discussions 中的設計討論（需要 GitHub 直接存取）
