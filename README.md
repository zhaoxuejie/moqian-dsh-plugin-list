# moqian-dsh-plugin-list

> 我的 DeepSeek Harness 插件（`dsh-plugin`）清单 —— 收录本人开源的 DSH 插件，按用途分类，附仓库地址、npm 包名与安装命令，方便发现、选型与安装。

[![GitHub](https://img.shields.io/badge/GitHub-zhaoxuejie-181717?logo=github)](https://github.com/zhaoxuejie?tab=repositories&q=dsh-plugin&type=public)
[![DeepSeek Harness](https://img.shields.io/badge/DeepSeek%20Harness-Plugin-blue)](https://github.com/deepseek-ai/deepseek-harness)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

DeepSeek Harness（简称 DSH）以 **「一切皆插件」** 为设计理念。本仓库把本人开发的插件整理成一份可检索的清单，共 **9 个功能插件 + 1 套开发教程**。

---

## 目录

- [快速开始](#快速开始)
- [插件总览](#插件总览)
- [插件详解](#插件详解)
  - [🛡️ 安全与治理](#安全与治理)
  - [🔍 可观测与复盘](#可观测与复盘)
  - [🧠 知识与研究](#知识与研究)
  - [⚙️ 开发提效](#开发提效)
  - [🔔 交互与提醒](#交互与提醒)
  - [🎮 趣味与氛围](#趣味与氛围)
- [相关项目](#相关项目)
- [环境要求](#环境要求)
- [贡献](#贡献)
- [License](#license)

---

## 快速开始

所有插件都是 **DSH profile bundle**（不是独立应用），通过 `dsh plugin` 命令装进某个 profile 后由 loader 自动组合生效。GUI 通常使用 `web` profile：

```bash
# 从 npm 安装（推荐，本清单所有插件均已发布 npm）
dsh plugin --profile web add <包名>

# 从 GitHub 源码安装（跟随上游最新提交）
dsh plugin --profile web add github:zhaoxuejie/<仓库名>

# 从本地目录安装（源码调试；link 保持与原目录联动）
dsh plugin --profile web add link:./<插件目录>
```

> **提示**
> - 把 `web` 换成你自己的 profile 名（如 `desktop`）。
> - 插件自带 `dsh.bundle.patch`（`cordis.patch.yml`），loader 会据此自动应用配置，无需手改宿主 `cordis.yml`。
> - 多数插件的配置项可在 DSH 设置界面的对应分区里直接修改，且支持热更新。

---

## 插件总览

| 插件 | 用途 | npm | 语言 | 仓库 |
|---|---|---|---|---|
| [`dsh-plugin-tool-guard`](#dsh-plugin-tool-guard) | 工具调用安全守卫（危险命令拦截、路径白名单、人工审批、审计） | `1.0.0` | TypeScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-tool-guard) |
| [`dsh-plugin-log-forwarder`](#dsh-plugin-log-forwarder) | 实时日志转发到 WebSocket / Loki / 本地文件 | `1.0.4` | TypeScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-log-forwarder) |
| [`dsh-plugin-session-export`](#dsh-plugin-session-export) | 会话黑匣子，导出 Markdown / HTML 复盘报告 | `1.0.1` | JavaScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-session-export) |
| [`dsh-plugin-vault-memory`](#dsh-plugin-vault-memory) | 把 Obsidian 知识库变成 Agent 长期记忆与工作台 | `0.3.1` | JavaScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-vault-memory) |
| [`dsh-plugin-academic-paper`](#dsh-plugin-academic-paper) | 学术文献真实检索与引用格式生成，杜绝编造 | `0.1.2` | TypeScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-academic-paper) |
| [`dsh-plugin-todo-scanner`](#dsh-plugin-todo-scanner) | 扫描代码 TODO/FIXME，生成结构化清单与「TODO 雷达」面板 | `1.0.0` | TypeScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-todo-scanner) |
| [`dsh-plugin-desktop-notice`](#dsh-plugin-desktop-notice) | 任务完成 / 等待输入 / 失败时桌面弹窗提醒 | `0.2.1` | JavaScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-desktop-notice) |
| [`dsh-plugin-feihualing`](#dsh-plugin-feihualing) | 飞花令对诗游戏（浏览器即时对战 / 对话模式） | `1.1.3` | TypeScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-feihualing) |
| [`dsh-plugin-internet-meme`](#dsh-plugin-internet-meme) | Web 端热梗弹幕字幕，零侵入的氛围层 | `0.4.4` | JavaScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-internet-meme) |
| [`dsh-plugin-learning-path`](#dsh-plugin-learning-path) | DSH 插件开发教程：15 节课 + 4 个实战项目 | — | JavaScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-learning-path) |

---

## 插件详解

### 安全与治理

#### dsh-plugin-tool-guard

**给 AI 的每一次工具调用装上防火墙** —— 危险命令拦截 · 禁止读取敏感文件 · 限定可操作目录 · 高危操作人工审批 · 全量审计留痕。

挂载即生效、卸载即失效，**零侵入** Agent 业务代码：不改一行业务逻辑，规则引擎在每次工具执行前独立决策。

```bash
dsh plugin --profile web add dsh-plugin-tool-guard
```

**亮点**

- 规则引擎拦截危险命令，路径白名单 + 敏感文件黑名单双重防线
- 高危操作触发人工审批，全量审计日志可追溯
- 附带侧边面板，规则与审计记录可视化

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-tool-guard) · [npm](https://www.npmjs.com/package/dsh-plugin-tool-guard)

---

### 可观测与复盘

#### dsh-plugin-log-forwarder

把 Agent 运行时的全部事件**实时**转发到外部日志系统（**WebSocket / Loki / 本地文件**），可用于外部页面实时监控 Agent 的每一步思考与工具执行，适合调试、监控、演示场景。

```bash
dsh plugin --profile web add dsh-plugin-log-forwarder
```

**亮点**

- 全量事件采集：订阅 `session/created`、`session/event`、`session/disposed`、`agent/error`，标准化为统一 JSON
- 三种输出通道：WebSocket（实时推送）、Loki（日志聚合）、本地 JSONL 文件
- 通道级状态查询与暂停/恢复控制，内置缓冲与失败重试

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-log-forwarder) · [npm](https://www.npmjs.com/package/dsh-plugin-log-forwarder)

---

#### dsh-plugin-session-export

**会话黑匣子** —— 以旁路监听方式全量采集每一轮对话的用户输入、模型回复、工具调用与报错，不干预 Agent 正常运行；一键导出美化 Markdown / HTML 复盘报告。

```bash
dsh plugin --profile web add dsh-plugin-session-export
```

> 也可从源码安装：`git clone https://github.com/zhaoxuejie/dsh-plugin-session-export.git` 后 `dsh plugin --profile web add link:./dsh-plugin-session-export`。

**亮点**

- 双格式导出：Markdown（结构化文本）+ HTML（自包含单文件、深浅主题、轮次时间轴、工具调用可折叠）
- 实时统计：轮次数、工具调用分布、耗时、最慢工具 Top3、报错数、Token 用量
- 快照补录不漏历史对话；`api_key / token / password` 等敏感字段自动掩码
- 内置 Web 统计面板，右上角一键导出 / 清空

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-session-export) · [npm](https://www.npmjs.com/package/dsh-plugin-session-export)

---

### 知识与研究

#### dsh-plugin-vault-memory

> 让 DeepSeek Harness agent 学会翻你的 Obsidian 库、用你的笔记、把成果存回你的库 —— **你的笔记从此是活的资产**。

全文 / 语义检索 · 会话记忆注入 · 一键捕获 · 巡检管家（只建议不擅改）。

```bash
dsh plugin --profile web add dsh-plugin-vault-memory
```

**亮点**

- **可检索**：问「我之前研究过 X 的什么结论」，Agent 翻遍全库作答并**标注出处文件，不编造**
- **有记忆**：新会话开头自动注入「记忆快照」（库状态、活跃项目、近期笔记），无需反复自我介绍
- **能沉淀**：一键把会话产出存成笔记，自动 frontmatter、归类打标签、推荐关联笔记
- **会巡检**：孤儿笔记、断链、缺总览逐条列出并给理由，**你批准它才动手**，写前自动备份可回滚

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-vault-memory) · [npm](https://www.npmjs.com/package/dsh-plugin-vault-memory)

---

#### dsh-plugin-academic-paper

**插件负责真实数据源检索与引用格式生成，大模型负责解读与综述** —— 从机制上杜绝模型编造论文、作者、DOI 等文献信息。

```bash
dsh plugin --profile web add dsh-plugin-academic-paper
```

**亮点**

- **检索**：arXiv（公开 API）＋ Semantic Scholar（公开 API），支持关键词、作者、年份范围过滤与相关性 / 日期 / 引用数排序
- **引用**：GB/T 7714 / APA / BibTeX 三种格式，本地确定性生成，**不依赖大模型**
- **文献库**：按会话隔离，DOI / arXiv ID / 标题自动去重，支持批量导出
- **侧边面板**：文献库列表 + 检索历史 + 一键导出（复制或下载 `.bib` / `.md`）
- **防幻觉**：system prompt 强制要求学术文献必须经工具检索

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-academic-paper) · [npm](https://www.npmjs.com/package/dsh-plugin-academic-paper)

---

### 开发提效

#### dsh-plugin-todo-scanner

递归扫描本地项目目录中的 `TODO / FIXME / HACK / NOTE / XXX / BUG / OPTIMIZE / REVIEW` 标记，生成结构化清单与统计，支持状态管理、Markdown 导出与侧边「**TODO 雷达**」面板。

```bash
dsh plugin --profile web add dsh-plugin-todo-scanner
```

**亮点**

- **Host 侧**注册 8 个工具（`todo_scan` / `todo_list` / `todo_get` / `todo_update_status` / `todo_batch_update` / `todo_export` / `todo_stats` / `todo_clear`），供模型读取与管理扫描结果
- **Client 侧**「TODO 雷达」面板：统计卡片 + 类型分布 + 可筛选 / 排序清单 + 状态操作
- 只读扫描，白名单 + 系统目录黑名单双重防护，文件数 / 大小上限保护

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-todo-scanner) · [npm](https://www.npmjs.com/package/dsh-plugin-todo-scanner)

---

### 交互与提醒

#### dsh-plugin-desktop-notice

> 让 DSH 的每一次「需要你」都及时抵达桌面。

任务完成、卡住等确认、出错失败 —— 右下角弹窗 + 音效一秒知晓，**你是被提醒的人，不是盯屏幕的人**。

```bash
dsh plugin --profile web add dsh-plugin-desktop-notice
```

**亮点**

- **三类事件**：任务完成 ✅ / 等待输入 ⏸ / 任务失败 ❌，各自独立开关
- **内容增强**：通知带项目名、git 分支、耗时、token、结果摘要，一眼判断要不要回来
- **防轰炸**：同类事件 10 秒窗口内自动合并；等待输入 5 分钟未处理自动重发（均可调）
- **前台检测**：DSH 窗口正在前台时自动不弹 —— 你在盯着看，弹窗是纯噪音
- 支持 Windows / macOS / Linux 原生通知，可配手机推送

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-desktop-notice) · [npm](https://www.npmjs.com/package/dsh-plugin-desktop-notice)

---

### 趣味与氛围

#### dsh-plugin-feihualing

DeepSeek Harness 飞花令对诗游戏，两种玩法：

- **🎴 浏览器即时对战（v1.1+）**：会话标题行点 🎴 打开浮层面板 —— 内置 AI 对手出题、限时对诗、连击计分、三档难度、局制结算与本地战绩。判定在本机完成，**不走模型回合** —— AI 在后台跑任务时，随手就能开一局休闲。
- **💬 对话模式**：简易 / 古法严格双模式，游戏状态（令字、得分、已用诗句、剩余提示次数）由插件**按会话独立维护**；插件自身不向主聊天输出流写入任何内容。

```bash
dsh plugin --profile web add dsh-plugin-feihualing
```

**亮点**

- **简易模式**：诗句（4–48 字）任意位置包含令字即可
- **古法严格模式**：令字必须位于本轮指定位置（第 1 字 → 第 7 字循环）
- 连对 7 题通关，超时 3 次惜败；检测到 shell / 文件写入类工具调用时自动暂停并保存现场

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-feihualing) · [npm](https://www.npmjs.com/package/dsh-plugin-feihualing)

---

#### dsh-plugin-internet-meme

给 DeepSeek Harness Web 页面添加一层轻量的「**热梗弹幕**」字幕：Agent 思考、调用工具、工具返回和本轮结束时，右侧显示向上漂移的短句提示。

```bash
dsh plugin --profile web add dsh-plugin-internet-meme
```

**亮点**

- **零侵入**：只增强浏览器界面，不改变模型对话、工具调用或会话记录
- 可自定义主题、颜色、密度与文案，支持可选提示音
- 提示音只在本地播放，不上传会话文本

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-internet-meme) · [npm](https://www.npmjs.com/package/dsh-plugin-internet-meme)

---

## 相关项目

### dsh-plugin-learning-path

**DeepSeek Harness 插件开发教程** —— 15 节课程 + 4 个实战项目 + 发布课，交互式单页应用，纯 HTML / CSS / JS 零构建。

想自己动手写插件，从这里开始：<https://zhaoxuejie.github.io/dsh-plugin-learning-path/>

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-learning-path) · [在线教程](https://zhaoxuejie.github.io/dsh-plugin-learning-path/)

### 其他

- [`dsh-daily-digest`](https://github.com/zhaoxuejie/dsh-daily-digest) —— DSH 每日工作摘要插件：自动记录任务 / 会话 / 错误，一键生成日报 / 周报 Markdown，附 Web 悬浮摘要卡。（命名不含 `dsh-plugin`，故未列入上方清单）

---

## 环境要求

- 已安装并可运行 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
- Node.js ≥ 20（`dsh-plugin-session-export` 要求 ≥ 18；`dsh-plugin-academic-paper` 要求 ≥ 22）
- 各插件的具体配置项与依赖，见对应仓库的 README

---

## 贡献

本仓库是个人插件清单，主要收录本人开发的插件。如果发现清单中的链接失效、版本过期或描述有误，欢迎提 [Issue](https://github.com/zhaoxuejie/moqian-dsh-plugin-list/issues) 指正。

如果你的插件希望被收录，也欢迎提 Issue 附上仓库地址和一句话简介。

---

## License

[MIT](LICENSE) © 2026 moqian-dsh-plugin-list contributors
