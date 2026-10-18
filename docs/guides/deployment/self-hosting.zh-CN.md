---
title: "自托管 eve"
description: "把 eve agent 作为 Node 服务运行，自带 workflow 存储、沙箱 provider 与路由。"
---

当你自己运维 Node 服务、容器平台或反向代理时，选择自托管 eve。你运行 eve 的 Nitro 服务器，并自行选择存储 workflow 状态、执行沙箱会话的基础设施。

## 构建并启动 Node 服务

先构建 agent，再启动生成的服务器：

```bash
eve build
PORT=3000 eve start --host 0.0.0.0
```

构建把 Nitro 服务器写入 `.output/`。`eve start` 服务于该产物，接受 `PORT` 环境变量或 `--port` 参数。

请用你运行其他 Node Web 服务同样的进程管理器或容器平台来运行此进程，并在该平台中配置传输层安全（TLS）、扩缩容、重启和日志收集。

## 配置模型访问与路由鉴权

在非 Vercel 主机上通过 Vercel AI Gateway 使用字符串模型 ID 时，设置 `AI_GATEWAY_API_KEY`。要直连某个 provider，请安装其 [AI SDK provider 包](https://ai-sdk.dev/docs/foundations/providers-and-models)，然后在 `agent.ts` 中传入其模型对象并设置对应的 API Key。示例见 [Agent 配置](../../agent-config#set-the-model)。

不要把 `vercelOidc()` 当作 Vercel 之外唯一的生产鉴权方式。请配置 Basic 认证、JSON Web Token（JWT）校验、通用 OpenID Connect（OIDC），或你的主机能够验证的自定义校验器。参见[鉴权](../auth-and-route-protection)。

## 持久化 workflow 状态

默认的本地 Workflow world 把运行状态存储在 `.eve/.workflow-data` 之下。请把该目录挂载到持久存储上，使运行状态在进程与容器替换后仍然存活。

你也可以在根 `agent.ts` 中选择一个已安装的 Workflow world 包：

```typescript
import { defineAgent } from "eve";

export default defineAgent({
  experimental: {
    workflow: {
      world: "@acme/eve-workflow-world",
    },
  },
});
```

该包必须导出默认工厂函数或 `createWorld()` 函数。凭据与主机选项请从运行时环境变量读取。安装的 world 必须与你的 eve 版本基于同一条 `@workflow/*` 产品线构建。当前产品线是 `5.0.0-beta`，运行时会拒绝不兼容的协议版本。

底层 Workflow 软件开发工具包（SDK）抽象见 [Workflow Worlds](https://workflow-sdk.dev/worlds)。

## 选择沙箱 provider

默认沙箱环境按本地可用性在 Docker、microsandbox、just-bash 之间自动选择。你可以显式指定其一，或用 `defineSandboxProvider()` 为你的容器、虚拟机、隔离服务实现自定义 provider。

除非你希望自托管进程创建托管式 Vercel 沙箱，否则不要选择 `VercelSandbox.environment()`。provider 配置与选择顺序见[沙箱](../../sandbox)。

## 配置代理路由

通过反向代理或入口（ingress）转发**两个**运行时路由前缀：

- `/eve/` 提供健康检查、会话、流、频道、工具与子 Agent
- `/.well-known/workflow/` 接收 workflow 回调

只转发 `/eve/` 的代理能让会话启动，但回调无法到达 eve 时运行会卡住。两个前缀都要保留，且不要改写其路径。

## 运行工作区成员

[Agent 工作区](../../concepts/project-structure#several-root-agents)在运行时不要求 Vercel，也不要求前端。请在各成员自己的目录下构建：根目录 `eve build` 产出的是 Vercel 工作区部署，不是一组自托管的 Node 服务器。

对于包含 `support` 和 `research` 的工作区，在 Vercel 构建环境之外、从工作区根目录执行：

```bash
(cd agents/support && npx eve build)
(cd agents/research && npx eve build)
```

在各自独立的终端里启动每个构建好的 agent，或在进程管理器中配置这些命令：

```bash
(cd agents/support && npx eve start --host 127.0.0.1 --port 3001)
(cd agents/research && npx eve start --host 127.0.0.1 --port 3002)
```

例如，下面这份 Caddy 配置给每个 agent 独立的域名，并在不改路径的前提下同时转发 `/eve/` 与 `/.well-known/workflow/`。把示例主机名指向你的服务器，并在同一台机器上运行 Caddy：

```text
support.example.com {
    reverse_proxy 127.0.0.1:3001
}

research.example.com {
    reverse_proxy 127.0.0.1:3002
}
```

把上文介绍的鉴权、持久存储和沙箱配置应用到每个成员。使用默认本地 Workflow world 时，分别持久化每个成员自己的 `.eve/.workflow-data` 目录。如果 agent 之间彼此委派，请用对方的 URL 与凭据配置显式的[工作区同级 transport](../../subagents#vercel-workspace-peers)；默认 transport 依赖 Vercel。确保回调 URL 能被调用它的服务访问到。

### 添加同级前端

`apps/web/` 下的前端是由你的主机托管的另一个服务，不由 `eve start` 管理。用其框架自己的命令构建并启动。浏览器客户端可以使用 `useEveAgent({ host: "https://support.example.com" })`；为该部署配置 [CORS](../../channels/eve#cors) 与浏览器凭据。或者，通过反向代理把每个 agent 挂载到前端的域名下。

按路径挂载时，先把公共前缀剥掉再转发请求给 agent，并在该 agent 的构建与运行环境中设置 `EVE_PUBLIC_ROUTE_PREFIX`。它的 workflow 回调路由与 eve 路由都要转发。保持浏览器客户端、回调 URL、同级 transport 与公共挂载路径一致。

如果你希望由 Next.js 启动构建好的 agent 进程并提供面向浏览器的代理路由，也可以改用 [`eve/next`](../frontend/nextjs#dev-vs-deploy-topology)。该集成是可选项；自托管进程管理器不会使用 `eve/vercel` 配置。

## 运行定时任务

标准的 `eve build && eve start` 路径会启动 Nitro 的调度执行器。如果你把产物适配到自定义的纯 HTTP 主机或 preset，请运行 Nitro 的计划任务，或用你自己的调度器执行同样的工作。

## 验证服务

在代理与鉴权配置生效后，检查健康路由：

```bash
curl https://your_agent.example.com/eve/v1/health
```

然后连接开发 TUI 并完成一次真实轮次：

```bash
eve remote connect --url https://your_agent.example.com
```

## 继续配置生产环境

用以下指南加固并观测已部署的 agent：

- [鉴权](../auth-and-route-protection)：配置主机的路由策略
- [可观测性](../../observability/instrumentation)：导出追踪、诊断运行时故障
- [沙箱](../../sandbox)：选择并加固沙箱 provider
