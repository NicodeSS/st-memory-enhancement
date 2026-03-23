# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

SillyTavern 记忆增强表格插件 (st-memory-enhancement)，为 SillyTavern 角色扮演提供结构化长期记忆管理。通过动态表格界面管理角色记忆、关键事件和重要物品，并将表格数据注入 AI 提示词上下文。

- **版本**: 2.2.8
- **运行环境**: SillyTavern >= 1.10.0 浏览器插件（纯 ES6 模块，无构建步骤，无 package.json）
- **入口**: `index.js` (由 `manifest.json` 声明)
- **语言**: JavaScript (ES6+)，使用 jQuery (来自 SillyTavern 宿主)

## 开发方式

无构建系统。代码以 ES6 模块形式被 SillyTavern 直接加载。开发时将此仓库克隆到 SillyTavern 的 `public/scripts/extensions/third-party/` 目录下，刷新浏览器即可生效。

调试工具: `scripts/settings/devConsole.js` 提供开发者控制台，`SYSTEM.codePathLog()` 用于代码路径追踪。

## 架构

### 五大全局管理器 (`core/manager.js`)

所有核心状态通过五个单例管理器访问，它们贯穿整个代码库：

| 管理器 | 职责 |
|--------|------|
| `APP` | SillyTavern 宿主 API 封装 (`services/appFuncManager.js`) |
| `USER` | 用户设置、聊天上下文、模板、隐私数据 |
| `BASE` | Sheet/Table 数据库操作 (CRUD、模板应用、hash sheet 转换) |
| `EDITOR` | UI 控制 (通知、弹窗、拖拽、调试日志) |
| `DERIVED` | 运行时中间缓存数据 |
| `SYSTEM` | 系统级工具 (模板渲染、代码路径日志、任务计时) |

### 数据模型 (`core/table/`)

- **SheetBase** (`base.js`): 核心属性定义 — uid, name, domain (`global`/`role`/`chat`), type (`dynamic`/`fixed`/`free`/`static`), cellHistory (只增不减的操作日志), hashSheet (2D 单元格 UID 数组)
- **Sheet** (`sheet.js`): 继承 SheetBase，添加渲染逻辑和持久化
- **Cell** (`cell.js`): 单元格管理，位置追踪，编辑/插入/删除操作
- **Template** (`template.js`): 预定义表格结构模板

### 核心数据流

```
模板 → buildSheetsByTemplates() → Sheet 实例化 → hashSheet 2D数组
    → cellHistory 记录操作 → 存储到 chatMetadata.sheets

AI 响应 → handleEditStrInMessage() → parseTableEditTag()
    → executeTableEditActions() → 更新 cells/rows
    → getTablePrompt() → 生成上下文注入下一条 AI 消息
```

### 模块分层

| 层 | 目录 | 说明 |
|----|------|------|
| 服务层 | `services/` | LLM API 集成 (`llmApi.js`)、i18n (`translate.js`)、宿主 API 封装 (`appFuncManager.js`) |
| 运行时 | `scripts/runtime/` | 表格刷新 (`absoluteRefresh.js`)、两步式表格更新 (`separateTableUpdate.js`) |
| 编辑器 | `scripts/editor/` | 单元格历史、模板编辑视图、样式编辑器、统计分析 |
| 渲染器 | `scripts/renderer/` | 表格推送到聊天界面、自定义样式渲染 |
| 设置 | `scripts/settings/` | 用户设置面板、外部 API (`standaloneAPI.js`)、开发控制台 |
| 组件 | `components/` | 弹窗菜单、拖拽管理器、表单管理器 |
| 工具 | `utils/` | 字符串处理、代码代理、路径处理 |
| 数据 | `data/` | 默认设置 (`pluginSetting.js`)、提示词模板 (`profile_prompts.js`) |
| 外部适配 | `external-data-adapter.js` | 为外部程序提供数据注入接口 |

### 关键设计模式

- **Proxy 模式**: `core/manager.js` 中通过 `createProxy` / `createProxyWithUserSetting` 拦截数据访问
- **事件驱动**: Cell 通过事件处理器响应 UI 交互
- **适配器模式**: `external-data-adapter.js` 包装核心逻辑供外部调用

## 数据存储

- **用户设置**: `APP.extension_settings` (SillyTavern 扩展设置)
- **表格数据**: `APP.getContext().chatMetadata.sheets` (聊天元数据)
- **全局/角色模板**: 存储在用户设置中
- **聊天表格**: 存储在消息元数据中 (`message.hash_sheets`)

## 国际化

三语言支持: `assets/locales/` 下 `zh-cn.json`, `zh-tw.json`, `en.json`。通过 `services/translate.js` 和 SillyTavern 的 `getCurrentLocale()` 切换。

## 注意事项

- 所有外部依赖来自 SillyTavern 宿主环境 (jQuery, Popup, 各种 API)，通过相对路径 import
- `cellHistory` 设计为只增不减的操作日志，不要清空或删除历史记录
- `hashSheet` 是 2D 数组，每个元素是 Cell UID 的引用
- 保存操作区分单人聊天 (`APP.saveChat()`) 和群组聊天 (`APP.saveGroupChat()`)
- `data/profile_prompts.js` 包含注入给 AI 的提示词模板，修改需谨慎
