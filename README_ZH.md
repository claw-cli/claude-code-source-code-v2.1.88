# Claude Code（泄露源码归档）

[English](./README.md) | 中文 | [繁體中文](./README_ZH_TW.md) | [조선어](./README_KO.md) | [日本語](./README_JA.md) | [Español](./README_ES.md) | [Русский](./README_RU.md)

<p align="center">
  <img src="https://img.shields.io/badge/status-archived%20%2B%20rebuilding-blue" />
  <img src="https://img.shields.io/badge/runtime-Bun%20%2F%20Node-black" />
  <img src="https://img.shields.io/badge/language-TypeScript%20→%20Rust-orange" />
  <img src="https://img.shields.io/badge/focus-Code%20Agent%20Systems-green" />
</p>

---

## 🚨 更新 → 正在用 Rust 重建

> **构建更好的 harness 工具，而不只是囤积泄露出来的 Claude Code。  
不要收集代码。要创造结果。**

我们正在积极地**用 Rust 重建 Claude Code**，目标是打造一个**更强大、更可靠、更适合生产环境的代码智能体系统**。

👉 如果你对 Rust 实现和最新进展感兴趣：  
https://github.com/claw-cli/claw-code-cli

---

## 📦 这个仓库是什么？

这个仓库是 **Claude Code v2.1.88 源代码的归档快照**。这份源码因为被打包进 npm 包中的 sourcemap 文件，而变得可以被公开访问。

它的用途包括：

- 📚 作为研究真实世界 AI 智能体系统的**研究材料**  
- 🔍 作为大规模 LLM 工具架构的**参考实现**  
- 🧪 作为**重建更好系统的基础**

---

2026 年 3 月 31 日，寿超凡发现了一件不同寻常的事：Claude Code 的整套源代码被悄悄发布到了 npm 上，并隐藏在随包附带的一个 sourcemap 文件里。

[![宣布泄露的推文](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)

这个仓库记录了那个时刻。它既是对这份暴露代码的归档，也是对泄露如何发生的拆解，同时还展示了这套系统幕后的一些信息。

我们开始吧。

## 这件事是怎么发生的？

这部分真的会让你停下来想一秒。

当一个 JavaScript 或 TypeScript 包发布到 npm 时，现代构建流水线通常会生成**source map 文件**（`.map`）。这些文件充当打包后或压缩后的生产输出与原始源代码之间的桥梁。它们的目的很直接：当生产环境里出了问题时，堆栈追踪可以映射回原始源码中的确切行号，而不是指向压缩 bundle 里某个难以阅读的位置。

但关键细节在于，**source map 往往会直接嵌入原始源码本身**。不是引用，而是真正的原始代码，以字符串形式存储在一个 JSON 结构里。

一个典型的 `.map` 文件大概长这样：

```json
{
  "version": 3,
  "sources": ["../src/main.tsx", "../src/tools/BashTool.ts", "..."],
  "sourcesContent": ["// 每个文件的完整原始源码", "..."],
  "mappings": "AAAA,SAAS,OAAO..."
}
````

那个 `sourcesContent` 字段就是关键。它可以包含每一个原始文件的完整内容：代码、注释、内部常量、提示词，所有东西。如果这个文件被发布出去，那么整个代码库实际上也就跟着一起发出去了。

而 npm 会很乐意分发它。任何执行 `npm pack`、检查 tarball，或者浏览包内容的人，都可以直接访问到它。

这并不是一种新型问题。它以前已经发生过很多次，而且未来大概率还会再次发生。根本原因通常很简单：要么 `*.map` 文件没有通过 `.npmignore` 被排除掉，要么打包工具没有被配置为在生产构建中禁用 source map 生成。

而在这个案例中，项目是用 Bun 构建的。Bun 默认会生成 source map，除非显式禁用，这让这种暴露变得非常容易被忽略。

![npm 包中暴露的 Claude Code 源文件](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)

尤其讽刺的是，代码里还存在一个名为“Undercover Mode”的内部系统，专门用来防止敏感信息通过提交信息或 PR 描述等生成内容泄露出去。

为了确保 AI 不会在文本中意外暴露内部细节，他们投入了相当多的精力，但最终整个代码库却通过一个构建产物被发布出去了。

---

## Claude 的底层到底是什么样？

如果你最近不是与世隔绝，那你大概知道，Claude Code 是 Anthropic 官方推出的、用于配合 Claude 编码的 CLI 工具，也是目前最受欢迎的 AI 编码智能体。

从外面看，它像是一个经过精心打磨、但相对简单的 CLI。

从内部看，它是一个 **785KB 的 [`main.tsx`](https://github.com/kuberwastaken/claude-code/blob/main/main.tsx)** 入口点，外加一个自定义 React 终端渲染器、40 多个工具、一个多智能体编排系统、一个名为 “dream” 的后台记忆整合引擎，以及更多内容。

不多废话了，下面是我花了一整个下午深挖源码后发现的一些真的很酷的部分：

---

## BUDDY - 你终端里的电子宠物

我不是在编。

Claude Code 里真的有一整套完整的**类电子宠物同伴系统**，名字叫 “Buddy”。它有一个**确定性的扭蛋系统**，带有物种稀有度、闪光变体、程序化生成的属性，以及像 OpenClaw 一样，在第一次孵化时由 Claude 写出的灵魂描述。

整套系统都放在 [`buddy/`](https://github.com/kuberwastaken/claude-code/tree/main/buddy) 目录下，并由 `BUDDY` 编译期开关控制。

### 扭蛋系统

你的 buddy 物种由一个 **Mulberry32 PRNG** 决定，这是一个快速的 32 位伪随机数生成器，它以你的 `userId` 哈希加上盐值 `'friend-2026-401'` 作为种子：

```typescript
// Mulberry32 PRNG - 对每个用户都是确定且可复现的
function mulberry32(seed: number): () => number {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    var t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  }
}
```

同一个用户永远会得到同一个 buddy。

### 18 个物种（在代码里做了混淆）

物种名称通过 `String.fromCharCode()` 数组隐藏起来了，Anthropic 显然不想让这些名字能被字符串搜索轻易发现。解码后，完整的物种列表如下：

| 稀有度 | 物种 |
|--------|---------|
| **普通** (60%) | Pebblecrab, Dustbunny, Mossfrog, Twigling, Dewdrop, Puddlefish |
| **不常见** (25%) | Cloudferret, Gustowl, Bramblebear, Thornfox |
| **稀有** (10%) | Crystaldrake, Deepstag, Lavapup |
| **史诗** (4%) | Stormwyrm, Voidcat, Aetherling |
| **传说** (1%) | Cosmoshale, Nebulynx |

除此之外，还有一个**完全独立于稀有度的 1% 闪光概率**。所以一只闪光传说级 Nebulynx 的出现概率是 **0.01%**。离谱。

### 属性、眼睛、帽子和灵魂

每个 buddy 都会被程序化生成：
- **5 项属性**：`DEBUGGING`、`PATIENCE`、`CHAOS`、`WISDOM`、`SNARK`（每项 0-100）
- **6 种可能的眼睛样式**和**8 种帽子选项**（其中一些受稀有度限制）
- **一个“灵魂”**，正如前面提到的那样，是 Claude 在首次孵化时以角色口吻生成的人格描述

这些 sprite 会被渲染为**5 行高、12 字符宽的 ASCII 艺术图**，并带有多个动画帧。它们有待机动画、反应动画，还会坐在你的输入提示框旁边。

### 设定

代码中把 2026 年 4 月 1 日到 7 日标记为一个**预热视频窗口**（所以大概是为了复活节？），而完整上线则被锁定在 2026 年 5 月。这个同伴有一段系统提示词，会告诉 Claude：

```
A small {species} named {name} sits beside the user's input box and 
occasionally comments in a speech bubble. You're not {name} - it's a 
separate watcher.
```

所以它不只是个装饰品，这个 buddy 有自己的人格，而且在别人叫它名字时还能作出回应。我真的希望他们会把它正式发布出来。

---

## KAIROS - “常驻在线的 Claude”

在 [`assistant/`](https://github.com/kuberwastaken/claude-code/tree/main/assistant) 目录中，有一个完整的模式叫 **KAIROS**，也就是一个持久化、始终运行的 Claude 助手，它不会等你输入才行动。它会观察、记录，并对它注意到的事情进行**主动**响应。

这个模式由 `PROACTIVE` / `KAIROS` 编译期开关控制，在对外构建中是完全不存在的。

### 它是怎么工作的

KAIROS 会维护**只追加的每日日志文件**，在一天中持续写入观察、决策和动作。系统会按固定间隔向它发送 `<tick>` 提示，让它自行决定是主动采取行动，还是保持安静。

这个系统有一个**15 秒阻塞预算**，任何会让用户工作流阻塞超过 15 秒的主动行为都会被延后。这是 Claude 在试图“帮忙”而不是“添乱”。

### Brief 模式

当 KAIROS 启用时，会有一种特殊的输出模式叫 **Brief**，它会给出极其简洁的回应，适合这种持续驻留、但又不该刷满你终端的助手。你可以把它理解成一个话很多的朋友，和一个只在真正有价值时才开口的专业助理之间的区别。

### 专属工具

KAIROS 能使用一些普通 Claude Code 没有的工具：

| 工具 | 作用 |
|------|-------------|
| **SendUserFile** | 直接把文件推送给用户（通知、摘要） |
| **PushNotification** | 向用户设备发送推送通知 |
| **SubscribePR** | 订阅并监控 PR 活动 |

 ---

## ULTRAPLAN - 30 分钟远程规划会话

这个从基础设施角度来看也很离谱。

**ULTRAPLAN** 是一种模式，在这种模式下，Claude Code 会把一个复杂的规划任务卸载到一个运行 **Opus 4.6** 的**远程 Cloud Container Runtime（CCR）会话**中，给它最多 **30 分钟** 的思考时间，然后让你在浏览器里审批结果。

基本流程如下：

1. Claude Code 识别出一个需要深度规划的任务
2. 它通过 `tengu_ultraplan_model` 配置启动一个远程 CCR 会话
3. 你的终端会显示轮询状态，每 **3 秒** 检查一次结果
4. 与此同时，一个基于浏览器的 UI 会让你看到规划过程，并可选择批准或拒绝
5. 当结果被批准后，会有一个特殊的哨兵值 `__ULTRAPLAN_TELEPORT_LOCAL__`，把结果“传送”回你的本地终端

---

## “Dream” 系统 - Claude 真的会做梦

好吧，这真的是这里面最酷的东西之一。

Claude Code 有一个叫 **autoDream** 的系统（[`services/autoDream/`](https://github.com/kuberwastaken/claude-code/tree/main/services/autoDream)） - 一个以**分叉子智能体**形式运行的后台记忆整合引擎。这个命名非常刻意。Claude 在……做梦。

这件事特别有意思，因为[我上周刚在 LITMUS 里有过类似想法，让 OpenClaw 的子智能体有点闲暇时间去找有趣的新论文](https://github.com/Kuberwastaken/litmus)

### 三重门触发器

dream 并不会在它“想跑”的时候就跑。它有一个**三重门触发系统**：

1. **时间门**：距离上一次 dream 已过去 24 小时
2. **会话门**：自上一次 dream 以来至少已有 5 次会话  
3. **锁门**：获取整合锁（防止并发 dream）

三者必须同时通过。这能防止 dream 过多，也防止 dream 过少。

### 四个阶段

当它运行时，dream 会遵循 [`consolidationPrompt.ts`](https://github.com/kuberwastaken/claude-code/blob/main/services/autoDream/consolidationPrompt.ts) 中提示词定义的四个严格阶段：

**阶段 1 - 定向**：对记忆目录执行 `ls`，读取 `MEMORY.md`，快速浏览现有主题文件，寻找可改进之处。

**阶段 2 - 收集近期信号**：找出值得持久保存的新信息。优先级来源为：每日日志 → 漂移的记忆 → 转录搜索。

**阶段 3 - 整合**：写入或更新记忆文件。把相对日期转换成绝对日期。删除已被矛盾事实推翻的信息。

**阶段 4 - 修剪与索引**：保持 `MEMORY.md` 不超过 200 行且约 25KB。移除过时指针。解决矛盾。

提示词里甚至直接写着：

> *“你正在执行一次 dream，这是一轮对你的记忆文件进行反思式整理的过程。把你最近学到的内容综合成持久、组织良好的记忆，以便未来的会话能够快速完成定向。”*

dream 子智能体只获得了**只读 bash** 权限，它可以查看你的项目，但不能修改任何内容。它纯粹是一轮记忆整合。

---

## 完整工具注册表 - 40+ 个工具

Claude Code 的工具系统位于 [`tools/`](https://github.com/kuberwastaken/claude-code/tree/main/tools)。下面是完整列表：

| 工具 | 作用 |
|------|-------------|
| **AgentTool** | 生成子 agent / subagent |
| **BashTool** / **PowerShellTool** | Shell 执行（可选沙箱） |
| **FileReadTool** / **FileEditTool** / **FileWriteTool** | 文件操作 |
| **GlobTool** / **GrepTool** | 文件搜索（可用时使用原生 `bfs`/`ugrep`） |
| **WebFetchTool** / **WebSearchTool** / **WebBrowserTool** | Web 访问 |
| **NotebookEditTool** | Jupyter notebook 编辑 |
| **SkillTool** | 调用用户定义的技能 |
| **REPLTool** | 交互式 VM shell（裸模式） |
| **LSPTool** | Language Server Protocol 通信 |
| **AskUserQuestionTool** | 向用户请求输入 |
| **EnterPlanModeTool** / **ExitPlanModeV2Tool** | Plan 模式控制 |
| **BriefTool** | 上传/总结文件到 claude.ai |
| **SendMessageTool** / **TeamCreateTool** / **TeamDeleteTool** | Agent swarm 管理 |
| **TaskCreateTool** / **TaskGetTool** / **TaskListTool** / **TaskUpdateTool** / **TaskOutputTool** / **TaskStopTool** | 后台任务管理 |
| **TodoWriteTool** | 写入待办（旧版） |
| **ListMcpResourcesTool** / **ReadMcpResourceTool** | MCP 资源访问 |
| **SleepTool** | 异步延迟 |
| **SnipTool** | 历史片段提取 |
| **ToolSearchTool** | 工具发现 |
| **ListPeersTool** | 列出 peer agents（UDS inbox） |
| **MonitorTool** | 监控 MCP 服务器 |
| **EnterWorktreeTool** / **ExitWorktreeTool** | Git worktree 管理 |
| **ScheduleCronTool** | 安排 cron 任务 |
| **RemoteTriggerTool** | 触发远程 agents |
| **WorkflowTool** | 执行 workflow 脚本 |
| **ConfigTool** | 修改设置（**仅内部**） |
| **TungstenTool** | 高级功能（**仅内部**） |
| **SendUserFile** / **PushNotification** / **SubscribePR** | KAIROS 专属工具 |

工具通过 `getAllBaseTools()` 注册，并根据功能开关、用户类型、环境标志和权限拒绝规则进行过滤。这里还有一个**工具 schema 缓存**（[`toolSchemaCache.ts`](https://github.com/kuberwastaken/claude-code/blob/main/tools/toolSchemaCache.ts)），用来缓存 JSON schema，以提升提示词效率。

---

## 权限与安全系统

Claude Code 在 [`tools/permissions/`](https://github.com/kuberwastaken/claude-code/tree/main/tools/permissions) 中的权限系统，远不只是简单的“允许/拒绝”：

**权限模式**：`default`（交互式提示）、`auto`（通过转录分类器进行基于 ML 的自动批准）、`bypass`（跳过检查）、`yolo`（拒绝全部，名字起得很反讽）

**风险分类**：每个工具动作都会被归类为 **LOW**、**MEDIUM** 或 **HIGH** 风险。这里还有一个 **YOLO classifier**，这是一个快速的、基于 ML 的权限决策系统，可以自动作出判断。

**受保护文件**：`.gitconfig`、`.bashrc`、`.zshrc`、`.mcp.json`、`.claude.json` 等文件都被保护，不能被自动编辑。

**路径穿越防护**：URL 编码穿越、Unicode 规范化攻击、反斜杠注入、大小写不敏感路径操控，全部都有处理。

**权限解释器**：在用户批准前，会额外调用一次 LLM，向用户解释工具风险。当 Claude 说“这个命令会修改你的 git config”时，这段解释本身也是 Claude 生成的。

---

## 隐藏的 Beta Header 与未发布的 API 功能

[`constants/betas.ts`](https://github.com/kuberwastaken/claude-code/blob/main/constants/betas.ts) 文件揭示了 Claude Code 与 API 协商的每一个 beta 功能：

```typescript
'interleaved-thinking-2025-05-14'      // 扩展思维
'context-1m-2025-08-07'                // 1M token 上下文窗口
'structured-outputs-2025-12-15'        // 结构化输出格式
'web-search-2025-03-05'                // Web 搜索
'advanced-tool-use-2025-11-20'         // 高级工具使用
'effort-2025-11-24'                    // Effort 级别控制
'task-budgets-2026-03-13'              // 任务预算管理
'prompt-caching-scope-2026-01-05'      // Prompt 缓存作用域
'fast-mode-2026-02-01'                 // Fast 模式（Penguin）
'redact-thinking-2026-02-12'           // 思维内容脱敏
'token-efficient-tools-2026-03-28'     // Token 高效工具 schema
'afk-mode-2026-01-31'                  // AFK 模式
'cli-internal-2026-02-09'             // 仅内部使用（ant）
'advisor-tool-2026-03-01'              // Advisor 工具
'summarize-connector-text-2026-03-13'  // Connector 文本摘要
```

`redact-thinking`、`afk-mode` 和 `advisor-tool` 也都尚未发布。

---

## 功能门控 - 内部构建 vs. 外部构建

这是整个代码库里从架构角度最有意思的部分之一。

Claude Code 通过 Bun 的 `bun:bundle` 提供的 `feature()` 函数，使用**编译期功能开关**。打包器会对这些开关做**常量折叠**，并在外部构建中对受控分支进行**死代码消除**。已知的完整开关列表如下：

| Flag | 控制的功能 |
|------|--------------|
| `PROACTIVE` / `KAIROS` | 常驻在线助手模式 |
| `KAIROS_BRIEF` | Brief 命令 |
| `BRIDGE_MODE` | 通过 claude.ai 进行远程控制 |
| `DAEMON` | 后台守护进程模式 |
| `VOICE_MODE` | 语音输入 |
| `WORKFLOW_SCRIPTS` | Workflow 自动化 |
| `COORDINATOR_MODE` | 多智能体编排 |
| `TRANSCRIPT_CLASSIFIER` | AFK 模式（ML 自动批准） |
| `BUDDY` | 同伴宠物系统 |
| `NATIVE_CLIENT_ATTESTATION` | 客户端证明 |
| `HISTORY_SNIP` | 历史裁剪 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能发现 |

此外，`USER_TYPE === 'ant'` 会控制 Anthropic 内部功能：staging API 访问（`claude-ai.staging.ant.dev`）、内部 beta header、Undercover mode、`/security-review` 命令、`ConfigTool`、`TungstenTool`，以及把调试提示词导出到 `~/.config/claude/dump-prompts/`。

**GrowthBook** 负责运行时功能门控，并使用了激进的缓存值。以 `tengu_` 为前缀的 feature flag 控制着从 fast mode 到记忆整合的一切。许多检查使用 `getFeatureValue_CACHED_MAY_BE_STALE()` 来避免阻塞主循环，也就是说，对于 feature gate 来说，使用可能过期的数据也是可接受的。

---

## 其他值得注意的发现

### 上游代理
[`upstreamproxy/`](https://github.com/kuberwastaken/claude-code/tree/main/upstreamproxy) 目录包含一个面向容器环境的代理中继，它使用 **`prctl(PR_SET_DUMPABLE, 0)`** 来防止同 UID 进程通过 ptrace 读取堆内存。它会从 CCR 容器中的 `/run/ccr/session_token` 读取会话 token，下载 CA 证书，并启动一个本地的 CONNECT→WebSocket 中继。Anthropic API、GitHub、npmjs.org 和 pypi.org 被明确排除在代理之外。

### Bridge 模式
[`bridge/`](https://github.com/kuberwastaken/claude-code/tree/main/bridge) 目录中有一个基于 JWT 认证的 bridge 系统，用于与 claude.ai 集成。支持的工作模式有：`'single-session'` | `'worktree'` | `'same-dir'`。其中还包含受信设备 token，用于更高等级的安全层级。

### 迁移中的模型代号
[`migrations/`](https://github.com/kuberwastaken/claude-code/tree/main/migrations) 目录揭示了内部代号的演变历史：
- `migrateFennecToOpus` - **“Fennec”**（耳廓狐）曾是 Opus 的代号
- `migrateSonnet1mToSonnet45` - 带 1M 上下文的 Sonnet 变成了 Sonnet 4.5
- `migrateSonnet45ToSonnet46` - Sonnet 4.5 → Sonnet 4.6
- `resetProToOpusDefault` - Pro 用户在某个时候被重置回默认使用 Opus

### Attribution Header
每个 API 请求都会包含：
```
x-anthropic-billing-header: cc_version={VERSION}.{FINGERPRINT}; 
  cc_entrypoint={ENTRYPOINT}; cch={ATTESTATION_PLACEHOLDER}; cc_workload={WORKLOAD};
```
`NATIVE_CLIENT_ATTESTATION` 功能会让 Bun 的 HTTP 栈把 `cch=00000` 这个占位符替换成计算出的哈希，本质上是一种客户端真实性检查，让 Anthropic 能验证请求是否来自一个真实的 Claude Code 安装实例。

### Computer Use - “Chicago”
Claude Code 包含完整的 Computer Use 实现，内部代号为 **“Chicago”**，构建于 `@ant/computer-use-mcp` 之上。它提供截图捕获、点击/键盘输入以及坐标变换能力。此功能仅对 Max/Pro 订阅开放（内部用户可通过 ant 绕过）。

### 定价
如果有人好奇的话，[`utils/modelCost.ts`](https://github.com/kuberwastaken/claude-code/blob/main/utils/modelCost.ts) 里的所有定价都和 [Anthropic 公开定价](https://docs.anthropic.com/en/docs/about-claude/models) 完全一致。这里没有什么新闻价值。

---

## ⚡ TL;DR

- Claude Code 远不只是一个 CLI，它是一个**完整的智能体平台**
- 包含：
  - 多智能体编排
  - 后台记忆（“Dream 系统”）
  - 工具体系（40+ 工具）
  - 主动式助手（KAIROS）
- 大量使用：
  - Feature flags
  - 运行时 + 编译期门控
  - 作为系统设计的 Prompt engineering

---

## 🧩 关键架构洞察

### 1. 多智能体编排

Claude Code 可以运行在**协调者模式**下，生成多个 worker：

- 并行研究
- 集中式规划
- 分布式执行
- 验证循环

> 并行性被当作一等原语来对待。

---

### 2. 工具系统（40+ 工具）

一个丰富的工具生态，包括：

- 文件系统操作
- Shell 执行
- Web 浏览
- 任务调度
- 智能体通信

所有工具都具备：
- Schema 驱动
- 权限门控
- 动态启用

---

### 3. 记忆系统（“Dream”）

一个后台进程，它会：

- 定期运行
- 整合知识
- 修剪过时信息
- 维护长期记忆

> 这本质上就是 **LLM 记忆压缩 + 反思**。

---

### 4. 主动式智能体（KAIROS）

一个始终在线的助手，它会：

- 观察活动
- 记录行为
- 在没有显式提示的情况下采取行动

这就是 **agent → system 演化**。

---

### 5. 功能门控与构建策略

- 编译期开关（通过 Bun）
- 死代码消除
- 内部与外部构建区分

这使得下面这些成为可能：
- 隐藏功能
- 渐进式发布
- 内部实验

---

## 🧠 我们学到了什么

- AI 编码工具正在变成**操作系统**，而不只是助手
- Prompt engineering = **系统架构**
- 记忆 + 工具 + 编排 = **真正的智能体**
- 生产系统高度依赖：
  - Guardrails
  - Permissions
  - Observability

---

## 🛠 为什么要用 Rust 重建？

这个代码库很强大，但是：

- ❌ 复杂且难以维护
- ❌ JS 运行时存在局限
- ❌ 性能保证较弱
- ❌ 并发行为难以推理

我们相信下一代智能体系统应该具备：

- ⚡ 更快（原生性能）
- 🔒 更安全（内存 + 执行）
- 🧵 从设计上支持并发
- 📦 更适合分发（CLI + 基础设施）

---

## 🚀 我们的方向

我们正在构建：

> 一个**下一代代码智能体运行时**，而不只是一个 CLI 包装器。

重点方向：

- 确定性的智能体执行
- 更好的工具沙箱
- 一等公民级别的多智能体编排
- 真正的记忆系统（而不是 prompt hack）
- 对 Bun / npm 友好的分发流水线

---

## 📦 构建与分发

目前正在推进：

- ✅ 基于 Bun 的构建流水线
- 📦 npm 分发
- ⚡ Bun 原生执行

目标：

> 无缝安装，即刻执行，零摩擦。

---

## ⚠️ 免责声明

这个仓库用于：

- 研究
- 教育
- 逆向工程洞察

我们**不主张拥有**原始 Claude Code 的所有权。

---

## ⭐ 最后说明

这个仓库最初只是一个归档。

现在，它是一个**发射台**。

> 不要收集代码。  
> 去构建真正能交付的系统。
