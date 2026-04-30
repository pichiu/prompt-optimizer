# Upstream 變更紀錄

> 本檔案記錄每次增量 trace 時 upstream `develop` 分支相對於上一次 trace 基準點的差異概覽。

---

## 2026-04-30 增量更新

### Base Commit 範圍

- 上次基準: `a9cbcd4` (2026-04-23)
- 本次基準: `3824b64` (2026-04-30)
- 範圍: `a9cbcd4..3824b64`

### 變更統計

```
180 files changed, 17387 insertions(+), 4501 deletions(-)
```

不含 `.trace/` 變更；包含原始碼、測試、e2e fixtures、release notes 等所有 upstream 檔案。

### Commit 列表（25 個 non-merge commits）

| Commit | 主題 |
|--------|------|
| `3824b64` | build(release): add v2.9.6 changelog entry |
| `2e30e6d` | build(release): bump version to 2.9.6 |
| `81674ea` | fix(ui): update favorite use assertions for optional options param |
| `f034bde` | feat(ui): enhance favorites with example editing, media management, and workspace apply |
| `2dc0b99` | test(e2e): refresh image generation replay fixture |
| `d7b3570` | test(ui): update favorite editor guard targets |
| `51df808` | feat(ui): expose favorite variables and examples |
| `4910e03` | fix(ui): clarify minimax api endpoint guidance |
| `d2e1473` | feat(core): refresh openai image provider models |
| `219af33` | feat(ui): add routed favorites page |
| `a1caf48` | refactor(ui): extract favorite library workspace |
| `d315c4d` | fix(core): clarify image prompt JSON wrapper handling |
| `6506c54` | fix(ui): preserve Garden favorite media fallback |
| `81ad5cc` | feat(ui): refine favorites workspace flow |
| `8dc3def` | build(release): fix v2.9.5 notes order |
| `79f835e` | fix(ci): relax no-change release notes sections |
| `f913ed0` | build(release): fix v2.9.5 notes order |
| `33849fe` | test(e2e): refresh DeepSeek VCR fixtures |
| `d1afce5` | build(release): bump version to 2.9.5 |
| `136dfac` | fix(ui): add missing parameter and import translations |
| `faeee9f` | feat(core): update DeepSeek provider for v4 models |
| `02010ad` | chore: ignore trellis workspace files |
| `08687ba` | fix(ui): clear content before Prompt Garden import |
| `89a8407` | feat(ui): add workspace clear content tool |
| `49ddd1c` | feat(core): add structured SOUL templates for OpenClaw and Hermes |

### 變更熱區（依目錄）

| 目錄 | 檔案數 | 性質 |
|------|--------|------|
| `packages/ui/src/components` | 31 | Vue 元件大改造（favorites workspace 拆解） |
| `packages/ui/tests/unit` | 24 | 對應元件單元測試 |
| `packages/ui/src/i18n` | 12 | 補齊 import / parameter 翻譯 |
| `packages/ui/src/stores` | 7 | favorites/composer/garden store 調整 |
| `packages/ui/src/composables` | 7 | 新增 favorite 相關 composables |
| `packages/ui/src/utils` | 4 | 新增 favorite-mode, favorite-reproducibility, external-data-loading |
| `packages/ui/src/router` | 4 | 新增 workspaceRoutes.ts，調整 guards/index/RootBootstrap |
| `packages/core/src/services` | 34 | image/llm adapters + image-optimize templates 全改、新增 SOUL templates |
| `packages/core/tests/unit` | 10 | DeepSeek + SOUL template 註冊測試 |
| `tests/e2e` | 10 | VCR fixture 刷新 + workspace-clear-content 新增 |
| `releases` | 4 | v2.9.5、v2.9.6 release notes |
| `scripts` | 2 | release-notes.js + 對應 unit test |
| `packages/extension/public` | 1 | manifest.json 版本 bump |
| `packages/desktop` | 1 | package.json |

### 新增檔案重點

**Core 端（新增 SOUL 結構化模板）**：

- `packages/core/src/services/template/default-templates/iterate/soul-iterate.ts` (+ `_en.ts`)
- `packages/core/src/services/template/default-templates/optimize/soul-hermes-compose.ts` (+ `_en.ts`)
- `packages/core/src/services/template/default-templates/optimize/soul-openclaw-compose.ts` (+ `_en.ts`)
- `packages/core/tests/integration/llm/deepseek-live.integration.test.ts`
- `packages/core/tests/unit/llm/deepseek-adapter.test.ts`
- `packages/core/tests/unit/template/soul-template-registration.test.ts`
- `packages/core/tests/unit/template/soul-template-manual-acceptance.md`

**UI 端（Favorites routed page 重構）**：

- `packages/ui/src/components/favorites/FavoritesPage.vue`
- `packages/ui/src/components/favorites/favorites-page-context.ts`
- `packages/ui/src/components/FavoriteDetailPanel.vue`
- `packages/ui/src/components/FavoriteEditorForm.vue`
- `packages/ui/src/components/FavoriteImportPanel.vue`
- `packages/ui/src/components/FavoriteLibraryWorkspace.vue`
- `packages/ui/src/components/FavoriteReproducibilityDisplay.vue`
- `packages/ui/src/components/FavoriteReproducibilityEditor.vue`
- `packages/ui/src/components/FavoriteWorkspaceListItem.vue`
- `packages/ui/src/components/common/WorkspaceUtilityMenu.vue`
- `packages/ui/src/components/app-layout/workspaceRouteSwitch.ts`
- `packages/ui/src/router/workspaceRoutes.ts`
- `packages/ui/src/utils/external-data-loading.ts`
- `packages/ui/src/utils/favorite-mode.ts`
- `packages/ui/src/utils/favorite-reproducibility.ts`

**測試**：

- `packages/ui/tests/unit/components/FavoritesPage.spec.ts` 等 11 個 favorites/route 相關 spec
- `tests/e2e/session-persistence/workspace-clear-content.spec.ts`

### 刪除檔案

- `packages/ui/src/components/FavoriteCard.vue`（被 `FavoriteWorkspaceListItem` 取代）

### 主要主題（依 commit 訊息歸類）

1. **Favorites Workspace 路由化重構**：將原本在 `BasicMode` / `ContextMode` 內嵌的 favorites 區塊抽離為獨立的路由化 page，並拆解為多個獨立元件。新增 `workspaceRoutes.ts`，favorites 變成第一級導覽項目。
2. **SOUL 結構化模板新增**：為 OpenClaw 與 Hermes 兩個外部系統（推測是 Asus 內部產品 / 框架）新增專用的結構化 prompt 優化與迭代模板，含中英文版本。
3. **DeepSeek v4 升級**：更新 DeepSeek adapter 以支援 v4 模型，並補齊 integration / unit test。
4. **OpenAI Image Provider 模型清單更新**：refresh 預設模型清單。
5. **Workspace Clear Content 工具**：新增清空工作區內容的工具，並補上 e2e session-persistence 測試。
6. **Image Optimize 模板批次調整**：全部 13 個 image-optimize 模板（中英文）內容修訂。
7. **Image Prompt JSON Wrapper 處理**：修正 image prompt 包 JSON wrapper 的邊界處理。
8. **i18n 翻譯補齊**：補齊 import / parameter 相關 locales。
9. **MiniMax API endpoint 提示文案**修正。
10. **Garden Favorite media fallback** 保留。
11. **Release / CI 改進**：v2.9.5、v2.9.6 release，relax no-change release notes sections，CI 修正。
12. **`.gitignore` 增補 trellis 工作區檔案**。

### 未列為「變更」的雜項

- `tests/e2e/fixtures/vcr/*.json`：VCR 錄製 fixtures，視為測試資產，不影響 trace 文件。
- `tests/e2e/fixtures/images/favorite-*.svg`：測試素材。
- `releases/v2.9.5.*.md`、`releases/v2.9.6.*.md`：release notes，CHANGELOG.md 主檔已涵蓋。

