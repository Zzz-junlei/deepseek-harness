# DeepSeek Harness 功能验证测试体系调研纪要

[English](dsh-functional-verification-analysis.md) | 中文

## 执行摘要

本文档是对 DeepSeek Harness (dsh) 仓库功能验证测试体系的完整源码级调研。DeepSeek Harness 是一个基于 [Cordis](https://github.com/cordiverse/cordis) 框架的「一切皆插件」的 agent harness,采用分层测试策略确保从单元到集成的全方位质量保障。

**关键发现:**

- **七层测试金字塔**: 从单元测试到性能基准,每层职责明确,互不重叠
- **100% 覆盖率门禁**: `packages/*/*/src` 按文件强制 100% 行覆盖,未覆盖行往往是死代码
- **真实 API 优先**: 推理成本低,倡导带密钥真实 API 测试而非过度 mock
- **录制会话快照**: 通过 replay/record/refresh 三模式实现无密钥回放,同时保留完整变更证据
- **事故驱动改进**: postmortem 0001 揭示「绿色单元测试、破损产品」问题,催生真实入口路径测试强制要求

本调研深入到具体符号名、文件路径和 API 签名,可作为添加新测试场景的操作手册。

---

## 目录

1. [仓库定位与整体架构](#1-仓库定位与整体架构)
2. [依赖与运行方式](#2-依赖与运行方式)
3. [功能验证测试专章](#3-功能验证测试专章)
4. [从事故中学习](#4-从事故中学习)
5. [设计取舍与风险](#5-设计取舍与风险)
6. [快速上手指南](#6-快速上手指南)

---

<a id="1-仓库定位与整体架构"></a>
## 1. 仓库定位与整体架构

### 1.1 项目定位

DeepSeek Harness 是由 DeepSeek AI 开发的开源 agent harness（智能体框架）,当前处于**开发者预览**阶段。核心特点:

- **一切皆插件**: 基于 Cordis 框架,模型适配器、工具注册表、会话日志、agent loop 本身均为可替换插件
- **无特权核心**: 不通过打补丁扩展,而是通过挂载插件并排组合
- **声明式组合**: 通过 profile 和 bundle 有序叠加配置层
- **可逆注册**: 所有贡献通过 `ctx.effect()` / `ctx.on()` 注册,插件卸载时自动回退

### 1.2 仓库布局

```
/workspace/
├── vendor/          # Cordis 及其依赖的固定源码副本
├── packages/        # @deepseek-ai/dsh-<pkg> 工作区包
│   ├── core/        # agent/session API
│   ├── llm/         # 模型提供方
│   ├── shell/       # 命令执行
│   ├── terminal/    # 持久终端
│   ├── fs/          # 文件系统访问
│   ├── tool-*/      # 模型面向工具
│   ├── test-support/ # 测试基础设施 ★
│   └── ...
├── apps/
│   ├── cli/         # dsh 命令行入口
│   └── web/         # Web UI 应用
├── snapshots/       # 录制会话快照 ★
│   ├── session/     # headless 场景
│   ├── sdk/         # SDK 协议场景
│   ├── acp/         # ACP 协议场景
│   └── web/         # 浏览器会话场景
├── benchmarks/      # 性能基准门禁
├── scripts/         # 质量门禁与生成器
├── docs/            # 架构与测试文档
└── python/          # Python SDK/runtime
```

### 1.3 核心架构

#### Profile 与 Bundle 组合

一个运行中的 `dsh` 是从有序层启动时组合出的插件树:

- **Profile** (配置集): 命名组合,如 `web`, `headless`, `sdk`, `acp`,列出要叠加的 bundle
- **Bundle** (分发格式): Cordis 配置行及其挂载的代码,例如:
  - `dsh-base`: `web`/`headless`/`sdk`/`acp` 的共享首层(模型适配器、工具、持久化)
  - `dsh-web-app`: 添加浏览器应用
  - `dsh-headless`: 添加一次性运行器
  - `dsh-sdk-app`: 添加 SDK JSON-RPC 服务器
  - `dsh-acp-app`: 添加自动化专用 ACP 服务器

层的应用顺序: 各 bundle 按 profile 列出的顺序 → profile 自身的 `cordis.patch.yml` → home 级别 patch → 任意 `--patch` 覆盖。

#### 核心包职责表

| 包 | 职责 | Context 键 |
|---|---|---|
| `core/session` | 仅追加的 `SessionEvent` 日志与内存存储 | `ctx.sessions` |
| `core/system-prompt` | 提示词段落与工具模式组装 | `ctx.systemPrompt` |
| `core/tools` | 作用域工具注册表与守卫执行管线 | `ctx.tools` |
| `core/agent` | `Agent` 接口、活跃注册表与 `agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现 `Agent` 接口的默认驱动 | `ctx.agentLoop` |
| `core/scope` | 按 agent 划分作用域的注册原语 | 库,无键 |
| `llm/llm` | 消息与流词汇表加适配器接缝 | `ctx.llm` |
| `webhook/webhook` | 认证交付调度与 Workspace Session 创建 | `ctx.webhookRuntime` |

#### 事件域

- **Session 事件**: 持久事实,追加到日志并通过 `session/event` 广播
- **Agent 事件** (`agent/*`): 携带活跃 `Agent`,观察或拦截进行中的工作(pre-step、request、status、continuation)
- **Capability 事件** (`fs/*`, `tools/*`, `telemetry/*`): 在接缝上附加策略与适配器,不导入 loop

#### Turn 流程简图

```
turn/start
  claim next-step input + 队列消息
  组装提示段落 + 工具模式; 投影运行时上下文
  -> agent/pre-step (reject | enter)
     step/start
     agent/request -> prepareCall
     协调系统/消息
     追加已进入消息为 user/message; 按需记录 request/header 和 request/context
     从日志派生并冻结模型历史
     流式传输 -> llm/stream -> agent/assistant-stream (start/chunk*/end)
       assistant/message | assistant/attempt
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     工具欠请求,或 next-step 输入到达 -> 下一 step
  -> agent/turn-stopping
turn/end
```

持久事件: `turn/*`, `step/*`, `system/message`, `user/message`, `assistant/message`, `assistant/attempt`, `tool/*`。

---

<a id="2-依赖与运行方式"></a>
## 2. 依赖与运行方式

### 2.1 核心依赖

```json
{
  "engines": {
    "node": "^22.19.0 || >=24.0.0"
  },
  "packageManager": "pnpm@11.7.0"
}
```

- **Node.js**: 要求 22.19+ 或 24+
- **pnpm**: 工作区管理器 11.7.0
- **TypeScript**: 6.0.3,严格模式 (`strict: true`, `noImplicitAny`)
- **vitest**: 4.1.8,测试运行器 + 覆盖率引擎
- **Cordis**: 厂商化依赖,每个 harness 包的 peerDependency

测试专用依赖:

- `@vitest/coverage-v8`: V8 覆盖率
- `jsdom`: 浏览器环境模拟
- `@testing-library/react`, `@testing-library/dom`: Web UI 测试
- `fast-check`: 属性测试
- `oxlint`: 快速 lint 工具

### 2.2 本地开发流程

```sh
# 1. 克隆与安装
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install

# 2. 构建(生成 lib/ 与捆绑运行时)
pnpm run build

# 3. 运行 Web UI
pnpm dsh web

# 4. 运行测试(选择性)
pnpm run test              # 单元测试
pnpm run test:coverage     # 覆盖率门禁
pnpm run test:e2e          # 真实 API e2e(需密钥)
pnpm run test:snapshot     # 快照回放
pnpm run test:web          # Web 浏览器快照

# 5. 其它质量门禁
pnpm run typecheck
pnpm run lint
pnpm run doc-sync
pnpm run hygiene
```

### 2.3 应用启动规则

**所有支持的 Node 应用通过 `dsh` CLI 以命名 profile 启动**。禁止项:

- 包 bin、demo、公共 SDK argv 逃逸
- 直接进程内插件挂载
- 调用者自行提供 Cordis 树

验证工具: [`scripts/verify-application-entrypoints.ts`](../scripts/verify-application-entrypoints.ts) 确保每个包 bin、可执行源、根 demo 都在明确类别中,拒绝绕过 `dsh` 的 Node 应用路径。

---

<a id="3-功能验证测试专章"></a>
## 3. 功能验证测试专章

### 3.1 测试分层概览

DeepSeek Harness 采用七层测试金字塔,每层职责明确,互不重叠:

| 层级 | 命令 | 职责 | 密钥要求 | CI 门禁 |
|---|---|---|---|---|
| **1. Unit** | `pnpm run test` | 包/示例的 `tests/**/*.spec.ts` 单元测试 | 否 | `node 24 / static` |
| **2. Coverage** | `pnpm run test:coverage` | 按文件 100% 覆盖 `packages/*/*/src` | 否 | `node 24 / coverage` |
| **3. E2E** | `pnpm run test:e2e` | 真实 API 调用,带密钥测试真实模型 | 是(自跳过) | `node 24 / e2e` |
| **4. Expected** | `pnpm run test:expected` | 无录制会话的 CLI/进程预期输出 | 否 | 各包自测 |
| **5. Snapshot** | `pnpm run test:snapshot` | 录制会话 JSONL 回放、录制、刷新 | 否(replay) | `node 24 / snapshot` |
| **6. Web** | `pnpm run test:web` | Chromium 浏览器 ARIA/geometry 快照 | 否 | `node 24 / web` |
| **7. Bench** | `pnpm run test:bench` | 性能基准,时间/堆/缩放预算 | 否 | `node 24 / benchmarks` |

### 3.2 目录地图

#### 3.2.1 测试支持包 (`packages/test-support/`)

```
packages/test-support/
├── session-snapshot/        # ★ 核心快照支持
│   ├── src/
│   │   ├── suite.ts        # defineAcpSnapshotSuite 等工厂
│   │   ├── launcher.ts     # 子进程/客户端启动与关闭
│   │   ├── harness.ts      # 脚本化场景驱动与会话日志收获
│   │   ├── manifest.ts     # 闭合 snapshot.yml 模式
│   │   ├── session-files.ts # 规范父子代际语法,选择最高角色
│   │   ├── identity.ts     # 跨父子日志的首见身份 token 化
│   │   ├── normalize.ts    # 纯归一化器与擦洗辅助函数
│   │   └── workspace.ts    # 场景 workspace 设置与完整预期状态比较
│   └── README.md
├── agent-loop-testkit/      # agent loop 测试工具
│   ├── src/
│   │   ├── index.ts        # mountAgentLoopTestDependencies, mountAgentLoopTestHarness
│   │   └── inbox.ts        # createInboxStub, unsupportedInbox
│   └── README.md
├── llm-replay/              # 无密钥 LLM 回放插件
│   ├── src/
│   │   ├── index.ts        # 回放插件主入口
│   │   ├── script.ts       # deriveReplayScript 从 JSONL 派生脚本
│   │   └── override.ts     # replay.override.json 模式
│   └── README.md
├── llm-mock-server/         # 脚本化 mock 服务器
├── loader-smoke/            # 模式感知子进程启动机制
├── client-runtime/          # 客户端运行时工具
└── remote-mock/             # 远程 mock
```

#### 3.2.2 快照场景树 (`snapshots/`)

```
snapshots/
├── AGENTS.md               # 快照所有权与归一化规则
├── session/                # Headless profile 场景
│   ├── text-turn/
│   ├── parallel-tool-calls/
│   ├── workspace-edit/
│   ├── subagent-multi/
│   ├── workflow-confinement/
│   └── ...                 # ~80+ 场景
├── sdk/                    # SDK 协议场景
│   ├── text-turn/
│   │   ├── session.jsonl       # v0 当前格式
│   │   ├── session.v1.jsonl    # v1 历史格式
│   │   ├── snapshot.yml        # 场景清单
│   │   ├── system-prompt.expected.md
│   │   ├── tool-schemas.expected.json
│   │   └── writer.expected.jsonl
│   ├── multi-turn/
│   ├── bash-tool/
│   ├── subagent-continuable/
│   └── sdk.snapshot.ts     # SDK 测试套件注册
├── acp/                    # ACP 协议场景
│   ├── handshake/
│   ├── escalation-approved/
│   ├── cancel-tool-calls/
│   └── acp.snapshot.ts
└── web/                    # 浏览器会话场景
    ├── live-interactions/
    ├── workspace-management/
    ├── subagent-conversation/
    └── ...
```

每个场景目录结构:

- `snapshot.yml`: 闭合清单(版本、场景名、profile、组合类、录制来源、环境、请求头类)
- `session[.vN].jsonl`: 规范父角色(v0 省略 `.v0`,正版本必须小写 `.vN`)
- `session.<ordinal>[.vN].jsonl`: 连续子角色
- `system-prompt.expected.md`: pin 拥有的生成系统提示词侧边文件
- `tool-schemas.expected.json`: pin 拥有的工具模式侧边文件
- `replay.override.json`: (可选)不可从持久化结算重建的故障模式(纯抛出、取消、挂起)
- `workspace/`: (可选)场景种子工作区
- `workspace.expected/`: (可选)变更场景提交的完整结果树

#### 3.2.3 包内测试 (`packages/*/*/tests/`)

每个包在自己的 `tests/` 下保存:

- `*.spec.ts`: 单元测试
- `*.e2e.ts`: 真实 API e2e 测试(自跳过,除非有密钥)
- `*.expected.e2e.ts`: 预期输出(构建产物下运行)
- `expected/`: 预期输出基准
- `harness.ts`: (可选)共享测试工具

示例: `packages/llm/llm-deepseek/tests/adapter.e2e.ts` 包含真实 DeepSeek API 的端到端测试,覆盖 V4 Flash 跨思考模式、工具调用往返、推理回传等。

### 3.3 关键 API 与工具

#### 3.3.1 快照套件工厂

**`defineAcpSnapshotSuite(options)`**

位置: `@deepseek-ai/dsh-session-snapshot`

注册一个 ACP 快照套件,将场景表转换为 record/replay/refresh vitest 测试。

```typescript
import { defineAcpSnapshotSuite, type Scenario } from '@deepseek-ai/dsh-session-snapshot'

const SCENARIOS: Scenario[] = [
  { name: 'text-turn', hasModelTurn: true, recorded: true, pinsHeader: true },
  { name: 'multi-turn', hasModelTurn: true, recorded: true },
]

defineAcpSnapshotSuite({
  agent: {
    binScript: '/absolute/path/to/apps/cli/src/bin.ts',
    configPath: '/absolute/path/to/cordis.yml',
    profile: 'acp',
    tsconfigPath: '/absolute/path/to/tsconfig.json',
  },
  snapshotsDir: '/absolute/path/to/snapshots/acp',
  scenarios: SCENARIOS,
  mode: 'replay', // 'record' | 'refresh'
})
```

**参数:**

- `agent`: 绝对路径集合(`binScript`, `libBinScript`可选, `configPath`, `profile`, `tsconfigPath`)
- `snapshotsDir`: 包含场景目录的根
- `scenarios`: `Scenario[]`,每个声明 `name`, `hasModelTurn`, `recorded`, `pinsHeader` 等
- `mode`: `'replay'` | `'record'` | `'refresh'`(通常从 `DSH_SNAPSHOT` 环境变量解析)

**等价工厂:**

- `defineHeadlessSnapshotSuite(options)`: Headless profile 场景
- `defineSdkSnapshotSuite(options)`: SDK 协议场景
- `defineWebSnapshotSuite(options)`: Web 浏览器场景(需构建前端 dist)

#### 3.3.2 Agent Loop 测试工具

**`mountAgentLoopTestHarness(ctx)`**

位置: `@deepseek-ai/dsh-agent-loop-testkit`

为 AgentLoop 测试挂载标准前提与生产 loop 驱动,暴露 Inbox 输入认领。

```typescript
import { Context } from '@deepseek-ai/cordis'
import { mountAgentLoopTestDependencies, mountAgentLoopTestHarness } from '@deepseek-ai/dsh-agent-loop-testkit'
import { SessionId } from '@deepseek-ai/dsh-session'

const ctx = new Context()
await mountAgentLoopTestDependencies(ctx) // LLM、session、system-prompt、tools、agents
// 在此注册测试适配器与任意负载顺序敏感插件
const harness = await mountAgentLoopTestHarness(ctx)
const agent = await harness.create(SessionId('test-agent'))

// 操作 Inbox
agent.inbox.append('next-turn', message)
const admitted = harness.claim(agent, 'next-turn', 1)
```

**结构化 Inbox 桩:**

```typescript
import { createInboxStub, unsupportedInbox } from '@deepseek-ai/dsh-agent-loop-testkit'

const agent = { inbox: createInboxStub() }        // 可变待定列表
const agent2 = { inbox: unsupportedInbox() }      // 所有变更抛出
```

#### 3.3.3 桥接测试工具

**`makeBridgeHarness(options)`**

位置: `packages/acp/acp/tests/harness.ts`

创建内存 ACP 传输夹具,在真实 agent 工厂与 loop 上构建,仅 mock 模型适配器。

```typescript
import { makeBridgeHarness, textResponse } from './harness'

const harness = await makeBridgeHarness({
  script: [
    textResponse('Hello'),
    toolCallResponse(),
    textResponse('Done'),
  ],
})

const { agent, client, ctx } = harness
// client: ACP JSON-RPC 客户端
// agent: 活跃的 Agent 实例
// ctx: Cordis Context

// 驱动场景
await client.request(methods.session.new, { ... })
```

**`MockAdapter`**: 脚本化适配器,按 `StreamChunk[]` 或 `'hang'` 逐请求消耗脚本,暴露 `requests` 数组供断言。

#### 3.3.4 LLM 回放插件

**`@deepseek-ai/dsh-llm-replay`**

挂载:

```yaml
- id: llm-replay
  name: '@deepseek-ai/dsh-llm-replay'
  config:
    file: $DSH_SNAPSHOT_FILE            # session.jsonl 路径
    overrideFile: $DSH_SNAPSHOT_OVERRIDE # replay.override.json
    childFiles: $DSH_SNAPSHOT_CHILD_FILES
    providers:                          # (可选)回放专用目录
      - id: deepseek-official
        name: DeepSeek
        models:
          - id: deepseek-v4-flash
            contextWindow: 128000
            systemPromptUpdate: in-history
```

**工作原理:**

1. 从选定的 `session[.vN].jsonl` 投影恢复事件
2. `deriveReplayScript` 按日志顺序展开每个 `assistant/message`/`assistant/attempt` 流
3. 第一个活跃 Session 调用认领父脚本,下一个新 Session 认领下一子脚本
4. 脚本字符串可嵌入 `{{fromRequest:<regex>}}`,流时解析活跃请求的字符串叶

**覆盖侧边文件** (`replay.override.json`):

```json
{
  "patches": [
    { "at": 0, "entry": { "throw": { "error": "pre-2xx failure" } } },
    { "at": 2, "entry": { "hang": { "readyFile": "/tmp/ready" } } }
  ]
}
```

- `throw` entry: 纯抛出(无分块时默认 pre-2xx 未接受,可设 `accepted: true`)
- `hang` entry: 等待取消,可命名 `readyFile` 实现确定性取消

### 3.4 环境变量与模式控制

| 变量 | 用途 | 值 |
|---|---|---|
| `DSH_SNAPSHOT` | 快照模式 | `replay`(默认) \| `record` \| `refresh` |
| `DSH_SNAPSHOT_FILE` | 主 fixture 路径 | 绝对路径到 `session[.vN].jsonl` |
| `DSH_SNAPSHOT_OVERRIDE` | 覆盖侧边文件 | 绝对路径到 `replay.override.json` |
| `DSH_SNAPSHOT_CHILD_FILES` | 子会话日志 | 逗号分隔的绝对路径 |
| `DSH_SNAPSHOT_MAX_CONCURRENCY` | 快照并发数 | 正整数(默认 5,上限 `os.availableParallelism()`) |
| `DSH_EXAMPLE_MODE` | 示例模式 | `src`(默认,零构建) \| `lib`(需先构建) |
| `DSH_E2E_MAX_WORKERS` | E2E 并发池 | 正整数(默认 4) |
| `DSH_GATE_CONCURRENCY` | 门禁并发数 | 正整数(CI: `'8'`) |
| `DSH_GATE_FAIL_FAST` | 首个阻塞故障后快速失败 | `'1'`(启用) |
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥 | 从环境或根 `.env` 读取 |
| `DEEPSEEK_BASE_URL` | (可选)自定义端点 | 覆盖公共端点 |

**模式语义:**

- **replay**: 无密钥默认,启动真实子进程路径,从录制模型响应回放,diff 组装请求、归一化协议/文本输出、持久化日志预期输出
- **record**: 调用真实 API,更新 fixture 与预期输出(审查所有 diff)
- **refresh**: 回放已提交脚本(无密钥),更新当前预期输出(审查所有 diff)

### 3.5 代表性场景深入剖析

#### 3.5.1 SDK `text-turn` 场景

**位置:** `snapshots/sdk/text-turn/`

**文件清单:**

```
text-turn/
├── snapshot.yml
├── session.jsonl              # v0 当前格式
├── session.v1.jsonl           # v1 历史格式(保留迁移覆盖)
├── system-prompt.expected.md
├── tool-schemas.expected.json
├── writer.expected.jsonl      # 原生当前格式写出器预期
├── cordis.yml                 # 场景专用组合
├── cordis.snapshot.yml        # 回放专用组合
└── model.cordis.yml           # 模型配置
```

**`snapshot.yml` 内容:**

```yaml
version: 1
scenario: text-turn
profile: sdk
composition: sdk-upload
recording: live
environment:
  DSH_SNAPSHOT_FEEDBACK: '1'
header:
  class: sdk-upload
  pin: true
```

**测试覆盖:**

- SDK JSON-RPC 协议: `session/new`, `session/sendMessage`
- 持久控制: 创建 Session → 发送消息 → 等待 turn 结束
- 通知流: 跟踪 `notification/agent/created`, `notification/agent/status`, `notification/stream/*`, `notification/turn/end`
- 系统提示词与工具模式 pin: 作为 `{{system}}` 和 `{{tools}}` token 化,侧边文件保留完整文本
- 当前与历史格式: `session.jsonl` 是当前回放代际,`session.v1.jsonl` 保留 v1 迁移覆盖,`writer.expected.jsonl` 是原生当前格式输出预言

**如何跑:**

```sh
# 回放(无密钥)
pnpm run test:snapshot -- -t text-turn

# 录制(需密钥,审查 diff)
DSH_SNAPSHOT=record pnpm run test:snapshot -- -t text-turn

# 刷新(无密钥,审查 diff)
DSH_SNAPSHOT=refresh pnpm run test:snapshot -- -t text-turn
```

#### 3.5.2 ACP `handshake` 场景

**位置:** `snapshots/acp/handshake/`

**测试覆盖:**

- ACP 协议启动握手: `initialize` RPC
- 能力协商: 服务器能力、客户端能力
- 无模型轮次: 纯协议,不调用 LLM
- Stdio 纯度: 断言标准输出无非协议文本污染

#### 3.5.3 Web `live-interactions` 场景

**位置:** `snapshots/web/live-interactions/`

**测试覆盖:**

- Chromium 浏览器驱动真实 Web UI
- ARIA 角色与可访问性快照
- 用户交互: 点击、输入、滚动
- 流式更新: 渐进式 Assistant 消息分块
- Geometry 快照: 布局与视觉回归

**需求:** `pnpm run build` 先构建前端 dist(CSS、JS bundle)。

### 3.6 如何添加新测试场景

#### 逐步指南

**场景 1: 为现有 profile 添加新快照场景**

假设为 SDK 添加 `image-upload` 场景:

1. **创建场景目录:**

   ```sh
   mkdir -p snapshots/sdk/image-upload
   cd snapshots/sdk/image-upload
   ```

2. **编写 `snapshot.yml` 清单:**

   ```yaml
   version: 1
   scenario: image-upload
   profile: sdk
   composition: sdk-upload
   recording: live
   header:
     class: sdk-upload
     pin: true
   ```

3. **准备场景专用组合** (如需):

   ```yaml
   # cordis.yml
   - id: test-image-tool
     name: '@deepseek-ai/dsh-tool-test-image'
   ```

4. **更新场景表:**

   在 `snapshots/sdk/sdk.snapshot.ts` 中添加:

   ```typescript
   const SCENARIOS: Scenario[] = [
     // ... 现有场景
     { name: 'image-upload', hasModelTurn: true, recorded: true, pinsHeader: false },
   ]
   ```

5. **录制场景** (需 `DEEPSEEK_API_KEY`):

   ```sh
   DSH_SNAPSHOT=record pnpm run test:snapshot:record -- -t image-upload
   ```

   产物:
   - `session.jsonl` (或 `session.vN.jsonl`)
   - `system-prompt.expected.md` (若 `pinsHeader: true`)
   - `tool-schemas.expected.json` (若 `pinsHeader: true`)

6. **审查 diff 并提交:**

   ```sh
   git add snapshots/sdk/image-upload/
   git commit -m "test(snapshot): add SDK image-upload scenario"
   ```

7. **验证回放:**

   ```sh
   pnpm run test:snapshot -- -t image-upload
   ```

**场景 2: 添加包级 e2e 测试**

假设为 `packages/llm/llm-pi-ai` 添加真实 API 测试:

1. **创建 `tests/adapter.e2e.ts`:**

   ```typescript
   import { describe, it, expect, beforeEach, afterEach } from 'vitest'
   import { Context } from '@deepseek-ai/cordis'
   import LlmRuntime from '@deepseek-ai/dsh-llm'
   import * as LlmPiAi from '../src/index.ts'

   const contexts: Context[] = []

   beforeEach(async () => {
     const ctx = new Context()
     contexts.push(ctx)
     await ctx.plugin(LlmRuntime)
     await ctx.plugin(LlmPiAi, {
       protocol: 'openai-compatible',
       baseURL: 'https://api.pi-ai.ai/v1',
     })
     return ctx
   })

   afterEach(async () => {
     await Promise.all(contexts.splice(0).map(ctx => ctx.fiber.dispose()))
   })

   describe.skipIf(!process.env.PI_AI_API_KEY)('llm-pi-ai e2e (real API)', () => {
     it('completes a simple prompt', async () => {
       const ctx = contexts[0]!
       const stream = ctx.llm.stream({
         provider: 'pi-ai',
         model: 'pi-model',
         messages: [{ role: 'user', content: 'Say pong' }],
       })
       const chunks = []
       for await (const chunk of stream) {
         chunks.push(chunk)
       }
       expect(chunks.at(-1)?.type).toBe('finish')
       const text = chunks
         .filter(c => c.type === 'text-delta')
         .map(c => c.text)
         .join('')
       expect(text.toLowerCase()).toContain('pong')
     })
   })
   ```

2. **验证自跳过:**

   ```sh
   # 无密钥: 测试自动跳过,套件绿色
   pnpm run test:e2e -- packages/llm/llm-pi-ai

   # 有密钥: 测试执行
   PI_AI_API_KEY=sk-... pnpm run test:e2e -- packages/llm/llm-pi-ai
   ```

3. **提交:**

   ```sh
   git add packages/llm/llm-pi-ai/tests/adapter.e2e.ts
   git commit -m "test(llm-pi-ai): add real API e2e smoke"
   ```

**场景 3: 为覆盖率门禁添加单元测试**

覆盖率门禁要求 `packages/*/*/src` 按文件 100% 行覆盖。若新增源文件 `packages/tool-example/tool-example/src/execute.ts`:

1. **创建 `tests/execute.spec.ts`:**

   ```typescript
   import { describe, it, expect } from 'vitest'
   import { execute } from '../src/execute.ts'

   describe('execute', () => {
     it('returns expected result for valid input', () => {
       expect(execute({ input: 'test' })).toBe('output')
     })

     it('throws on invalid input', () => {
       expect(() => execute({ input: '' })).toThrow('Input is empty')
     })
   })
   ```

2. **验证覆盖率:**

   ```sh
   pnpm run test:coverage -- packages/tool-example
   ```

   若行覆盖率 < 100%,报告未覆盖行:

   ```
   ERROR: packages/tool-example/tool-example/src/execute.ts missed 100% gate
   UNCOVERED: execute.ts:15:3 (branch not taken)
   UNCOVERED: execute.ts:23:5 (line never executed)
   ```

3. **补全覆盖或删除死代码:**

   - **情况 A**: 未覆盖行是活跃逻辑 → 添加测试
   - **情况 B**: 未覆盖行是死代码 → 删除行,更新测试

4. **重新验证:**

   ```sh
   pnpm run test:coverage -- packages/tool-example
   # ✓ All files passed 100% per-file gate
   ```

### 3.7 与生产代码的耦合点

#### 3.7.1 Cordis Loader

**位置:** `vendor/loader/src/index.ts`

**职责:** 加载插件模块,解包导出,构建 fiber 树。

**测试耦合:**

- `Loader.unwrapExports(exports)`: 优先 `.default ?? exports`
  - **陷阱:** `export default apply` + `export const inject = [...]` → Loader 丢弃命名空间,取裸函数,无 `inject` → 服务解析失败
  - **防御:** `verify-application-entrypoints.ts` + 真实 Loader e2e + 显式 `expect('default' in mod).toBe(false)` 守卫

#### 3.7.2 Profile 与 Bundle

**Bundle 定义:** `package.json` 中的 `dsh` 字段

```json
{
  "dsh": {
    "bundle": "./cordis.yml"   // 或
    "profile": ["dsh-base", "dsh-web-app"]
  }
}
```

**测试耦合:**

- Profile 级集成测试位于 `apps/cli/tests/profiles/`
- 每个交付 profile 必须有真实 Loader 冒烟测试(启动 → 发送提示词 → 检查外界)
- 快照场景通过真实 `dsh` CLI 启动,exercises 完整 bundle 栈

#### 3.7.3 Agent Loop

**位置:** `packages/core/agent-loop/src/index.ts`

**职责:** 实现 `Agent` 接口,驱动 turn/step 循环。

**测试耦合:**

- `agent/pre-step` 瀑布: 测试可重写或拒绝认领消息
- `agent/request` 瀑布: 测试可在 `prepareCall()` 前取消
- `agent/turn-stopping` 串行: 测试可停止 turn
- **错误恢复测试:** 按步骤分离 pre-chunk/post-chunk 故障,证明失败分块不派生消息或工具副作用
- **取消测试:** `replay.override.json` 的 `hang` entry 实现确定性取消

#### 3.7.4 Session JSONL 格式

**位置:** `packages/session/session-persistence-jsonl/src/`

**格式版本:**

- **v0**: `session.jsonl[.zstd]`,可能使用规范 packed rows
- **v1+**: `session.vN.jsonl[.zstd]`,每事件一行,小写 `.vN`

**测试耦合:**

- **格式目录:** `packages/session/session-format-catalog/src/` 静态目录 + 版本规范
- **迁移链:** 相邻迁移包 (`session-format-v0-to-v1`, `session-format-v1-to-v2`, ...)
- **快照 fixture:** 选择最高代际,header 同意 filename,replay 在内存迁移历史输入
- **归一化:** `normalizeSessionSnapshot` 省略顶层 `seq`/`time` envelope,保留完整 header 与事件 payload,归一化路径、擦洗系统提示词文本与工具模式
- **格式语料库测试:** `packages/test-support/llm-replay/tests/session-format-corpus.spec.ts` 通过真实目录恢复 `snapshots/`, `packages/`, `scripts/snapshots/python-sdk-single-exe/` 下的每个版本化 `session*.jsonl`,无改源字节,pin 故意历史拒绝

---

<a id="4-从事故中学习"></a>
## 4. 从事故中学习

### 4.1 Postmortem 0001: ACP default export 丢失 inject

**位置:** `docs/postmortem/0001-acp-default-export-drops-inject.md`

**执行摘要:**

两个集成错误在 100% 单元覆盖率下破坏 ACP:

1. `export default apply` 导致 Loader 丢弃 `inject`
2. 追踪可选服务查找在 shadow 边界失败

手动挂载测试绕过两条路径。

**根因 #1: `export default apply` 丢弃插件的 `inject`**

`packages/acp/acp/src/index.ts` 是命名空间插件:

```typescript
export const name = 'acp'
export const inject = ['agents', 'sessions', 'sessionPersistence']
export function apply(ctx: Context, config: AcpConfig): void { /* ... */ }
export default apply   // ← 陷阱
```

Cordis Loader `unwrapExports`:

```typescript
unwrapExports(exports: any) {
  if (isNullable(exports)) return exports
  exports = exports.default ?? exports   // ← 优先 .default
  if (!exports.__esModule) return exports
  return exports.default ?? exports
}
```

有 default export 时,`exports.default ?? exports` 解析为**裸 `apply` 函数**,无 `inject`/`name`/`Config` 属性,Loader 丢弃命名空间。`apply` 运行在**无注入服务**的 fiber 中,第一行 `const agents = ctx.agents` 遍历 fiber 树(ROOT → Include → Loader → ROOT),在无 fiber 的存储中找到 `agents`,到达根 fiber (`runtime === null`),抛出 `cannot get property "agents" without inject`。

**修复:** 删除 `export default apply`。

**根因 #2: 可选服务读取通过 traceable shadow 触发 inject 守卫**

`AgentLoop.resume()` 读取 `this.ctx.sessionPersistence`,但 `static inject` 故意不包含它(注入会让非持久化 demo 永远 pend)。

服务访问通过上下文代理。当通过外部 fiber 获得的 traceable 代理调用服务方法时(这里:桥接 fiber 调用 `ctx.agents.resume`,注册表返回 `this.factory` 重新包装为绑定到调用者的 traceable 代理),`createShadowMethod` 将 `this` 重新绑定到 shadow 对象,其 `ctx` 携带 `[symbols.shadow]` 指向 `AgentLoop` 自身的构建上下文。在 `resume` 内部,`this.ctx.sessionPersistence` 用代理处理程序从 shadow 的 fiber 开始 fiber 遍历:

```typescript
let fiber = (ctx[symbols.shadow] as Context ?? ctx).fiber
while (true) {
  const impl = fiber.store?.[prop]
  if (impl) return getTraceable(ctx, impl.value)
  if (prop in fiber.inject) { /* inactive-context error */ }
  if (!fiber.runtime) throw error   // ← 到达根,抛出
  if (fiber.parent[symbols.isolate][prop] !== key) throw error
  fiber = fiber.parent.fiber        // ← 仅祖先
}
```

遍历是**仅祖先**的。`sessionPersistence` 既不在 `AgentLoop` 的 fiber 存储中(不在其 `static inject` 中),也不在去往根的任何祖先中(它活跃在**兄弟**分支上),遍历到达根 fiber 并抛出。

内存 `AgentLoop` resume 测试没有捕获这个,因为它们**从测试代码**直接调用 `ctx.agents.resume(...)`——在任何插件 fiber 之外。那里,`ctx.fiber.runtime` 是 `null`,代理处理程序采取早期绕过:

```typescript
if (!ctx.fiber.runtime) return ctx.reflect.get(prop, false)  // ← 直接全局存储查找,无 fiber 遍历
```

`ctx.reflect.get(name, false)` 是全局服务存储中按 isolate 符号键的直接查找,完全忽略 fiber 拓扑并找到服务。所以从顶层测试读取工作;从真实插件 fiber 内部,通过 shadow 到达,抛出。

**修复:** 用 `ctx.get('sessionPersistence')` 读取可选服务,它使用全局 isolate 键存储同时保留活跃状态检查。

**为何每个测试都漏掉它(真正的失败):**

1. **内存 harness 手动挂载:** `ctx.plugin({ name, inject, apply })` 手动提供 `inject`,无法重现 Bug #1,`unwrapExports` 仅被 Loader 调用
2. **扁平根上下文:** 同一 harness 在一个根上下文挂载所有,`AgentLoop` resume 从那里到达要么顶层运行(`!runtime` 绕过),要么通过原点仍解析在根上的 shadow,掩盖 Bug #2 的祖先遍历失败
3. **无密钥 e2e 仅测试 `initialize`:** 不触达工厂,飘过两个 bug
4. **密钥门控的唯一 `session/new`/`session/load` 测试:** CI(无密钥)跳过,本地「通过」因陈旧构建 `lib/` 碰巧满足模块解析

**护栏添加:**

1. **移除 `export default apply`** (`packages/acp/acp/src/index.ts`)
2. **`AgentLoop.resume` 读取 `this.ctx.get('sessionPersistence')`** (`packages/core/agent-loop/src/index.ts`),附注释解释 shadow 遍历陷阱
3. **无密钥 `session/new` e2e,真实 stdio** (`apps/cli/tests/profiles/acp/tests/acp.e2e.ts`): 通过真实 Loader 作为子进程启动 profile,断言 `session/new` 解析。在 Bug #1 下失败,无需 API 密钥。验证恢复 `export default apply` 时失败
4. **子进程 spawn 中的 `TSX_TSCONFIG_PATH`:** 子进程从临时 cwd 运行,tsx 无法通过向上搜索找到 repo-root tsconfig `paths` 映射,dsh-* 导入静默回退到构建 `lib/`。将 tsx 指向 repo tsconfig 使解析 cwd 独立,确保测试运行**源码**,而非可能陈旧的构建
5. **[docs/testing.md](testing.md) 规则:** "测试真实入口路径",行覆盖率不是行为覆盖——为每个未来插件成文课训

**教训:**

- 命名空间插件与 default export 在 cordis Loader 下互斥。选择命名空间形式(`name`/`inject`/`Config`/`apply`),不要添加 `export default`,`unwrapExports` 会丢弃命名空间
- 对于插件机会读取但不在 `static inject` 中声明的服务,使用 `ctx.get(name)`,而非 `ctx.<name>`。属性代理通过仅祖先 fiber 遍历解析,通过外部 shadow 失败;`ctx.get(name)` 是拓扑独立查找(默认严格——不活跃后端读为 `undefined`,而非在 teardown 中期交还)
- 手动构建插件的测试无法验证插件如何加载。至少一个测试必须端到端驱动真实 Loader/export 路径。当头条操作不调用模型时,那个测试不需要 API 密钥——所以它属于 CI,不在密钥门禁后
- 信任 trace,不信理论。优雅 shadow 解释是真的,但是**第二个** bug;**第一个**是一行导出错误,fiber 遍历 `console.error` 在数小时合理但错误推理后数分钟找到

### 4.2 测试体系如何体现教训

**真实入口路径测试:**

- **构建产物冒烟:** `packages/examples/*/tests/built-bin.e2e.ts` 从构建后 `lib/bin.js` 运行包 bin,暴露 tsx 掩盖的失败(结算竞态、模块解析、吞掉的加载失败)
- **Profile-level 集成:** `apps/cli/tests/profiles/*/tests/*.e2e.ts` 通过真实 Loader 启动交付 profile,驱动公共协议或浏览器接口
- **单例作用域测试:** `packages/sdk/server/tests/built-scope-carrier.e2e.ts` 验证多 bundle 共享单例模块的构建产物

**守卫回归:**

- `expect('default' in mod).toBe(false)` + `unwrapExports` 往返断言
- **证明守卫:** 引入回归 → 观察红 → 回退,在同一 commit/PR 中
- **无单元,真实组合:** 通过 Loader 启动测试专用 `cordis.yml`,只 mock 外部服务或非确定性输入

**可选服务查找:**

- `ctx.get('name')` 拓扑独立,严格活跃检查
- `ctx.<name>` 保留给声明注入的服务,属性代理拓扑敏感

---

<a id="5-设计取舍与风险"></a>
## 5. 设计取舍与风险

### 5.1 设计取舍

#### 5.1.1 100% 覆盖率门禁 vs 实用主义

**选择:** 每文件 100% 行覆盖 `packages/*/*/src`,例外极少且显式(pwsh-only、Windows-only、subprocess-only 入口)。

**权衡:**

- **赢得:** 未覆盖行是死代码的强信号,门禁标记删除候选
- **付出:** 偶尔需要微不足道的边缘 case 测试(空输入、错误 case)只为满足覆盖
- **风险:** 行覆盖率 ≠ 行为覆盖率,「绿色覆盖率,破损产品」仍可能(postmortem 0001)
- **缓解:** 强制 profile-level 真实入口路径测试 + 快照套件

#### 5.1.2 真实 API e2e vs 成本

**选择:** 倡导带密钥真实 API 测试,自跳过让无密钥 CI 绿色。

**权衡:**

- **赢得:** 只有带密钥运行证明 agent 对接真实模型工作;捕获适配器/协议回归
- **付出:** 每跑消耗 API 配额(缓解:DeepSeek 推理便宜)
- **风险:** 共享内部密钥碰撞并发配额 → CI 偶发限流 → 重试(配置:最多 2 次重试,慷慨超时)
- **缓解:** `DSH_E2E_MAX_WORKERS` 环境旋钮(默认 4,可降至 1 恢复串行)

#### 5.1.3 录制会话快照 vs 纯脚本

**选择:** 快照使用录制 Session JSONL 作 fixture,派生回放脚本,而非手写脚本。

**权衡:**

- **赢得:** Fixture 是真实 agent 一次运行的投影,保证可行性;回放与 record 使用同一产物
- **付出:** 无法从持久化结算重建的故障模式(纯抛出、取消、挂起)需 `replay.override.json` 侧边文件
- **风险:** 历史 fixture 格式迁移复杂性 → 相邻迁移链(v0→v1→v2→…),format 目录版本规范
- **缓解:** 回放内存迁移历史输入,原生当前格式写出器预期(`writer.expected.jsonl`) 独立比较,格式语料库测试防止迁移回归

#### 5.1.4 归一化与 token 化 vs 原始 fixture

**选择:** Fixture 省略 `seq`/`time` envelope,归一化路径,擦洗系统提示词与工具模式为 `{{system}}` / `{{tools}}` token。

**权衡:**

- **赢得:** 跨运行结构比较,无波动 timestamp/id/cwd
- **付出:** 记录与预期 fixture 不是原始持久化日志的逐字节副本
- **风险:** 归一化器 bug 可能掩盖真实回归
- **缓解:** `normalizeSessionSnapshot` 纯函数 + 单元测试,回放合成 envelope,比较编码保留接受目录输出

### 5.2 风险

#### 5.2.1 测试执行并发风险

**风险:** forked worker 同时运行多个 spec,共享宿主机、端口、路径、子进程。不拥有资源到 teardown 的测试可能碰撞。

**缓解:**

- [dsh-ci-test-reliability](../.agents/skills/dsh-ci-test-reliability/SKILL.md) 资源分配、状态恢复、同步、teardown 规则
- 每个 spec 在 `afterEach` 中 dispose owning context,即使失败/重试/超时
- 随机端口(`createServer({ port: 0 })`),唯一临时目录(`mkdtemp`)
- 「只有单独运行时才通过」读作 spec 缺陷,非 runner 不稳定

#### 5.2.2 格式迁移复杂性

**风险:** Session JSONL 格式版本增加 → 相邻迁移链 → 回放内存迁移历史输入 → 迁移 bug 破坏回放。

**缓解:**

- 相邻迁移包拥有恰好一个 `vN -> vN+1` 步,经单元测试,独立合成
- 历史 fixture 明确声明 `sessionFormat.version` + 闭合 `coverage` 名,record/refresh 不改写
- 格式语料库测试恢复 `snapshots/`, `packages/`, `scripts/snapshots/python-sdk-single-exe/` 下每个版本化 `session*.jsonl`,pin 故意历史拒绝,任何拒绝消失或改变失败

#### 5.2.3 Mock 不足风险

**风险:** 过度 mock → 「绿色单元测试,破损产品」(postmortem 0001)。

**缓解:**

- 只 mock 开销高或非确定性边界(LLM 适配器、网络、时钟)
- 保持下游一切真实:`makeBridgeHarness` 挂载真实 loop、session 存储、工具注册表、JSONL 持久化,唯一 mock 是 `MockAdapter`
- 强制 profile-level 真实 Loader 冒烟测试,无密钥可行时

#### 5.2.4 文档漂移风险

**风险:** 文档脱离代码 → 误导开发者。

**缓解:**

- [docs/testing.md](testing.md) & [docs/AGENTS.md](AGENTS.md) 承载测试策略与规则,与代码在同一 PR 更新
- `verify-doc-budgets` 守卫文档预算,防止无限扩张
- `doc-typecheck` 验证 fenced `ts` 块编译
- `verify-type-equiv` 捕获类型声明漂移
- 生成参考(config-catalog、tool-catalog、persistence-catalog) 从源重新生成,新鲜度门控

### 5.3 文档缺口

本调研完成后,以下领域可受益于专门文档:

1. **测试隔离诊断手册**: `dsh-ci-test-reliability` 涵盖规则,但缺乏「如何诊断偶发失败」操作手册
2. **性能基准指南**: `benchmarks/` 规则在 [Agent Note](../.agents/notes/implemented/testing/2026-09-04-session-open-performance-gate.md),但无「如何添加新基准」指南
3. **Web 浏览器快照深入剖析**: `test:web` 使用 Chromium + ARIA,但缺乏「如何调试视觉回归」指南
4. **Python SDK 测试映射**: `scripts/snapshots/python-sdk-single-exe/` 快照覆盖 Python 投影,但无 Python 运行时 CI 完整架构文档

---

<a id="6-快速上手指南"></a>
## 6. 快速上手指南

### 6.1 首次克隆后

```sh
# 1. 克隆
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness

# 2. 安装(pnpm 工作区)
pnpm install

# 3. 构建(生成 lib/ + 捆绑运行时)
pnpm run build

# 4. 验证安装
pnpm run test -- packages/util  # 运行一个小包的单元测试
```

### 6.2 先读哪些文件

**理解仓库:**

1. **[AGENTS.md](../AGENTS.md)**: 仓库规约,命令,插件规则
2. **[docs/architecture.md](architecture.md)**: Cordis、profile/bundle、核心包、事件域
3. **[docs/testing.md](testing.md)**: 测试分层、策略、when-to-test 规则

**深入测试:**

4. **[packages/test-support/session-snapshot/README.md](../packages/test-support/session-snapshot/README.md)**: 快照支持核心
5. **[packages/test-support/agent-loop-testkit/README.md](../packages/test-support/agent-loop-testkit/README.md)**: Agent loop 测试工具
6. **[snapshots/AGENTS.md](../snapshots/AGENTS.md)**: 快照所有权与归一化规则
7. **[docs/postmortem/0001-acp-default-export-drops-inject.md](postmortem/0001-acp-default-export-drops-inject.md)**: 事故案例研究

**代码:**

8. **[`packages/acp/acp/tests/harness.ts`](../packages/acp/acp/tests/harness.ts)**: `makeBridgeHarness` 参考实现
9. **[`snapshots/sdk/sdk.snapshot.ts`](../snapshots/sdk/sdk.snapshot.ts)**: `defineSdkSnapshotSuite` 使用示例
10. **[`packages/llm/llm-deepseek/tests/adapter.e2e.ts`](../packages/llm/llm-deepseek/tests/adapter.e2e.ts)**: 真实 API e2e 模式

### 6.3 先跑哪条命令

**健康检查(无密钥):**

```sh
pnpm run test:coverage -- packages/util/brand  # 小包覆盖率
pnpm run test:snapshot -- -t sdk/text-turn    # 回放一个快照
pnpm run lint                                  # Lint
pnpm run typecheck                             # TypeScript
```

**添加新测试后:**

```sh
# 包单元测试
pnpm run test -- packages/your-pkg

# 覆盖率门禁
pnpm run test:coverage -- packages/your-pkg

# E2E(需密钥)
YOUR_API_KEY=... pnpm run test:e2e -- packages/your-pkg

# 快照回放
pnpm run test:snapshot -- -t your-scenario

# 快照录制(需密钥)
DEEPSEEK_API_KEY=... DSH_SNAPSHOT=record pnpm run test:snapshot:record -- -t your-scenario

# 快照刷新(无密钥)
DSH_SNAPSHOT=refresh pnpm run test:snapshot:refresh -- -t your-scenario
```

**CI 级别检查(推送前):**

使用 [dsh-pre-push-checks](../.agents/skills/dsh-pre-push-checks/SKILL.md) 技能选择最小测试与检查覆盖 outgoing diff,无需反射性运行完整仓库套件。

```sh
# 示例:仅改动 packages/tool-bash
pnpm run test:coverage -- packages/tool-bash
pnpm run lint
pnpm run typecheck

# 示例:改动核心 agent-loop
pnpm run test:coverage -- packages/core/agent-loop
pnpm run test:snapshot  # agent-loop 改动影响会话输出
pnpm run lint
pnpm run typecheck
```

### 6.4 添加测试的决策树

```
┌─ 改动是什么?
│
├─ 新包 / 新源文件
│  └─> 添加单元测试(tests/*.spec.ts)
│     └─> 验证覆盖率: pnpm run test:coverage -- packages/your-pkg
│
├─ 新 LLM 适配器 / 新提供方
│  ├─> 添加单元测试(协议编解码、错误处理)
│  └─> 添加真实 API e2e(tests/*.e2e.ts,自跳过)
│
├─ 新模型面向工具
│  ├─> 添加单元测试(工具逻辑、模式)
│  └─> 添加或更新快照场景(工具实际使用)
│
├─ 新 agent-loop 行为 / 新事件
│  ├─> 添加单元测试(事件发射、listener 顺序)
│  ├─> 添加 agent-loop-testkit harness 测试(Inbox、projection)
│  └─> 添加或更新快照场景(持久化结算、协议输出)
│
├─ 新 profile / 新 bundle
│  ├─> 添加 profile-level 冒烟 e2e(apps/cli/tests/profiles/)
│  └─> 添加快照场景(完整 profile 栈,真实 Loader)
│
├─ 改动 Session JSONL 格式
│  ├─> 添加相邻迁移包(vN -> vN+1)
│  ├─> 更新 session-format-catalog
│  ├─> 添加迁移测试(往返)
│  └─> 更新受影响快照(record → refresh → 审查 diff)
│
├─ 新 Web UI 特性
│  ├─> 添加单元测试(组件行为)
│  └─> 添加或更新 Web 浏览器快照(tests/expected/ 或 snapshots/web/)
│
└─ 性能关键路径改动
   └─> 添加或更新性能基准(benchmarks/)
```

---

## 附录 A: 关键路径速查

### A.1 测试命令速查表

| 目标 | 命令 | 门禁 |
|---|---|---|
| 运行所有单元测试 | `pnpm run test` | — |
| 运行一个包的测试 | `pnpm run test -- packages/core/agent` | — |
| 按文件 100% 覆盖率 | `pnpm run test:coverage` | `node 24 / coverage` |
| 真实 API e2e | `pnpm run test:e2e` | `node 24 / e2e` |
| 预期输出 | `pnpm run test:expected` | — |
| 快照回放 | `pnpm run test:snapshot` | `node 24 / snapshot` |
| 快照录制 | `DSH_SNAPSHOT=record pnpm run test:snapshot:record` | — |
| 快照刷新 | `DSH_SNAPSHOT=refresh pnpm run test:snapshot:refresh` | — |
| Web 浏览器快照 | `pnpm run test:web` | `node 24 / web` |
| 性能基准 | `pnpm run test:bench` | `node 24 / benchmarks` |
| Lint | `pnpm run lint` | `node 24 / static` |
| Typecheck | `pnpm run typecheck` | `node 24 / static` |
| 文档同步 | `pnpm run doc-sync` | `node 24 / static` |
| 卫生检查 | `pnpm run hygiene` | — |

### A.2 环境变量速查表

| 变量 | 用途 | 值 |
|---|---|---|
| `DSH_SNAPSHOT` | 快照模式 | `replay`(默认) \| `record` \| `refresh` |
| `DSH_SNAPSHOT_MAX_CONCURRENCY` | 快照并发 | 正整数,默认 5 |
| `DSH_EXAMPLE_MODE` | 示例模式 | `src` \| `lib` |
| `DSH_E2E_MAX_WORKERS` | E2E 并发 | 正整数,默认 4 |
| `DSH_GATE_CONCURRENCY` | 门禁并发 | 正整数,CI: `'8'` |
| `DSH_GATE_FAIL_FAST` | 快速失败 | `'1'` |
| `DEEPSEEK_API_KEY` | DeepSeek 密钥 | 从环境或 `.env` |
| `DEEPSEEK_BASE_URL` | 自定义端点 | 覆盖公共端点 |
| `DEEPSEEK_VISION_E2E` | 启用 vision e2e | `'1'` |
| `DEEPSEEK_FLASH_E2E` | 启用 flash e2e | `'1'` |
| `DEEPSEEK_IN_HISTORY_MODEL` | in-history 模型 | 模型 id |

### A.3 测试支持包速查表

| 包 | 主要导出 | 用途 |
|---|---|---|
| `@deepseek-ai/dsh-session-snapshot` | `defineAcpSnapshotSuite`, `defineHeadlessSnapshotSuite`, `defineSdkSnapshotSuite`, `defineWebSnapshotSuite` | 快照套件工厂 |
| `@deepseek-ai/dsh-agent-loop-testkit` | `mountAgentLoopTestDependencies`, `mountAgentLoopTestHarness`, `createInboxStub`, `unsupportedInbox` | Agent loop 测试工具 |
| `@deepseek-ai/dsh-llm-replay` | (插件,挂载) | 无密钥 LLM 回放 |
| `@deepseek-ai/dsh-llm-mock-server` | `MockLlmServer` | 脚本化 mock 服务器 |
| `@deepseek-ai/dsh-loader-smoke` | 启动工具 | 模式感知子进程启动 |

---

## 附录 B: 代表性文件清单

### B.1 测试配置

- `vitest.config.ts`: 主单元测试配置
- `vitest.e2e.config.ts`: 真实 API e2e 配置
- `vitest.snapshot.config.ts`: 快照 replay/record/refresh 配置
- `vitest.web.config.ts`: Web 浏览器快照配置
- `vitest.bench.config.ts`: 性能基准配置
- `vitest.expected.config.ts`: 预期输出配置

### B.2 测试支持源码

- `packages/test-support/session-snapshot/src/suite.ts`: 场景表套件工厂
- `packages/test-support/session-snapshot/src/launcher.ts`: 子进程/客户端启动
- `packages/test-support/session-snapshot/src/harness.ts`: 脚本化场景驱动
- `packages/test-support/session-snapshot/src/normalize.ts`: 归一化器
- `packages/test-support/agent-loop-testkit/src/index.ts`: loop 测试工具
- `packages/test-support/llm-replay/src/index.ts`: 回放插件
- `packages/test-support/llm-replay/src/script.ts`: 脚本派生

### B.3 代表性测试

- `packages/llm/llm-deepseek/tests/adapter.e2e.ts`: DeepSeek 真实 API e2e
- `packages/acp/acp/tests/harness.ts`: `makeBridgeHarness` 实现
- `packages/acp/acp/tests/edges.spec.ts`: 桥接边缘 case
- `snapshots/sdk/sdk.snapshot.ts`: SDK 场景注册
- `snapshots/acp/acp.snapshot.ts`: ACP 场景注册
- `apps/cli/tests/profiles/acp/tests/acp.e2e.ts`: ACP profile 冒烟

### B.4 文档

- `AGENTS.md`: 仓库规约
- `docs/architecture.md`: 架构
- `docs/testing.md`: 测试策略
- `docs/postmortem/0001-acp-default-export-drops-inject.md`: 事故案例
- `packages/test-support/session-snapshot/README.md`: 快照支持
- `packages/test-support/agent-loop-testkit/README.md`: Loop 测试工具
- `snapshots/AGENTS.md`: 快照所有权规则

---

## 结语

DeepSeek Harness 的功能验证测试体系是一个精心设计、多层协同的质量保障体系。从单元测试到快照回放,从覆盖率门禁到真实 API e2e,每一层都有明确职责,互相补充而不重叠。postmortem 0001 的教训催生了真实入口路径测试的强制要求,确保「绿色覆盖率」不再等同于「破损产品」。

本调研深入到具体符号名(`defineAcpSnapshotSuite`, `makeBridgeHarness`, `mountAgentLoopTestHarness`)、文件路径(`packages/test-support/session-snapshot/src/suite.ts`, `snapshots/sdk/text-turn/`)与 API 签名,可作为添加新测试场景的操作手册。无论是新插件、新适配器、新 profile,还是新事件,开发者都能在本文档中找到对应的测试策略与具体步骤。

**关键要点:**

1. **测试真实入口路径**: 手动挂载无法验证 Loader/export 机制
2. **优先真实实现**: 只 mock 开销高或非确定性边界
3. **快照是证据**: 录制会话 JSONL 保留完整变更证据
4. **覆盖率是必要非充分条件**: 100% 行覆盖 + profile-level 冒烟 = 可信绿色
5. **从事故学习**: postmortem 催生护栏,护栏成文为规则

未来改进方向:

- 补充测试隔离诊断手册
- 完善性能基准指南
- 深化 Web 浏览器快照调试文档
- 映射 Python SDK 测试完整架构

---

**文档元数据**

- **版本:** 1.0
- **作成日:** 2026-09-17
- **作者:** Cloud Agent (dsh 源码调研)
- **范围:** deepseek-harness fork @ 2026-09-17
- **状态:** 已完成

---

**变更历史**

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-09-17 | 1.0 | 初始版本 |
