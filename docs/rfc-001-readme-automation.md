# RFC-001：清单仓库 README 自动化方案

> 状态：**草案（未落地）**
> 目标：新增 `dsh-plugin-xxx` 时，**无需打开本清单仓库**即可让 README 自动保持最新。

---

## 1. 背景与问题

### 现状流程

```
开发 dsh-plugin-xxx  →  回到 moqian-dsh-plugin-list  →  手动改 README 的 4 处
```

需要手改的 4 处：

1. 总览表新增一行
2. 「插件详解」新增一个小节（分类 + 定位 + 安装命令 + 亮点 + 链接）
3. 目录（TOC）新增锚点
4. 「相关项目」/「环境要求」按需调整

### 已发生的维护事故

发布 `dsh-plugin-session-export@1.0.1` 时，因为 README 里硬编码了版本号，**同一轮对话内回改了两次 README**：

- `35bd265` 把 `未发布` 改为 `1.0.0`
- `a9e7c20` 把 `1.0.0` 改为 `1.0.1`

这类改动本该是零维护的。

### 问题本质

不是「Action 能不能写 README」（能），而是**清单要用的元数据存在哪里**：

- 现在存在**清单仓库的手写 Markdown 里** → 每次新增/变更都要人工同步
- 应存在**插件仓库的结构化字段里** → 开发插件时顺手写一次，清单自动聚合

---

## 2. 设计目标与非目标

### 目标

| # | 目标 | 验收标准 |
|---|---|---|
| G1 | 新增插件零改动清单仓库 | 只在插件仓库改 `package.json` |
| G2 | 版本号永不漂移 | README 中无硬编码版本，或由 Action 每周刷新 |
| G3 | 手写散文不被覆盖 | 标记区外的内容 Action 永不触碰 |
| G4 | 输出幂等 | 内容无变化时不产生 commit |
| G5 | 零第三方依赖 | 仅用 GitHub Actions + GitHub/npm 公开 API |

### 非目标

- ❌ 不追求自动生成「判断类」文字（分类归属、文风、排序策略）——这些由人在插件仓库一次性决定
- ❌ 不引入外部服务、数据库或构建平台
- ❌ 不自动发现并收录**第三方**插件（本清单是个人清单，收录范围需显式声明）

---

## 3. 核心设计

### 3.1 单一数据源上移

```
┌─────────────────────────┐
│ dsh-plugin-xxx          │
│   package.json          │
│     dsh.list: {         │  ← 唯一数据源（人写一次）
│       category,         │
│       tagline,          │
│       highlights        │
│     }                   │
└───────────┬─────────────┘
            │  GitHub Contents API
            ▼
┌─────────────────────────┐
│ moqian-dsh-plugin-list  │
│   scripts/sync-plugins  │  ← 只做聚合与渲染，不做判断
│   README.md（标记区）    │
└─────────────────────────┘
```

### 3.2 为什么放 `package.json` 而不是清单仓库的 `plugins.yml`

| 维度 | 放插件 `package.json` | 放清单仓库 `plugins.yml` |
|---|---|---|
| 新增插件时要改几个仓库 | **1 个**（你已经在改的那个） | 2 个 |
| 写文字的时机 | 开发时思路最清楚 | 事后凭记忆重述 |
| 是否需要跨仓库写权限 | 否（Action 只读插件仓库） | 否 |
| 对 npm 的影响 | 无（自定义字段被忽略） | — |
| 复用性 | 第三方插件也能用同一约定 | 仅本清单 |

**结论**：放插件 `package.json`。自定义顶层 `dsh` 字段本项目已在用（`dsh.bundle`），新增 `dsh.list` 是自然扩展。

### 3.3 读取源：GitHub 优先，npm 仅作校验

关键权衡——**从 npm 读会陷入「必须发版才能更新清单」的陷阱**：

| 读取源 | 时效性 | 是否需发新版 | 限流 | 结论 |
|---|---|---|---|---|
| npm `/<pkg>/latest` | 需发版后生效 | **是**（9 个存量插件要各发 1 个 patch） | 宽松 | ❌ 作为主源 |
| GitHub Contents API `package.json` | 提交即生效 | 否 | 认证后 5000 req/h | ✅ **主源** |
| `raw.githubusercontent.com` | 提交即生效 | 否 | 未认证 60 req/h | ⚠️ 备选 |

**采用**：`dsh.list` / `tagline` / `highlights` 从 **GitHub Contents API** 读；npm registry **只用于**判断「是否已发布」（决定安装命令写 `包名` 还是 `github:owner/repo`）。

这样 **9 个存量插件补字段后立刻生效，无需发新版本**。

---

## 4. 字段约定（Schema）

### 4.1 定义

```jsonc
// dsh-plugin-xxx/package.json
{
  "name": "dsh-plugin-xxx",
  "version": "1.0.0",
  "description": "……",           // 保留，Action 在缺 tagline 时回退用它
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "list": {
      "category": "可观测与复盘",   // 必填，枚举见 §5
      "tagline": "一句话定位",      // 必填，≤ 80 字
      "highlights": ["亮点一", "亮点二"],  // 可选，≤ 6 条，每条 ≤ 60 字
      "order": 10,                 // 可选，分类内升序；缺省按包名
      "exclude": false             // 可选，true = 不收录（内部试验插件）
    }
  }
}
```

### 4.2 校验规则

| 字段 | 必填 | 类型 | 约束 | 违规处理 |
|---|---|---|---|---|
| `category` | ✅ | string | 必须命中 §5 枚举 | 归入「未分类」+ 开 Issue |
| `tagline` | ✅ | string | 1–80 字，不含换行 | 回退 `description` + 开 Issue |
| `highlights` | ❌ | string[] | ≤ 6 条，每条 ≤ 60 字 | 超限截断 + 开 Issue |
| `order` | ❌ | number | 整数 | 忽略，按包名排序 |
| `exclude` | ❌ | boolean | — | `true` 则跳过 |

### 4.3 缺字段的行为

| 情况 | 行为 |
|---|---|
| 仓库无 `dsh.list` | 仍收录，`category` = 「未分类」，`tagline` 回退 `description`，**开 Issue 提醒登记** |
| 仓库已 `archive` | 从清单移除，Issue 提示（不静默删除） |
| npm 未发布 | 仍收录，安装命令用 `dsh plugin --profile web add github:zhaoxuejie/<repo>` |
| `package.json` 拉取失败 | 保留上次生成内容，记录 warning，不产生 diff |

---

## 5. 分类枚举（必须固定）

Action 靠 `category` 分组，因此**枚举必须封闭**。取值与当前 README 一致，另加「学习与教程」：

| 枚举值 | 用途 |
|---|---|
| `安全与治理` | 权限、审批、审计、拦截 |
| `可观测与复盘` | 日志、监控、会话导出 |
| `知识与研究` | 知识库、检索、文献 |
| `开发提效` | 代码扫描、任务管理 |
| `交互与提醒` | 通知、桌面集成 |
| `趣味与氛围` | 游戏、装饰、娱乐 |
| `学习与教程` | 教学、示例、文档站 |
| `未分类` | 兜底（应始终为空） |

新增分类需同步改枚举常量（`scripts/lib/categories.mjs`），否则落入「未分类」。

---

## 6. README 标记区约定

### 6.1 插入两个独立标记对

```markdown
## 插件总览

<!-- BEGIN:OVERVIEW (由 .github/workflows/sync-plugins.yml 自动生成，请勿手改) -->
…生成的表格…
<!-- END:OVERVIEW -->

---

## 插件详解

<!-- BEGIN:DETAILS (由 .github/workflows/sync-plugins.yml 自动生成，请勿手改) -->
…生成的分节…
<!-- END:DETAILS -->
```

**为什么两个标记对而不是一个**：总览表变更是高频的（版本/新增），详解变更是低频的。分开后可以只重建其中一段，也让 review 时的 diff 更聚焦。

### 6.2 标记区外永不改动

以下章节由人维护，Action 只读写标记区之间的内容：

- 标题与描述（H1 + 开篇段落）
- 目录（TOC）
- 快速开始
- 相关项目
- 环境要求、贡献、License

> ⚠️ **例外**：TOC 中的分类锚点依赖 §5 枚举。若分类枚举变化，TOC 需人工同步。可在脚本中额外生成 TOC（`<!-- BEGIN:TOC -->`）以彻底自动化——**列为可选增强，默认不做**。

### 6.3 自愈行为

若标记缺失或不成对，脚本**必须立即失败并退出非零**，绝不猜测插入位置。这能防止最坏情况（把用户手写内容整段覆盖）。

---

## 7. 版本号策略

### 方案对比

| 方案 | 优点 | 缺点 |
|---|---|---|
| **A. shields.io 动态徽章** | 零维护，实时准确 | 表格变宽；依赖 shields.io 可用性 |
| **B. Action 每周写入版本** | 表格紧凑 | 最多滞后一周；仍会产生 diff |
| **C. 不显示版本** | 最简单 | 失去「这个包是否活跃」的信号 |

**建议 A**，并在表格「版本」列渲染：

```markdown
[![npm](https://img.shields.io/npm/v/dsh-plugin-todo-scanner?label=)](https://www.npmjs.com/package/dsh-plugin-todo-scanner)
```

未发布 npm 的包显示静态文本 `—`。

---

## 8. 工作流设计

### 8.1 触发方式

| 触发 | 支持 | 说明 |
|---|---|---|
| `schedule` | ✅ | 每周一 03:00 UTC（北京时间 11:00） |
| `workflow_dispatch` | ✅ | 手动立即同步 |
| `push`（本仓库） | ❌ | 会与自动提交形成递归 |
| 新仓库创建事件 | ❌ | **GitHub 不提供该事件** |
| `repository_dispatch` | ⏳ 阶段 3 | 插件发版时秒级通知，需跨仓库 token |

> **注意**：GitHub Actions 没有「仓库被创建」这一触发事件，所以「自动发现新插件」只能靠**定时轮询**，无法做到事件驱动（除非插件仓库主动通知）。

### 8.2 `.github/workflows/sync-plugins.yml`

```yaml
name: Sync plugin list

on:
  schedule:
    - cron: '0 3 * * 1'      # 每周一 03:00 UTC
  workflow_dispatch:

permissions:
  contents: write            # 提交 README
  issues: write              # 登记提醒

concurrency:
  group: sync-plugin-list
  cancel-in-progress: true

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: 生成清单
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: node scripts/sync-plugins.mjs

      - name: 有变化才提交
        run: |
          if [ -z "$(git status --porcelain README.md)" ]; then
            echo "README 无变化，跳过提交"
            exit 0
          fi
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add README.md
          git commit -m "chore: 自动同步插件清单"
          git push
```

**要点说明**

- `concurrency` 防止手动触发与定时任务撞车
- 未加 `--force`，标记区缺失时脚本已提前失败，不会误提交
- 若后续启用分支保护，把最后一步改为 `peter-evans/create-pull-request` 开 PR

### 8.3 阶段 3（可选）：`repository_dispatch`

在每个插件仓库的 release 流程追加：

```yaml
# dsh-plugin-xxx/.github/workflows/notify-list.yml
name: Notify plugin list
on:
  release:
    types: [published]
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.LIST_DISPATCH_TOKEN }}   # fine-grained PAT，仅需 list 仓库 contents:write
          repository: zhaoxuejie/moqian-dsh-plugin-list
          event-type: plugin-updated
```

并在清单仓库的 workflow 中追加：

```yaml
on:
  repository_dispatch:
    types: [plugin-updated]
```

**代价**：需维护一个跨仓库 PAT（有过期风险）。**收益**：发版后秒级同步。**建议**：先落地阶段 1+2，确有实时需求再加。

---

## 9. 同步脚本设计（`scripts/sync-plugins.mjs`）

### 9.1 流程

```
1. 发现候选仓库
2. 拉取每个仓库的 package.json
3. 校验 dsh.list
4. 判断 npm 发布状态
5. 渲染 OVERVIEW / DETAILS
6. 切分 README 标记区并写回
7. 汇总 warning → 开/更新 Issue
```

### 9.2 发现候选仓库

```
GET /search/repositories?q=user:zhaoxuejie+dsh-plugin+in:name&per_page=100
```

**必须处理的坑**：

| 坑 | 处理 |
|---|---|
| 本清单仓库名 `moqian-dsh-plugin-list` **也含 `dsh-plugin`**，会被搜出来 | 显式排除 `moqian-dsh-plugin-list` |
| 已归档仓库 | 跳过（并提示） |
| fork | 跳过 |
| 命名不含 `dsh-plugin` 的自有插件（如 `dsh-daily-digest`） | 由 `plugins.extra.yml` 显式补充收录 |

`plugins.extra.yml` 示例：

```yaml
# 命名不符合 dsh-plugin-* 约定、但希望收录的自有插件
extra:
  - dsh-daily-digest
```

> 该文件是**唯一**允许出现在清单仓库的手写清单，且只写包名，不写描述。

### 9.3 校验与告警

| 检查 | 处理 |
|---|---|
| 缺 `dsh.list` | 收录到「未分类」+ 汇总进 Issue |
| `category` 非法 | 同上 |
| `tagline` 缺失/超长 | 回退 `description` + 告警 |
| 包名与仓库名不一致 | 以 `package.json` 的 `name` 为准 + 告警 |
| `dsh.list.exclude === true` | 跳过，不告警 |

**Issue 去重**：查找标题为 `[auto] 待登记的插件` 的开启中 Issue，存在则编辑正文，否则新建。避免每周刷一个新 Issue。

### 9.4 渲染

- **排序必须稳定**：`分类枚举顺序` → `order`（缺省 `Infinity`）→ `包名` 升序。否则每周产生无意义 diff。
- **同一次运行内不重复请求**：按包名缓存 API 结果。
- **重试**：GitHub/npm 请求失败重试 3 次（指数退避）；仍失败则该插件沿用上次渲染结果（从现有 README 解析回读，或整体放弃本次运行）。

### 9.5 幂等性（G4 的关键）

生成结果必须**只依赖输入**，不依赖时间、随机数、API 返回顺序。可行做法：

- 不写入「最后更新时间」之类的动态内容
- 依赖数据全部排序后再拼接
- 换行符统一 `\n`（配合 `.gitattributes` 的 `eol=lf`）

---

## 10. 迁移步骤

> 全程可回滚：每个阶段都是独立 commit，出问题 `git revert` 即可。

### 阶段 0：准备标记区（低风险）

1. 在 README 插入 §6.1 的两个标记对，**先不删原有内容**
2. 用现有 9 个插件的数据试跑脚本，确认生成结果与现有内容**语义等价**
3. 确认无误后，删除标记区内的手写内容，保留生成结果
4. 提交

**验收**：`git diff` 只体现为「内容移入标记区」，无信息丢失。

### 阶段 1：补齐 9 个插件的 `dsh.list`

给这 9 个插件的 `package.json` 补字段：

`dsh-plugin-tool-guard`、`dsh-plugin-log-forwarder`、`dsh-plugin-session-export`、`dsh-plugin-vault-memory`、`dsh-plugin-academic-paper`、`dsh-plugin-todo-scanner`、`dsh-plugin-desktop-notice`、`dsh-plugin-feihualing`、`dsh-plugin-internet-meme`

> ✅ **无需发新 npm 版本**——因为 §3.3 采用 GitHub 作为主读取源，提交即生效。

### 阶段 2：上线 Action

1. 添加 `scripts/sync-plugins.mjs`、`scripts/lib/categories.mjs`、`plugins.extra.yml`
2. 添加 `.github/workflows/sync-plugins.yml`
3. `workflow_dispatch` 手动跑一次验证
4. 开启定时

**验收**：手动连跑两次，第二次 `README 无变化，跳过提交`（幂等）。

### 阶段 3（可选）：事件驱动

按 §8.3 加 `repository_dispatch`。

---

## 11. 风险与缓解

| 风险 | 影响 | 缓解 |
|---|---|---|
| 误覆盖手写内容 | **最严重**，文档损坏 | 标记区隔离 + 标记不成对时失败退出 + 阶段 0 先验证 |
| 自动提交刷 commit 记录 | 仓库历史噪音 | 幂等排序 + 有 diff 才提交；或改用开 PR |
| API 限流/故障 | 同步失败 | 认证请求（5000 req/h）+ 重试 + 失败不写文件 |
| 分类枚举漂移 | 落入「未分类」 | 校验 + Issue 提醒 |
| `GITHUB_TOKEN` 提交不触发其他 workflow | 无害 | 本场景无需链式触发 |
| 分支保护阻止 push | Action 失败 | 改用开 PR |
| 依赖 shields.io | 徽章不显示 | 可回退到方案 B（写入静态版本） |
| 收录范围失控（第三方插件混入） | 清单变质 | 发现源限定 `user:zhaoxuejie`，第三方需显式加入 `plugins.extra.yml` |

---

## 12. 待决策清单

落地前需确认：

- [ ] **触发方式**：仅 `schedule` + `workflow_dispatch`，还是也要阶段 3 的 `repository_dispatch`？
- [ ] **提交方式**：Action 直接 push，还是开 PR 由你合并？
- [ ] **版本展示**：shields.io 徽章（方案 A）还是静态写入（方案 B）？
- [ ] **TOC 是否也自动化**（加第三个标记区）？
- [ ] **是否收录 npm 未发布的插件**？（当前策略：收录，安装命令用 `github:`）
- [ ] **分类枚举是否需要调整**？（现 7 类 + 兜底）

---

## 附录 A：生成结果示例

### A.1 OVERVIEW 区

<!-- 以下为示意，非实际生成内容 -->

```markdown
| 插件 | 用途 | npm | 语言 | 仓库 |
|---|---|---|---|---|
| [`dsh-plugin-tool-guard`](#dsh-plugin-tool-guard) | 工具调用安全守卫（危险命令拦截、路径白名单、人工审批） | [![npm](https://img.shields.io/npm/v/dsh-plugin-tool-guard?label=)](https://www.npmjs.com/package/dsh-plugin-tool-guard) | TypeScript | [↗](https://github.com/zhaoxuejie/dsh-plugin-tool-guard) |
```

### A.2 DETAILS 区

<!-- 以下为示意，非实际生成内容 -->

````markdown
#### dsh-plugin-tool-guard

给 AI 的每一次工具调用装上防火墙 —— 危险命令拦截 · 禁止读取敏感文件 · 限定可操作目录 · 高危操作人工审批 · 全量审计留痕。

```bash
dsh plugin --profile web add dsh-plugin-tool-guard
```

**亮点**

- 规则引擎拦截危险命令，路径白名单 + 敏感文件黑名单双重防线
- 高危操作触发人工审批，全量审计日志可追溯

**链接**：[GitHub](https://github.com/zhaoxuejie/dsh-plugin-tool-guard) · [npm](https://www.npmjs.com/package/dsh-plugin-tool-guard)
````

---

## 附录 B：与现有约定的兼容性

| 现有约定 | 本方案是否冲突 | 说明 |
|---|---|---|
| `dsh.bundle.patch` | 否 | `dsh.list` 与 `dsh.bundle` 并列 |
| `dsh.client.platform` | 否 | 同上 |
| npm `files` 白名单 | 否 | `package.json` 始终被打包 |
| 插件 README 结构 | 否 | 不再从 README 提取，改从 `package.json` 读 |
| 各插件 CHANGELOG | 否 | 本方案不触碰 |

---

## 附录 C：为何不直接从各插件 README 提取文案

初版设想是让 Action 读取每个插件的 README，抓取「一句话简介」和「亮点」。实测 9 个插件的 README 结构**并不统一**：

| 插件 | 简介形式 | 亮点形式 |
|---|---|---|
| `dsh-plugin-todo-scanner` | `>` 引用块 | 无序列表 + `**加粗**` |
| `dsh-plugin-log-forwarder` | 正文段落 | Markdown 表格 |
| `dsh-plugin-tool-guard` | 徽章 + 加粗句 | 无统一列表 |
| `dsh-plugin-internet-meme` | 正文段落 | 表格 |
| `dsh-plugin-academic-paper` | 正文段落 | 无序列表（无加粗） |

**结论**：从 README 提取需要脆弱的正则/启发式规则，且一旦作者改动 README 结构就会静默失效。**显式结构化字段（`dsh.list`）比启发式解析可靠得多**，这是本方案选择前者的直接原因。
