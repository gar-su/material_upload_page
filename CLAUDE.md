# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目性质

单文件 HTML 交互原型，无构建、无依赖安装、无测试、无包管理。`index.html` 是唯一实现，中文需求文档是唯一规格来源。

预览：直接用浏览器打开 `index.html`，或 `python3 -m http.server`。页面依赖 CDN（Tailwind CDN、Font Awesome），需联网。

验证改动：走 DOM 断言（headless Chrome + CDP），不使用截图——全局规范已定义具体做法。本仓库无 Python 代码，全局的 mypy/ruff/pyright 后置校验不适用。

## index.html 架构

### 页面切换（顶部 tab）

`[data-page]` tab 与 `#page-batch`（批量素材库）、`#page-ai`（AI素材库）通过 `pageEls` 映射，切换靠设置 `hidden` 属性。新增页面需同时改 `pageEls` 对象和 DOM。

顶部前四个 tab（内部/外部/TT/海外素材库）无 `data-page`，是**无行为占位**，不要为它们补逻辑。

### 弹窗约定

五个弹窗 `#uploadModal` / `#dramaModal` / `#unboundModal` / `#dupModal` / `#syncModal`：

- 开合一律用 `element.hidden = true/false`（**不是** `classList`）。
- 关闭按钮用 `[data-close-<modal>]` 属性批量绑定，新增关闭入口沿用该属性。

**关键陷阱**：`.lib-page` 与 `.modal-mask` 用 `@apply flex` 定义，会盖掉浏览器默认的 `[hidden]{display:none}`，因此 `<style type="text/tailwindcss">` 中必须显式补 `[hidden] { display: none }`。任何新增的、`@apply` 了 `display` 的组件类，都要照此补一条，否则 `hidden` 会静默失效。

### CSS 约定

Tailwind CDN + `<style type="text/tailwindcss">` 的 `@layer utilities` / `@layer components`：

- 自定义色板与阴影在 `tailwind.config`（`primary` / `borderGray` / `bgPage` / `tagOrange` / `tagGreen` / `danger`、`shadow-modal`）——不要在 HTML 里写死十六进制色值。
- 组件类（`.form-label` / `.select-full` / `.btn-footer` / `.drama-bind` / `.ai-card` 等）集中在 `@layer components`，HTML 中复用类名，不重复堆 utility。

### JS 约定

原生 ES5 风格（`var` + `function`，无框架、无模块），全部写在 `</body>` 前的单个 `<script>` 内，直接执行（无 `DOMContentLoaded`）。保持此风格：不引入构建、不改用 `const`/箭头函数混写、不拆分文件。

状态只有两处：

- `bindMap`：`data-row`（文件行的行号字符串）→ `{ playId, playName, lang, drama }`，短剧绑定的唯一数据源。
- `aiSelected`：AI素材库当前勾选的 checkbox 数组，每次由 `updateAiSelection()` 从 DOM 重算。

另有 `uploadRowSeq` / `uploadFileCursor` 两个计数器，只服务于动态追加文件行，不承载业务状态。素材名称的冲突判定不设状态，一律由 `collectDupRows()` 从 DOM 现算。

数据属性是 DOM 与逻辑之间的契约：

- `.drama-bind[data-row]` —— 索引 `bindMap` 的键。
- `.ai-pick[data-name]` —— 素材原始文件名**含扩展名**，展示前用 `stripExt()` 去扩展名。
- `[data-full-name]` —— 配合全局 `mouseover` 监听与 `#nameTip` 浮层，仅当 `scrollWidth > clientWidth`（实际被截断）时显示完整文本。
- `.file-name-input` + `.file-ext` —— 上传文件行的名称，主名可编辑、扩展名固定；整名由 `rowFullName()` 拼出，是查重的输入。

## Mock 数据

四组前端写死数据，是接入真实接口时的**唯一替换点**，交互逻辑无需改动：

| 变量 | 位置 | 说明 |
| ---- | ---- | ---- |
| `MOCK_PLAYS` / `MOCK_LANGUAGES` / `MOCK_DRAMAS` | 脚本顶部 | 剧本 / 语种 / 短剧，`MOCK_DRAMAS` 的 key 为 `剧本ID_语种` |
| `MOCK_AI_FOLDERS` | AI素材库段落 | 同步目标文件夹，`initSyncFolder()` 填充下拉 |
| `MOCK_LIBRARY_NAMES` | 脚本顶部 | 全库已有素材名（含扩展名），同名拦截的查重数据源 |
| `MOCK_UPLOAD_FILES` | 脚本顶部 | 模拟"加入列表"的文件，点击拖拽区 / 上传按钮逐个追加 |

`MOCK_AI_FOLDERS` 的取值与 `#page-batch` 左侧素材树中的文件夹名对应，改一侧时注意另一侧。

## 需求文档 ↔ 代码映射

| 文档 | 覆盖内容 | 代码入口 |
| ---- | ---- | ---- |
| `需求文档.md` | 上传弹窗短剧绑定、未绑定拦截、同名素材拦截 | `#uploadModal` / `#dramaModal` / `#unboundModal` / `#dupModal`、`bindMap`、`confirmDrama()`、`refreshDramaOptions()`、`onFinishUpload()`、`isDupName()` / `collectDupRows()` / `refreshDupMarks()`、`appendFileRow()` |
| `AI素材库需求文档.md` | AI素材库 tab、批量命名与同步 | `#page-ai` / `#syncModal`、`filterAiCards()`、`updateAiSelection()`、`findSeedSegment()` / `buildSyncedName()` / `stripExt()` |
| `未绑定素材自动绑定需求文档.md` | 后端定时任务补绑 | 无前端改动（明确要求 upload 页零改动） |
| `重名素材拦截需求文档.md` | 素材名全库唯一的判定口径、两个入口（上传页 / AI素材库同步）的拦截与查名接口 | 同上两处，判定口径的唯一权威来源 |

文档中的「待确认项（Q1–Q16）」「范围外」「不涉及」章节是范围边界，改动超出已验证条目时先回到文档确认。

代码中无 handler 的元素（如 `#page-batch` 的 `.del-btn`、批量操作按钮组、`#page-batch` 筛选栏）是原型占位，未被任何 JS 引用；不要因为「看起来该有行为」就顺手补上。
