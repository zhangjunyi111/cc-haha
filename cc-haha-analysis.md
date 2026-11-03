# cc-haha 项目分析报告

> 分析对象：`zhangjunyi111/cc-haha`（上游：`NanmiCoder/cc-haha`）
> 分析时间：2026-11-03
> 代码基线：`main` @ `569bf22 fix(power): wait for prompt predictions before idle actions (#1787)`
> 说明：本文基于实际源码、文档、构建脚本与仓库元数据整理，不含商业背书或法律意见。

---

## 结论先说

这不是一个普通的“Claude Code 套壳 UI”，而是**把 Claude Code 的完整 CLI 内核内置进 Electron 桌面端的重型二次发行版**，外加手机 App、IM 接入和本地 Server。工程量非常大（约 136 万行 TS/TSX），工程化程度远超一般个人项目；但它的**源码来源和法律授权存在根本性风险**，这是评估这个项目时最需要先知道的事。

另外：本仓库 `zhangjunyi111/cc-haha` 的 README 徽章、文档与发布链接均指向上游 **`NanmiCoder/cc-haha`**。这是一个 fork，通常落后于上游，看项目应以 NanmiCoder 仓库为准。

---

## 一、它到底是什么

自述定位（`docs/start/index.md`）说得很直白：

> “cc-haha 的内核就是从 Claude Code 源码修复而来的一份 CLI，桌面端是包在它外面的图形界面。”

代码层面可以证实这一点：

- `src/` 下是 Claude Code 的完整内部实现：`QueryEngine.ts`、`main.tsx`、`tools/`（BashTool、FileEditTool、MCPTool 等 60+ 工具）、`commands/`、`services/api/claude.ts`、`utils/permissions/`、`vendor/`、`memdir/`、`proactive/` 等。
- 存在 **1509 处 `tengu_*` 内部埋点事件名**（如 `tengu_streaming_error`、`tengu_auto_mode_config`），这是 Claude Code 内部遥测命名，与官方实现高度一致。
- 根 `package.json` 名字是 `claude-code-local`，版本 `999.0.0-local`，bin 叫 `claude-haha`。

所以它包含三部分：

| 部分 | 内容 | 性质 |
|---|---|---|
| CLI 内核 | 从 Claude Code 源码“修复”而来的 `src/` | 绝大多数非作者原创 |
| 桌面端 | `desktop/` Electron + React | 作者自建 |
| 外围生态 | 手机 App、IM Adapter、H5、文档站、质量门禁 | 作者自建 |

---

## 二、架构

四类进程协作，边界划得很清楚：

```text
Electron main
├── Chromium renderer (React UI)
├── claude-sidecar server   (Bun.serve: REST + WS + Provider 代理 + H5 静态资源)
│   └── 按会话 spawn CLI 子进程
└── claude-sidecar adapters (每个 IM 平台独立进程)
    ├── Telegram / Feishu / WeChat / DingTalk / WhatsApp
    └── WeCom / QQ / Slack
```

关键设计：

- **统一 Sidecar 入口** `desktop/sidecars/claude-sidecar.ts`，一个二进制三份用途：`server` / `cli` / `adapters`。
- **本地 Server** 是会话权威源，不是 Renderer。`Bun.serve` 同一个 fetch 边界处理 `/api/*`、`/ws/:sessionId`（桌面/H5/宠物）、`/sdk/:sessionId`（内部 CLI 通道）、OAuth 回调、受限文件访问。
- **WebSocket 语义**写得很细：30s ping / 10s pong 超时 / 指数退避封顶 30s / 断线内存队列 / 重连后 `sync_state`；最后一个客户端断开不会立即杀任务，有宽限期。
- **Provider 协议代理** `src/server/proxy/`：Anthropic Messages、OpenAI Chat Completions、OpenAI Responses 三套互转，含流式转换。
- **本地索引**：SQLite + 全文检索 + 文件监听，启动过程可续跑、分阶段日志。
- **持久化分区明确**：UI 偏好、会话、Provider 设置、Electron 状态、IM 配置各自独立，且要求任何结构变更必须有前向迁移。

---

## 三、规模与技术栈

| 区域 | 文件数 | TS/TSX/JS 行数 | 说明 |
|---|---:|---:|---|
| `src/` | 3,395 | ~894k | CLI、工具、本地 Server |
| `desktop/` | 1,663 | ~366k(src) + 32k(electron) | React 18 + Vite 8 + Zustand 5 + Tailwind 4 |
| `mobile/` | 504 | ~70k | Expo 58 / React Native 0.88 / Expo Router |
| `adapters/` | 148 | ~31k | 9 个 IM 平台 |
| `native/` | 3,173 | Rust 2,693 + Swift 124 + Python 158 | 含 vendored 的 Vercel agent-desktop (Apache-2.0) |
| `docs/` | 350 | 116 篇 Markdown | 中英双语 + 文档站 |
| `scripts/` | 196 | — | 质量门禁、发布、PR 路由 |

其他：**1,673 个测试文件**、8 条 GitHub Actions、57 份 release notes（当前 desktop v0.7.6 / mobile v0.1.2）。所有主要目录都有 `AGENTS.md`，是明显面向 AI Agent 协作开发的仓库。

技术栈：Bun 1.4.3、TypeScript、Electron 42、React、Vite、Ink（TUI）、Commander、Anthropic SDK、MCP SDK、OpenTelemetry、node-pty/xterm、Expo、Zustand。

---

## 四、工程成熟度：这是加分项

- **分层质量门禁**：`scripts/quality-gate/` 有 `pr` / `baseline` / `release` 三种模式，包含 deterministic mock-LLM 端到端、replay cassettes、provider contract、desktop smoke、package smoke、persistence upgrade、coverage 阈值、quarantine 机制。
- **测试隔离纪律严格**：AGENTS.md 明确禁止测试读写真实 `~/.claude`、keychain、token、用户项目设置，要求临时 HOME、假凭据、loopback/拦截请求。
- **安全边界有意识**：H5 Token、精确 Origin 校验、反向代理头识别、`/sdk/*` 不对外、CORS 不被当认证、Computer Use 一次授权边界明确。
- **数据库/配置变更**强制“前向迁移 + 旧 fixture + `check:persistence-upgrade`”。
- **发布流程完整**：签名公证、自动更新、桌面/移动双发布工作流。
- **PR/Issue 治理**：根目录有两份长篇 triage 和 remediation 计划，说明作者在认真处理反馈。

---

## 五、主要风险（按重要性排序）

### 1. 源码来源与许可证——最大的问题

`LICENSE` 是自定义的 **cc-haha Community Source License 1.0**，明确写着：

> 不是 MIT / Apache / BUSL，**不是 OSI 认可的开源许可证**。

它允许个人非商业、教育、研究及组织内部自用；接单、收费交付、商业分发、付费服务都需作者书面授权。

但关键矛盾在于：**这个许可证只能约束作者拥有版权的部分**。CLI 内核既然来自 Anthropic 的 Claude Code 源码，作者并没有权利对它重新授权或商用授权。也就是说：

- 许可证里“授予你使用、修改、再分发”的条款，对内核部分在法律上可能无效；
- 企业做合规/法务审查时基本会直接卡住；
- 把它用于商业项目风险很高，不能因为许可证写了“允许内部自用”就认为安全。

> 本节仅为技术性风险提示，不构成法律意见。

### 2. 信任模型很重

装一个桌面 App，等于：

- 让一个本地服务执行任意 shell 命令；
- 处理你的 Anthropic / OpenAI / Grok 账号登录与 API Key；
- 开启 Computer Use 后可控制桌面；
- 接受来自手机、H5、9 个 IM 平台的指令。

虽然有权限审批和 Token 机制，但**整个信任链最终落在作者一人**。安装包是闭源分发的；README 还提到 draft/unsigned 包需要手动放行。企业环境应视为高风险软件。

### 3. 可维护性 / Bus factor

- README 自述“个人利用业余时间维护”。
- 多个超大文件：`src/server/ws/handler.ts` 6,188 行、`sessionService.ts` 6,118 行、`desktop/src/stores/chatStore.ts` 8,516 行。
- 内核深度绑定 Claude Code 内部实现，上游更新无法自动合入，需要持续人工“修复”，长期维护成本极高。

### 4. Fork 与版本

- 当前克隆是 fork，且为 `--depth 1` 浅克隆，看不到完整分支/历史演进。
- 版本号分层：根 `999.0.0-local`、desktop `0.7.6`、mobile `0.1.2`。

### 5. 依赖激进

React 19 / Expo 58 / TS 6.0 / Vite 8 / Tailwind 4 等都很新，升级节奏快，插件生态和稳定性需要自己验证。

---

## 六、如何运行

```bash
# 需要 Bun 1.4.3
bun install

# 1) 纯 CLI/TUI
bun run start

# 2) 本地 Server（默认 127.0.0.1:3456）
bun run src/server/index.ts

# 3) 桌面端（需另装 desktop 依赖）
cd desktop && bun install && bun run dev

# 4) 质量门禁
bun run check:impact     # 按改动路径选择检查
bun run verify           # 完整 PR 门禁
```

注意：桌面构建会顺带把 `mobile/` 导出成 H5，所以还需要在 `mobile/` 里单独 `bun install`（Node 22）。干净克隆默认没有 `node_modules`。

---

## 七、总体评价

**技术上**：架构设计、进程边界、WebSocket 语义、安全边界、质量门禁都做得相当扎实，是一个“认真在维护”的大型混合应用，而不是玩具项目。它把 Claude Code 从终端扩展成了桌面 + 手机 + IM 的完整工作台，功能覆盖面很惊人。

**合规和信任上**：由于内核直接来自 Claude Code 源码，自定义许可证无法真正覆盖这部分权利，这是结构性风险。个人学习、研究、自用没问题；企业商用、对外分发、接入敏感代码库前，建议先走法务评估，并优先考虑官方 Claude Code / Claude Agent SDK 方案。

---

## 附：适合谁

| 场景 | 建议 |
|---|---|
| 个人想图形化使用 Claude Code、多模型、手机/IM 审批 | 可以尝试 |
| 想学习 Agent/桌面端/多进程架构 | 代码与文档价值很高 |
| 企业内部自用 | 先做安全与法务评估，视为高风险第三方软件 |
| 商业分发、接单、收费产品 | 不建议，除非取得作者书面授权并解决内核权利问题 |
| 需要 OSI 开源许可 | 不适用 |
