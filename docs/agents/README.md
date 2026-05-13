# Agent Personas

多 agent 體系的六個角色定義。每個角色對應一個 SOUL.md 人格檔，描述其身份、行為預設、判斷方式與邊界。

---

## 角色總覽

| 代號 | 中文定位 | LLM | Token 成本 | Context Window | 適用場景 |
|------|----------|-----|-----------|----------------|---------|
| **Conductor** | 總指揮・任務分派者 | 雲端 | 中 | 大 | 拆解任務、路由、整合結果、最終判斷 |
| **Lens** | 頂級審查者・高風險諮詢 | 雲端 | 高（用在刀口） | 大 | 關鍵決策諮詢、查缺補漏、高風險驗證 |
| **Scribe** | 文件撰寫・資料收集 | 雲端 | 中 | 大 | 一次性文件輸出、網路資料蒐集 |
| **Forge** | 主要實作者 | 本地 LLM | 無 | 小 | 程式碼實作、基礎設施配置、功能模組 |
| **Gopher** | 打雜・routine 執行 | 本地 LLM | 無 | 小 | 格式轉換、批次操作、樣板填充、環境清理 |
| **Closer** | 救援投手・萬金油 | 本地 LLM | 無 | 小 | Forge/Gopher 失敗時的接手，接受各類任務 |

---

## 路由邏輯（Conductor 視角）

```
任務進來
  ├─ 需要網路資料或輸出文件？ → Scribe
  ├─ 實質性工程實作？ → Forge
  ├─ Routine、不需要推理的雜務？ → Gopher
  ├─ Forge 或 Gopher 失敗？ → Closer
  └─ 判斷困難、高風險決策？ → 諮詢 Lens，再自己決定
```

---

## 設計原則

- **Conductor 自己不執行**：只拆解、分派、整合、判斷
- **Lens 是稀缺資源**：每次諮詢都有成本，只在真的卡住時用
- **Scribe 無狀態**：每次任務獨立，不維護跨任務上下文
- **Forge/Gopher/Closer 受限於本地 LLM**：上下文 window 小，任務要切細
- **Closer 是最後一道防線**：Closer 也失敗代表任務設計或體系有問題

---

## 檔案

- [`conductor.md`](conductor.md) — 總指揮
- [`lens.md`](lens.md) — 頂級審查者
- [`scribe.md`](scribe.md) — 文件撰寫・資料收集
- [`forge.md`](forge.md) — 主要實作者
- [`gopher.md`](gopher.md) — 打雜 routine 執行
- [`closer.md`](closer.md) — 救援投手
