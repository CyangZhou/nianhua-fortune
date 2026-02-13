# OpenWork 工作流程深度分析

## 项目概述

**OpenWork**（现更名为 **Accomplish™**）是一个开源的 AI 桌面代理应用，作为 Claude Cowork/Codex 的开源替代方案。

| 属性 | 说明 |
|------|------|
| GitHub | `different-ai/openwork` → `accomplish-ai/accomplish` |
| 许可证 | MIT |
| 技术栈 | Tauri 2.x + SolidJS + TailwindCSS |
| 核心引擎 | OpenCode CLI |

---

## 核心定位

```
OpenCode 是「引擎」
OpenWork 是「体验层」
```

- **OpenCode**: 提供 Agent 能力、API、SDK
- **OpenWork**: 提供用户界面、权限管理、工作流编排、多端接入

---

## 核心哲学

### 1. Local-first, Cloud-ready
- 一键本地运行
- 可连接云端工作流

### 2. Composable（可组合）
- 桌面应用、WhatsApp/Slack/Telegram 连接器、服务器模式可自由组合
- 无锁定

### 3. Ejectable（可退出）
- 基于 OpenCode，所有 OpenCode 能力都可用
- 即使 UI 还没实现，也能通过底层能力完成

### 4. Sharing is Caring（分享优先）
- 从单人使用开始，一键分享给团队

---

## 双模式架构

### Mode A - Host 模式（桌面/服务器）

```
┌─────────────────────────────────────────┐
│           OpenWork Desktop              │
│  ┌─────────────────────────────────┐    │
│  │         Tauri Shell             │    │
│  │  ┌───────────────────────────┐  │    │
│  │  │    SolidJS UI (Renderer)  │  │    │
│  │  └───────────────────────────┘  │    │
│  └─────────────────────────────────┘    │
│                    ↓ spawn              │
│  ┌─────────────────────────────────┐    │
│  │   OpenCode Server (localhost)   │    │
│  │   127.0.0.1:4096               │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

**工作流程**:
1. OpenWork 启动 OpenCode 服务器
2. UI 通过 SDK 连接本地服务器
3. 用户选择项目文件夹
4. 所有任务在本地执行

### Mode B - Client 模式（移动端/远程）

```
┌──────────────────┐         ┌──────────────────┐
│  OpenWork Client │  ←LAN→  │  OpenWork Host   │
│  (iOS/Android)   │         │  (Desktop/Server)│
│                  │         │                  │
│  - 远程控制      │         │  - OpenCode引擎  │
│  - 监控任务      │         │  - 文件系统访问  │
│  - 审批权限      │         │  - 实际执行      │
└──────────────────┘         └──────────────────┘
```

**配对流程**:
1. Host 端显示 QR 码
2. Client 扫码配对
3. 通过 LAN 或隧道安全传输
4. Client 调用 `global.health()` 验证连接

---

## OpenCode 扩展原语

OpenWork 完全基于 OpenCode 的原生扩展能力：

| 原语 | 用途 | 适用场景 |
|------|------|----------|
| **MCP** | 第三方 OAuth 认证流 | 需要安全暴露认证边界时 |
| **Plugins** | 代码级工具扩展 | 需要权限范围控制的工具 |
| **Skills** | 自然语言行为模式 | 可重复的工作流模板 |
| **Agents** | 多模型任务执行 | 需要不同模型处理子任务 |
| **Commands** | `/` 触发的快捷命令 | 用户自定义快捷操作 |
| **Bash/CLI** | 原始命令行 | 高级用户/内部调试 |

---

## 核心工作流程

### 1. 任务执行流程

```
用户输入目标
    ↓
生成执行计划（可编辑）
    ↓
用户批准计划
    ↓
创建 Session: client.session.create()
    ↓
发送 Prompt: client.session.prompt()
    ↓
订阅事件流: client.event.subscribe()
    ↓
实时渲染步骤 + 工具调用
    ↓
权限请求弹窗（如需要）
    ↓
用户审批: client.permission.reply()
    ↓
显示产出物（Artifacts）
```

### 2. 权限管理流程

```
事件触发权限请求
    ↓
UI 显示请求详情（范围 + 原因）
    ↓
用户选择:
├── Allow Once（仅本次）
├── Allow for Session（本次会话）
└── Deny（拒绝）
    ↓
调用 client.permission.reply({ requestID, reply })
    ↓
记录到审计日志
```

### 3. 文件夹授权模型

```
┌─────────────────────────────────────────┐
│          双层授权机制                    │
├─────────────────────────────────────────┤
│  Layer 1: OpenWork UI 授权              │
│  - 用户显式选择允许的文件夹              │
│  - 按配置文件记忆授权根目录              │
├─────────────────────────────────────────┤
│  Layer 2: OpenCode 服务器权限            │
│  - 按需请求权限                          │
│  - OpenWork 拦截并显示                   │
└─────────────────────────────────────────┘

规则:
- 默认拒绝授权根目录外的访问
- "Allow once" 不扩展持久权限
- "Allow for session" 仅限当前会话
- "Always allow" 必须显式且可撤销
```

---

## 产品原语

### 1. Task（任务）
- 用户描述的目标
- 一个 Run = 一个 OpenCode Session + 事件流

### 2. Plan（计划）
- 执行前生成（可编辑）
- 执行中更新（步骤状态 + 时间戳）
- 存储为结构化 JSON

### 3. Step（步骤）
- 每个工具调用对应一个步骤
- 包含：工具名、参数摘要、权限状态、起止时间、输出预览

### 4. Artifact（产出物）
- 创建/修改的文件
- 生成的文档/表格/演示文稿
- 导出的日志和摘要

### 5. Audit Log（审计日志）
- Prompts、Plan、Tool calls、Permission decisions、Outputs

---

## 技术栈详解

```
┌─────────────────────────────────────────────────────┐
│                    技术架构                          │
├─────────────────────────────────────────────────────┤
│  Desktop/Mobile Shell    │  Tauri 2.x              │
│  Frontend                │  SolidJS + TailwindCSS  │
│  State Management        │  Solid stores + IndexedDB│
│  IPC                     │  Tauri commands + events │
│  OpenCode Integration    │  Spawn CLI / embed binary│
│  Package Manager         │  pnpm (monorepo)        │
└─────────────────────────────────────────────────────┘
```

### Monorepo 结构

```
openwork/
├── packages/
│   ├── app/              # UI 界面 (SolidJS)
│   ├── desktop/          # 桌面应用 (Tauri)
│   ├── headless/         # CLI 版本 (openwrk)
│   ├── server/           # OpenWork 服务器
│   ├── owpenbot/         # WhatsApp/Telegram 机器人
│   ├── landing/          # 着陆页
│   └── agent-lab/        # Agent 实验室
├── .opencode/
│   ├── agent/            # Agent 配置
│   ├── commands/         # 自定义命令
│   └── skills/           # 技能模板
├── AGENTS.md             # Agent 开发指南
├── ARCHITECTURE.md       # 架构文档
├── PRODUCT.md            # 产品需求
├── VISION.md             # 愿景文档
├── PRINCIPLES.md         # 设计原则
└── INFRASTRUCTURE.md     # 基础设施原则
```

---

## SDK 使用模式

### 启动服务器 + 客户端（Host 模式）

```typescript
import { createOpencode } from "@opencode-ai/sdk/v2";

const opencode = await createOpencode({
  hostname: "127.0.0.1",
  port: 4096,
  timeout: 5000,
  config: {
    model: "anthropic/claude-3-5-sonnet-20241022",
  },
});

const { client } = opencode;
```

### 连接已有服务器（Client 模式）

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk/v2/client";

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  directory: "/path/to/project",
});
```

### 核心 API

| API | 用途 |
|-----|------|
| `client.global.health()` | 健康检查 |
| `client.event.subscribe()` | SSE 事件订阅 |
| `client.session.create()` | 创建会话 |
| `client.session.prompt()` | 发送提示 |
| `client.session.abort()` | 中止会话 |
| `client.permission.reply()` | 权限响应 |
| `client.find.text()` | 文本搜索 |
| `client.find.files()` | 文件搜索 |
| `client.file.read()` | 读取文件 |
| `client.config.get()` | 获取配置 |

---

## 基础设施原则

### 1. CLI-first
- 所有组件必须可通过 CLI 运行
- UI 是 CLI 的封装，不是替代

### 2. Unix-like 接口
- JSON stdout、flags、env vars
- 可读日志、可预测退出码

### 3. Sidecar-composable
- 任何组件可作为 sidecar 运行
- UI 连接 CLI 暴露的相同接口

### 4. 清晰边界
- OpenCode 是引擎
- OpenWork 是薄配置 + UX 层

### 5. 可观测性
- 健康端点、结构化日志
- 每次配置变更记录审计事件

---

## 用户流程

### 首次使用（Host 模式）

```
1. 欢迎 + 安全概述
2. 选择工作空间文件夹
3. 选择允许访问的文件夹
4. 配置 Provider/Model
5. 健康检查
6. 运行测试会话
7. 显示示例命令
```

### 快速任务流程

```
1. 用户输入目标
2. OpenWork 生成计划
3. 用户批准
4. 创建 Session
5. 发送 Prompt
6. 订阅事件流
7. 渲染输出 + 步骤
8. 显示产出物
```

---

## 目标用户

| 用户类型 | 描述 | 需求 |
|----------|------|------|
| **Bob (IT)** | 已使用 OpenCode | 分享 workflows 给团队 |
| **Susan (Accounting)** | 非技术用户 | 开箱即用的解决方案 |
| **知识工作者** | "帮我做这个" | 带护栏的工作流 |
| **移动优先用户** | 手机启动/监控 | 远程控制能力 |
| **高级用户** | 追求速度和深度 | UI 对等 + 检查能力 |

---

## 成功指标

| 指标 | 目标 |
|------|------|
| 首次任务成功时间 | < 5 分钟 |
| 无终端回退成功率 | > 80% |
| 权限提示理解率 | 高理解 + 低误拒 |
| UI 性能 | 60fps、<100ms 交互延迟 |

---

## 与 Claude Cowork 对比

| 特性 | OpenWork | Claude Cowork |
|------|----------|---------------|
| 开源 | ✅ MIT | ❌ 闭源 |
| 本地优先 | ✅ | ❌ 云端 |
| 模型选择 | ✅ 多模型 | ❌ 仅 Claude |
| 自托管 | ✅ | ❌ |
| 移动端 | ✅ Client 模式 | ❌ |
| 价格 | 免费 | 订阅制 |

---

## 关键设计原则

1. **Parity（对等）**: UI 操作映射到 OpenCode Server API
2. **Transparency（透明）**: 计划、步骤、工具调用、权限可见
3. **Least Privilege（最小权限）**: 仅用户授权文件夹 + 显式审批
4. **Prompt is Workflow**: 产品逻辑存在于 prompts、rules、skills
5. **Graceful Degradation**: 缺失访问时引导用户

---

## 总结

OpenWork 的核心价值在于：

1. **薄层设计** - 不重复造轮子，最大化利用 OpenCode 原语
2. **双模式架构** - Host/Client 分离实现跨端体验
3. **权限透明** - 双层授权 + 审计日志
4. **可组合性** - CLI-first + Sidecar 架构
5. **开放生态** - Skills/Plugins/MCP 原生支持

这是一个将 Agent 能力产品化的优秀范例，值得借鉴其架构思想和工程实践。
