# Proposal

## Why

Strata 的 Web 界面（Chat / Monitor / About 三个标签页）目前只有英文，而本仓库是面向中文用户的中文分支。本地部署的用户第一次打开 `http://127.0.0.1:8080` 看到的就是全英文界面：采样抽屉里的 `Thinking / Top-p / Seed / Use for other apps too`、Monitor 里的 `Experts in VRAM / Hit rate / Reading prompt`、以及所有 toast 提示，都需要中文用户去猜含义。这直接挡住了"在自己 PC 上跑一个模型"这个核心体验的第一步。

需要的是一个**中英文可切换**的界面：默认跟随浏览器语言，用户可以随时切换并记住选择，切换即时生效、不需要刷新页面。

## What Changes

- 新增 `serve/web/i18n.js`：中英词典、`t(key, params)` 查找、DOM 本地化（`data-i18n` / `data-i18n-title` / `data-i18n-aria` / `data-i18n-placeholder`）、语言选择的读取与持久化（`localStorage` 的 `strata.lang` 键，与现有 `strata.theme` 同一套 `store` 约定）。
- `serve/web/index.html`：静态文案改为带 `data-i18n` 标记；`<html lang>` 随语言变化；顶栏在主题按钮旁新增一个语言切换按钮（`EN / 中`）；About > Settings 增加一行语言选择；`<head>` 内联脚本在首屏绘制前决定语言，避免先闪英文。
- `serve/web/app.js`：约 120 处动态字符串改为 `t()` 调用——状态胶囊（`Reading prompt · 42%`、`3 queued`）、Monitor 八张指标卡的标签与副标题、模型状态卡、上下文仪表、请求表的状态徽章与表头、About 的事实表键名、MCP 卡片与工具调用块、所有 toast 文案、导出 Markdown 的 `## You` / `Thinking` 标题、输入框提示 `Shift+Enter: new line`。
- 数字、日期、时间按当前语言格式化：现有的 `toLocaleString(undefined, …)` / `toLocaleTimeString([], …)` 改为传入当前 locale。
- 语言切换即时重绘：已打开的界面（含正在流式输出的回答的界面外壳）立刻换语言，历史聊天记录本身不动。
- **不做**：不改 `serve/server.py` 的接口和错误文案（已与用户确认）。服务端返回的英文错误（如 `missing or wrong API key`）原样显示在前端。
- **不做**：不翻译模型回答、MCP 服务器名 / 工具名 / 工具描述、GPU 与 CPU 型号名——这些是数据，不是界面文案。
- 无 **BREAKING** 变更：浏览器语言不是中文的用户，看到的界面与现在完全一致。

## Capabilities

### New Capabilities

- `web-ui-i18n`: Strata Web 界面的语言本地化——支持的语言、默认语言的判定、用户切换与记住选择、切换的即时生效、文案与数字/日期格式随语言变化、以及哪些内容属于不翻译的数据。

### Modified Capabilities

（无。`openspec/specs/` 目前为空，本项目还没有任何已归档的能力规格。）

## Impact

**改动的文件**

- `serve/web/i18n.js`（新增）——词典与本地化运行时
- `serve/web/index.html`——`data-i18n` 标记、语言按钮、`<html lang>`、`<head>` 内联脚本、`<script src="web/i18n.js">`
- `serve/web/app.js`——动态文案改 `t()`，格式化改带 locale
- `serve/web/tokens.css`——`--st-font` 增加 CJK 回退字体（Outfit 只覆盖拉丁字符）
- `serve/test_server.py`——`test_page_and_files` 增加 `/web/i18n.js` 的 200 断言

**服务端**：无接口变更。`serve/server.py:1526` 的 `/web/<name>` 白名单已经允许扁平的 `.js` 文件名，`i18n.js` 直接可取，不需要改服务器。

**依赖与构建**：无新增第三方依赖，无构建步骤，无网络请求（词典随 `i18n.js` 一起加载，与现有"除本服务器外无网络"的约定一致）。

**文档**：README 的 Web 界面段落（约 168 行）与 `docs/DETAILS.md` 需要说明语言切换与默认行为。

**不受影响**：聊天历史与采样设置的存储格式（`strata.chat` / `strata.sampling`）不变；API 客户端（omp 等）看到的响应不变。
