# Tasks

## 1. 本地化运行时 `serve/web/i18n.js`

- [x] 1.1 新建 `serve/web/i18n.js`，内含 `const STRATA_I18N = {en: {…}, zh: {…}}`，两个词典分别用 `// --- en ---` 与 `// --- zh ---` 标记；文件顶部注释写明书写约定（一条词条一行、键用双引号、点号分组名），因为 8.2 的测试按这个约定解析。验证：`python serve/test_server.py` 全部通过（此时只是新增一个未被引用的文件）。
- [x] 1.2 实现语言判定：`?lang=` 查询参数 → 已存的 `strata.lang` → `navigator.language` 以 `zh` 开头判中文；只接受 `en` / `zh`，其他值回退到上一级判定（spec「支持的界面语言」「默认语言的判定」）。验证：`?lang=fr` 打开时界面为英文；存了 `strata.lang="zh"` 后 `?lang=fr` 仍是中文。
- [x] 1.3 实现 `t(key, params)`：`{name}` 占位符替换、缺词条回退 `en`、再缺则返回键名本身并 `console.warn`；复数用 `_one` / `_other` 两键。验证：控制台里 `t("pill.reading", {pct: 42})` 返回整句，`t("nope.missing")` 返回 `"nope.missing"` 且有 warn。
- [x] 1.4 实现 `applyLang()`：遍历 `[data-i18n]` / `[data-i18n-title]` / `[data-i18n-aria]` / `[data-i18n-placeholder]`，分别用 `textContent` 与对应属性赋值，并同步 `document.documentElement.lang`。验证：控制台改某元素的 `data-i18n` 值后调 `applyLang()`，该元素文案随之变化。
- [x] 1.5 实现 `setLang(lang)`：`store.set("lang", …)`（沿用 `store` 的 try/catch，隐私模式不抛错）→ 更新 `LANG` 与 `LOCALE` → `applyLang()` → 调用 `window.strataRedraw`（由 app.js 后注册，可能为空）。验证：隐私窗口里 `setLang("zh")` 不抛错且界面换语言，刷新后回到默认判定。

## 2. 骨架本地化 `serve/web/index.html`

- [x] 2.1 扩展 `<head>` 里已有的首屏脚本（`index.html:10-17`）：读 `strata.lang`，为 `zh` 时把 `<html lang>` 设为 `zh`。验证：存 `strata.lang="zh"` 后刷新，`document.documentElement.lang === "zh"`，且肉眼确认首屏直接是中文、没有先闪英文。
- [x] 2.2 给三个标签页按钮、状态胶囊文字、抽屉标题、卡片标题加 `data-i18n`，**保留英文原文作为元素的默认内容**（缺词条时结构性回退）。验证：中文下顶栏与所有卡片标题为中文；临时删掉词典里 `tab.chat` 一条，该处仍显示 "Chat" 而不是空白或键名。
- [x] 2.3 给全部 `title` / `aria-label` / `placeholder` 加 `data-i18n-title` / `data-i18n-aria` / `data-i18n-placeholder`（附件、新建对话、导出、采样、停止、发送、主题、关闭、复制、移除附件、输入框、以及 `Hit rate` 表头那段长 `title`）。验证：`grep -c 'aria-label' serve/web/index.html` 的每个命中都能找到配对的 `data-i18n-aria`；中文下逐个悬停图标按钮看到中文提示。
- [x] 2.4 顶栏主题按钮旁新增语言按钮 `#lang-btn`（显示 `中` / `EN`），About > Settings 新增一行语言分段控件（复用 `app.css:144` 的 `.seg`）。验证：两个入口都能切换语言，且入口本身在两种语言下都能被认出（spec「语言入口」）。
- [x] 2.5 在 `<script src="web/app.js">` 之前插入 `<script src="web/i18n.js">`。验证：`grep -n 'web/i18n.js' serve/web/index.html` 显示它在 `web/app.js` 之前；浏览器 Network 里两个脚本同源、无外部请求。

## 3. Chat 区动态文案（`serve/web/app.js`）

- [x] 3.1 改 `setBusy` 的 `Shift+Enter: new line`（`app.js:665`）、空状态 `Ask anything` 与 `loadHealth` 里的 `${health.model} runs on this PC. Nothing leaves it.`（`app.js:104`）、`attach-btn` 的两个动态 `title`（`app.js:102-103`）。验证：中文下空状态、输入框提示、附件按钮提示均为中文，模型名保持原样。
- [x] 3.2 改消息外壳：`You · 时间`（`app.js:508`）、`Thinking…` / `Thought for N s` / `Thoughts`（`app.js:572-573`）、`Writing` 占位（`app.js:586`）、`Stopped`、复制按钮的 `aria-label` 与 `title`（`app.js:514`）。验证：中文下发送一条消息，气泡下方小字与思考标题为中文，回答正文不变。
- [x] 3.3 改工具调用块：`TOOL_STATE` 五个状态、`Arguments` / `Result` / `Error` / `(being written)` / `N characters` / `cut for the model`（`toolHtml`，`app.js:529-545`）。验证：中文下展开一个工具块，标签为中文，工具名与参数、结果内容原样。
- [x] 3.4 改附件路径：`renderAttachments` 的 `Remove`、`addFiles` 的四个 toast（`Pictures are off` / `Picture too large` / `Not a text file` / `File too large`，`app.js:809-821`）。验证：中文下拖一个 600 KB 的 .txt，toast 标题与正文为中文、文件名原样。
- [x] 3.5 改导出 Markdown：`## You` 与 `Thinking` 标题走 `t()`（`app.js:790-792`），文件名前缀 `strata-chat-` 不变。验证：中文下导出，打开文件确认段落标题为中文、回答正文与工具内容不变。
- [x] 3.6 改聊天相关 toast：`New chat` + `Undo` 按钮、`Nothing to save yet`、`Still writing`、`The request failed`、`Copied to clipboard`、`API key saved`、`API key needed`（`app.js:53,96,159,745,777,783,786,959-966`）。验证：中文下逐条触发（新建对话、空对话导出、忙时新建、断网发送、复制、保存密钥、无密钥访问）。

## 4. Monitor 区动态文案

- [x] 4.1 `METRICS` 八张卡的 `label`（`app.js:111-119`）改为词典键；`setMetric` 的副标题全部走 `t()`：`Decode now` / `Prefill last request` / `of N W limit` / `to GPU N MB/s` / `idle Gen4` / `N cores · M threads` / `needs psutil (setup installs it)` / `write N MB/s` / `N experts cached` / 多卡时的 `GPU 0 …` 串。验证：中文下八张卡标签与副标题全中文，单位 `t/s`、`GB`、`°C`、`W`、`Gen4`、`x16` 不变。
- [x] 4.2 状态胶囊：`Connecting…`（HTML）、`Reading prompt · N%` / `Generating · N tok/s` / `Idle` / `N queued`（`render`，`app.js:183-191`）、`API key needed` / `Server not reachable`（`poll`，`app.js:158,168`）。验证：中文下发送请求观察胶囊；停掉服务器约 3 秒后胶囊显示中文"服务器不可达"。
- [x] 4.3 模型状态卡：`Waiting for a request` / `Reading prompt` / `Generating`（`app.js:210-224`）、`live.phase` 的首字母大写那处改成词典映射（`app.js:221`，phase 来自服务端：已知值翻译，未知值原样显示）、`N / M tokens · P%` 与 `last: N tokens at X tok/s` 两个 detail 串。验证：中文下状态标签与进度条右侧明细为中文；伪造一个未知 phase 值时该处仍是原文。
- [x] 4.4 上下文卡：`Context fill` / `Experts in VRAM` / `System RAM` / `GPU temperature`（HTML）与 `ctx-sub`、`slots-text`、`ram-text`、`temp-text` 的数值串（`app.js:265-280`）。验证：中文下四行标签为中文，数值与 `–` 占位不变。
- [x] 4.5 请求表：表头 8 列与 `Hit rate` 的长 `title`（HTML）、`No requests yet`、`badge` 五种状态（`Done` / `Max tokens` / `Stopped` / `Closed` / `Error`）、`ESP` / `stock` 徽章及其 `title`、`Show all (N)` / `Show fewer`（`app.js:285-305`）、`renderTotals` 的整句（`app.js:196-203`，按 D3 改成一条带占位符的整句模板而不是片段拼接）。验证：中文下跑两次请求，表头、徽章、累计行全中文，"自 X 起"那句中文读得通。
- [x] 4.6 MCP 卡：`MCP_STATE` 五种状态、`mcp-sum`、`mcp-row-sub`、每行的 `N tools`（`app.js:366-385`）与卡片说明段（HTML）。验证：配一个 MCP 服务器后中文下卡片文案为中文，服务器名、工具名、工具描述原样。

## 5. About 与采样抽屉

- [x] 5.1 `renderAbout` 的三组事实表：键名（`Model` / `Engine` / `Context` / `KV cache` / `Experts in VRAM` / `Speculation` / `Images` / `Experimental speed projection` / `GPU` / `CPU` / `RAM`）与值文案（`built from source`、`streamed: … in VRAM, the rest in RAM`、`not readable (NVML)`、`on` / `off`、`MTP drafts up to N tokens`、`projectionText` 的整段说明，`app.js:333-356`）。验证：中文下 About 两张卡全中文，GPU / CPU 型号名与版本号原样。
- [x] 5.2 About 的 API 卡与设置卡：`Connect your tools`、`Any OpenAI- or Anthropic-compatible client works with these addresses.`、`API key` 及其 `<output>` 说明、`Not needed` 占位符、`Dark theme`、`Chats, settings and the key are kept in this browser only.`、GitHub 链接文字。验证：中文下全卡片文案为中文，base URL 与模型名不变。
- [x] 5.3 采样抽屉：`Sampling` 标题、六个字段标签与其 `<output>` 说明、`Off/Low/Medium/High` 四段、`outputs()` 里的 `0 · greedy` 与 `{none,low,medium,high}` 四个描述（`app.js:946-950`）、五个 toggle 行的标题与副标题、`Reset` / `Apply`、`Until done` / `Random` 占位符。验证：中文下抽屉无一处英文残留，滑杆数值按 locale 格式化。
- [x] 5.4 抽屉保存的 toast：`Sampling saved` 的三种正文与 `Saved here, but not for other apps`（`app.js:955-966`）。验证：中文下开、关"给其他应用也用"各保存一次，两条正文都是读得通的中文整句。

## 6. 消息元数据结构化（design D6）

- [x] 6.1 `send()` 末尾把 `m.meta` 的字符串拼装（`app.js:751-759`）改成结构化 `m.metaData = {tokens, tokS, stopped, projection, toolCalls, limit}`。验证：`JSON.parse(localStorage.getItem("strata.chat"))` 里新记录有 `metaData` 字段。
- [x] 6.2 `updateAssistant` 用 `t()` 从 `metaData` 渲染文本；没有 `metaData` 的旧记录原样显示已存的 `meta` 字符串，不重算、不猜测。验证：手工往 `strata.chat` 塞一条只有 `meta: "12 tokens · 40 tok/s"` 的记录，刷新后该行仍是英文且不报错；新记录在中文下是中文。
- [x] 6.3 `m.error` 保持服务端原文，外层 toast 标题走 `t()`。验证：中文下触发一次 401，错误文本仍是 `missing or wrong API key`，提示语是中文。

## 7. 格式化与字体

- [x] 7.1 `fmt` / `kfmt` / `ctxfmt` / `gb`（`app.js:9-14`）与 `toLocaleString` / `toLocaleTimeString`（`app.js:198,303,474`）全部传入 `LOCALE` 而不是 `undefined` / `[]`。验证：中文下请求表时间列与"自 X 起"的星期名为中文；切回英文后恢复英文。
- [x] 7.2 `tokens.css:11` 的 `--st-font` 追加 CJK 回退（`"Microsoft YaHei UI","PingFang SC","Noto Sans CJK SC"`）；`app.css` 加 `.seg--2 { grid-template-columns: repeat(2, 1fr) }`。验证：中文界面无缺字方块；About 的语言分段控件两列等宽；375px 宽下无横向滚动，中英文各看一次。

## 8. 测试

- [x] 8.1 `serve/test_server.py::WebApp.test_page_and_files`（`test_server.py:810`）增加 `/web/i18n.js` → 200 + `javascript` 的断言，并断言 `/` 的 body 含 `"web/i18n.js"`。验证：`python serve/test_server.py WebApp -v` 通过。
- [x] 8.2 新增词典完整性测试（design D9）：读 `serve/web/i18n.js`，按 `// --- en ---` / `// --- zh ---` 标记块用 `^\s*"([a-z0-9_.]+)":` 收集键，断言两个键集合**相等**、en 侧无重复键、en 键数 ≥ 180。验证：`python serve/test_server.py -v` 通过；临时删掉一条 zh 词条后该测试失败，补回后恢复通过。
- [x] 8.3 端到端手工验收（跨任务）：中英文各走一遍三个标签页 + 采样抽屉 + 一次带 MCP 工具调用的回答 + 导出 Markdown + 375px 窄屏；并在流式输出中途切换语言一次。验证：用 Playwright 在 Chrome 中逐项执行（check2.mjs / check3.mjs / check4.mjs），结果全部通过。对照 `specs/web-ui-i18n/spec.md` 的每个 Scenario 逐条打勾，无失败项。

## 9. 文档

- [x] 9.1 README 的 Web 界面段落（`README.md:168` 附近）补一句语言切换与默认判定；"Your chats: only in your browser"（`README.md:186-188`）处补上 `strata.lang` 同样只存浏览器本地。验证：在 README 里能查到"默认语言怎么定"和"在哪里切换"。
- [x] 9.2 `docs/DETAILS.md` 新增一小节：`i18n.js` 的词典书写约定（一条一行、双引号键、两个标记块被 8.2 的测试依赖）、`t()` 的回退规则、以及"不翻译的数据"清单（模型输出、MCP 工具名与描述、硬件型号名、服务端错误文本）。验证：照文档的约定新增一条词条到两个词典，8.2 的测试仍通过。
