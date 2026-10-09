# Design

## Context

动机见 `proposal.md - Why`；行为契约见 `specs/web-ui-i18n/spec.md`。这里只记录塑造方案的真实现状与约束。

**前端现状**（`serve/web/`）：无框架、无构建步骤、无 `package.json`、无 JS 测试框架。三个文件构成界面：

- `index.html`（223 行）——静态骨架，约 80 处英文文案，含 `title` / `aria-label` / `placeholder` 属性。`<html lang="en" data-theme="light">`，`<head>` 里已有一段内联脚本在首屏前决定主题（读 `localStorage` 的 `strata.theme`，否则跟随系统）。
- `app.js`（980 行）——顶层 `const $ = (id) => …`，脚本在 `</body>` 前加载并**立即执行**（`$("theme-btn").onclick = …` 在顶层）。约 120 处动态文案，绝大多数以模板字符串拼接，例如 `app.js:186` 的 `Reading prompt · ${pct}%`、`app.js:204` 的 `Since ${since}: ${fmt(t.requests)} requests · …`。
- `tokens.css` / `components.css` / `app.css`——`--st-font:"Outfit",system-ui,…`；Outfit 的两个 `@font-face` 的 `unicode-range` 只覆盖拉丁，汉字会自动落到系统回退字体。

**服务端现状**（`serve/server.py:1526`）：`/web/<name>` 只接受**扁平文件名**（`"/" in name` 直接 404）且扩展名限 `.css` / `.js` / `.svg`。`/` 返回 `index.html`。`serve/test_server.py:810` 的 `test_page_and_files` 逐个断言这些资源，`test_only_the_app_files_are_served` 断言穿越与越界文件 404。

**存储现状**：`store.get/set` 用 `strata.` 前缀存 JSON（`strata.chat`、`strata.sampling`、`strata.apikey`），`strata.theme` 单独用裸字符串。聊天历史里 `m.meta` 存的是**已渲染成英文的字符串**（`app.js:749`），不是结构化数据。

## Goals / Non-Goals

**Goals:**

- 一次改动覆盖全部界面文案，不留下"半中半英"的死角；缺词条有确定回退。
- 语言状态与已有的主题状态同构（同样的存储约定、同样的首屏前决策、同样的即时切换），不引入第二套机制。
- 服务端零改动：不新增路由、不改白名单、不新增网络请求。
- 词典完整性可被现有 pytest 套件自动检查。

**Non-Goals:**

- 不做服务端响应（错误信息、`/status` 文本）的本地化——已与用户确认，服务端不动。
- 不做繁体中文（`zh-Hant`）、不做第三种语言、不做语言文件的运行时加载。
- 不做翻译记忆、不做外部 i18n 库、不做构建期提取工具。
- 不改聊天历史的存储结构以外的任何持久化格式（`strata.sampling` / `strata.apikey` 不变）。

## Decisions

### D1：词典内嵌在 `serve/web/i18n.js`，不用 JSON 文件 + fetch

`i18n.js` 里 `const STRATA_I18N = {en: {…}, zh: {…}}`，随 `<script src="web/i18n.js">` 一次性加载。

- 为什么不用 `serve/web/locales/zh.json`：`/web/` 白名单拒绝带 `/` 的名字，也拒绝 `.json`，要走这条路必须改 `serve/server.py`——与"服务端零改动"冲突。
- 为什么不用扁平的 `i18n.zh.json`（改白名单加 `.json`）：仍是服务端改动，且多一次异步请求，带来"文案还没到就渲染"的竞态与首屏闪英文（spec 的"首屏不闪英文"）。
- 代价：中文词典即使用不上也会下载。约 15–20 KB 量级，本地同源，可忽略。

### D2：`index.html` 保留英文原文 + `data-i18n` 标记，运行时替换

```html
<button … id="new-btn" data-i18n-title="btn.newchat" title="New chat" aria-label="New chat">
<span id="pill-text" data-i18n="pill.connecting">Connecting&hellip;</span>
```

约定四个属性：`data-i18n`（textContent）、`data-i18n-title`、`data-i18n-aria`、`data-i18n-placeholder`。`applyLang()` 遍历 `[data-i18n],[data-i18n-title],[data-i18n-aria],[data-i18n-placeholder]` 逐个赋值。

- 为什么不在 HTML 里留空由 JS 填：词典缺词条时就是一片空白，违反 spec 的"缺词条回退英文"。保留英文原文使回退是**结构性**的，不依赖词典完整性。
- 为什么不把骨架改成 JS 生成：diff 巨大、可读性差、且首屏依赖脚本执行。
- 注意：`data-i18n` 用 `textContent` 赋值，所以 HTML 实体（`&hellip;`、`&ndash;`）在词典里要写成真实字符（`…`、`–`）。

### D3：`t(key, params)` 用整句模板 + 占位符，禁止拼接翻译片段

`t("pill.reading", {pct: 42})`，词典值 `en: "Reading prompt · {pct}%"` / `zh: "正在读取提示 · {pct}%"`。

- 为什么：现有代码大量在 JS 里拼接（`app.js:204` 的累计统计行有 5 个片段）。中文语序与英文不同，逐片段翻译会产出病句。整句模板把语序交给词典。
- 复数：`1 tool call` / `2 tool calls` 用 `t("chat.toolcalls", {n})` 加词典内的 `en` 单复数两键（`_one` / `_other`），中文只用 `_other` 一键。
- 分隔符 `·` 与单位（`t/s`、`GB`、`°C`）不进词典，保持原样——它们是符号不是文案。
- 键名：扁平点号，按界面区域分组（`tab.*`、`pill.*`、`mon.*`、`req.*`、`about.*`、`drawer.*`、`toast.*`、`chat.*`、`mcp.*`、`export.*`）。约 200 键 × 2 语言。

### D4：语言状态与主题同构

- 存储键 `strata.lang`，值 `"en"` / `"zh"`；用现有 `store.get/set`（JSON 编码，与 `strata.sampling` 一致，而不是 `strata.theme` 的裸字符串——少一种特例）。
- `index.html` 的 `<head>` 内联脚本扩展现有的主题脚本：读 `strata.lang` → 若为 `zh` 则 `document.documentElement.lang = "zh"`。这样首屏 `<html lang>` 就是对的。
- `<script src="web/i18n.js">` 放在 `<script src="web/app.js">` **之前**（`</body>` 前）。此时骨架已全部解析，`i18n.js` 顶层立即 `applyLang()`，所以中文用户在 `app.js` 执行前就已看到中文——不需要 `DOMContentLoaded`，也不会闪英文。
- 默认判定：无 `strata.lang` 时 `(navigator.language || "en").toLowerCase().startsWith("zh") ? "zh" : "en"`。
- 一次性覆盖：`?lang=zh` / `?lang=en` 优先于存储，但**不写入存储**（用于快捷方式与手工验证；`app.js:975` 已有 `?q=` 的先例）。

### D5：切换后走"重绘"而不是"重建"

`setLang(lang)`：存 `strata.lang` → 设 `<html lang>` → 更新 `LOCALE` → `applyLang()` 处理骨架 → 调用**已有**的重绘函数处理动态区：`renderChat()`、`render(lastMetrics)`、`renderMcp()`、`renderAttachments()`、`setBusy(!!busy)`、`loadDrawer()`、`outputs()`、`loadHealth()` 的文案部分。

- 为什么复用而不是新写：这些函数已经是幂等的、从数据渲染 DOM 的（`renderMonitor` / `renderAbout` / `updateAssistant`），换语言只是让它们读新的 `t()`。
- 例外：**正在流式的回答不能重建消息 DOM**（会丢滚动位置与光标）。`renderChat()` 在非流式时安全；`busy` 存在时只对该消息调 `updateAssistant(el, m, true)`。

### D6：`m.meta` 从"已渲染字符串"改为结构化数据

`app.js:749` 把 `123 tokens · 45.6 tok/s · stopped · projection on` 作为字符串存进聊天历史。若不改，历史消息在切换语言后仍是英文，违反 spec 的"切换即时生效"。

改法：`m.meta` 拆成 `m.metaData = {tokens, tokS, stopped, projection, toolCalls, limit}`，由 `updateAssistant()` 在渲染时用 `t()` 生成文本。

- 兼容：读到旧的 `strata.chat` 记录里没有 `metaData` 而有 `meta` 字符串时，原样显示那个英文字符串（不猜测、不重算）。不做一次性迁移脚本——数据量小、只影响旧记录的一行灰色小字。
- `m.error` 保持原文：它是服务端文本，spec 归入"不翻译的数据"。

### D7：格式化 locale 跟随界面语言

新增 `let LOCALE = "en"`（或 `"zh-CN"`），`fmt` / `kfmt` / `ctxfmt` / `gb` / `renderTotals` 的 `toLocaleString` / `toLocaleTimeString` 全部传 `LOCALE` 而不是 `undefined` / `[]`。

- 为什么：现在传 `undefined` 意味着数字分组跟浏览器系统语言，会出现"中文界面 + 英文日期"的混搭（spec 的"数字与日期时间按语言格式化"）。

### D8：字体只加回退，不打包 CJK 字体

`--st-font` 追加 `"Microsoft YaHei UI","PingFang SC","Noto Sans CJK SC","Source Han Sans SC"`。Outfit 的 `unicode-range` 已限定拉丁，汉字自动走回退，拉丁与数字仍用 Outfit。

- 为什么不打包 CJK 字体：一个可用的中文字体子集也是几 MB，且界面文案会变，子集化需要构建步骤——与"无构建步骤"冲突。系统字体在 Windows / macOS / Linux 桌面都可用。
- 需要一处新 CSS：`.seg` 是 `grid-template-columns: repeat(4, 1fr)`（`app.css:144`），About 里的两选项语言分段控件加 `.seg--2 { grid-template-columns: repeat(2, 1fr) }`。

### D9：词典完整性由 pytest 检查

`serve/test_server.py` 新增一个测试：读 `serve/web/i18n.js`，按 `// --- en ---` / `// --- zh ---` 两个标记块，用 `^\s*"([a-z0-9_.]+)":` 收集键，断言 zh 的键集合是 en 的子集且 en 无重复键。

- 为什么不用 Node 跑 JS 断言：仓库没有 JS 测试框架，也不该为一条断言引入 node 依赖（CI 上未必有 node）。
- 代价：依赖"一条词条一行、键用双引号"的书写约定。缓解：`i18n.js` 顶部注释写明该约定，测试同时断言 en 键数 ≥ 一个下限，防止标记块写错导致空集合通过。
- 运行时兜底：`t()` 找不到词条时回退 `en`，再找不到就返回键名并在控制台 `console.warn`——开发时能立刻看见漏词条，用户看到的是英文而不是空白。

## Risks / Trade-offs

- **[漏改字符串]** 约 200 处散在两文件，中文下残留英文很容易发生 → 收尾用 grep 审计：`grep -nE '"[A-Z][a-z]+[ a-z]' serve/web/app.js` 与 `grep -nE '>[A-Za-z][A-Za-z ]+<' serve/web/index.html`，逐条确认剩下的都是数据（型号名、URL、单位）；再人工走一遍三个标签页 + 抽屉 + 每种 toast。
- **[病句]** 中文按英文语序拼装 → D3 的整句模板；审计时专门检查含 `{` 占位符的词条在两种语言下都读得通。
- **[流式中重绘丢状态]** 语言切换发生在 SSE 输出中途，重建消息 DOM 会丢滚动与光标 → D5 的"重绘不重建"；测试场景里显式覆盖。
- **[旧聊天记录的 meta 是英文]** 不做迁移，旧记录那一行小字保持英文 → D6 明确记录为已知限制，不假装修好。
- **[词典体积]** 两种语言全量内嵌，英文用户也下载中文词典（约 15–20 KB）→ 同源本地服务，无网络成本；不做按需加载以换取无竞态与无闪英文。
- **[`i18n.js` 书写约定被破坏]** 有人把多条词条写在一行，完整性测试的键集合就漏 → 测试断言 en 键数下限；约定写在文件顶部。
- **[CJK 字体度量差异]** 汉字比拉丁宽，按钮/表头可能溢出 → 中文实际比英文紧凑（`Generating` → `生成中`），风险偏低；仍需在中英文两种语言下各看一次窄屏（`app.css` 已有响应式断点）。
- **[双语维护漂移]** 将来加文案只写英文 → D9 的测试是主要防线；`t()` 的回退保证界面不坏。

## Migration Plan

无数据迁移、无服务端部署步骤。

1. 前端四个文件（`i18n.js` 新增，`index.html` / `app.js` / `tokens.css` 修改）随仓库更新，服务器重启后生效（`/web/` 已经是 `Cache-Control: no-cache`，浏览器不会用旧脚本）。
2. 老浏览器里已存的 `strata.chat` / `strata.sampling` 不动；没有 `strata.lang` 时按浏览器语言判定，中文浏览器用户会立刻看到中文，无需任何操作。
3. 回滚：还原这四个文件即可，没有需要清理的存储键（多余的 `strata.lang` 会被忽略）。

## Open Questions

- 是否需要繁体中文（`zh-Hant`）与地区差异用词（"计算机/电脑"、"内存/显存"）——留待真实用户反馈，词典结构已支持再加一个块。
- `?lang=` 是否应该写入存储成为持久选择——当前决定是不写（一次性预览），若用户反馈希望分享链接即固定语言，再改一行。
