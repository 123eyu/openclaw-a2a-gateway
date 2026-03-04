# OpenClaw A2A Gateway 插件

[OpenClaw](https://github.com/openclaw/openclaw) 的 [A2A（Agent-to-Agent）协议 v0.3.0](https://google.github.io/A2A/) 网关插件 —— 让你的 OpenClaw 智能体与任何兼容 A2A 协议的平台互联互通。

## 功能特性

- **完整实现 A2A v0.3.0 协议** — Agent Card 自动发现、JSON-RPC 2.0 和 REST 端点全覆盖
- **双向通信** — 既能接收外部 A2A 请求，也能主动向对等智能体发送消息
- **Gateway RPC 调度** — 通过网关内部 WebSocket API 将 A2A 消息路由到 OpenClaw 智能体，自动解析会话
- **多层降级方案** — Gateway RPC 不可用时，自动降级到 `dispatchToAgent` 桥接或 `/hooks/wake`
- **Bearer Token 认证** — 可选的入站请求认证
- **对等节点管理** — 配置多个远程 A2A 对等智能体，支持独立的 Bearer/API Key 认证
- **内存任务存储** — 按照 A2A 规范跟踪任务生命周期（working → completed/canceled）
- **网关内部扩展** — 自定义可靠性层，包括 HMAC-SHA256 签名、发件箱模式、幂等性处理和消息路由

## 架构概览

```
                          ┌─────────────────────────────────┐
   外部 A2A 智能体 ──────▶ │  /.well-known/agent.json        │
                          │  /a2a/jsonrpc  (JSON-RPC 2.0)   │
                          │  /a2a/rest     (REST)            │
                          │                                  │
                          │  ┌───────────────────────────┐   │
                          │  │  OpenClawAgentExecutor     │   │
                          │  │  ┌─────────────────────┐   │   │
                          │  │  │ Gateway RPC (WS)    │◀──┼───┼── OpenClaw 网关
                          │  │  │ Legacy 桥接          │   │   │
                          │  │  │ /hooks/wake 降级     │   │   │
                          │  │  └─────────────────────┘   │   │
                          │  └───────────────────────────┘   │
                          │                                  │
   OpenClaw 智能体 ───────▶│  a2a.send（网关方法）            │──▶ 远程 A2A 对等节点
                          └─────────────────────────────────┘
```

## 安装

### 1. 将插件复制到 OpenClaw 工作区

```bash
# 从 OpenClaw 工作区根目录
cp -r /path/to/a2a-gateway plugins/a2a-gateway
cd plugins/a2a-gateway
npm install
```

### 2. 在 OpenClaw 配置中注册

通过 CLI 添加插件配置：

```bash
openclaw gateway config.patch '{
  "plugins": {
    "a2a-gateway": {
      "enabled": true,
      "config": {
        "agentCard": {
          "name": "我的 OpenClaw 智能体",
          "description": "基于 OpenClaw 的 AI 助手",
          "url": "https://your-domain.com/.well-known/agent.json",
          "skills": [
            {
              "id": "chat",
              "name": "通用对话",
              "description": "自然语言对话能力"
            }
          ]
        },
        "server": {
          "host": "0.0.0.0",
          "port": 18800
        },
        "security": {
          "inboundAuth": "bearer",
          "token": "your-secret-token"
        },
        "routing": {
          "defaultAgentId": "main"
        }
      }
    }
  }
}'
```

### 3. 重启网关

```bash
openclaw gateway restart
```

## 配置说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `agentCard.name` | `string` | *必填* | A2A Agent Card 中显示的名称 |
| `agentCard.description` | `string` | `"A2A bridge for OpenClaw agents"` | 智能体描述 |
| `agentCard.url` | `string` | 自动生成 | Agent Card 端点的公开 URL |
| `agentCard.skills` | `array` | *必填* | 技能列表（字符串或 `{id, name, description}` 对象） |
| `server.host` | `string` | `"0.0.0.0"` | HTTP 服务器绑定地址 |
| `server.port` | `number` | `18800` | A2A HTTP 服务器端口 |
| `peers` | `array` | `[]` | 远程 A2A 对等智能体列表（详见下方） |
| `security.inboundAuth` | `"none" \| "bearer"` | `"none"` | 入站认证方式 |
| `security.token` | `string` | `""` | 入站请求的 Bearer Token |
| `routing.defaultAgentId` | `string` | `"default"` | 入站消息默认路由到的 OpenClaw 智能体 |

### 对等节点配置

`peers` 数组中的每个对等节点：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 唯一标识符（在 `a2a.send` 中引用） |
| `agentCardUrl` | `string` | 对等节点的 Agent Card URL |
| `auth.type` | `"bearer" \| "apiKey"` | 出站请求认证方式 |
| `auth.token` | `string` | 认证令牌/密钥 |

**配置示例：**

```json
{
  "peers": [
    {
      "name": "research-agent",
      "agentCardUrl": "https://research.example.com/.well-known/agent.json",
      "auth": {
        "type": "bearer",
        "token": "peer-secret-token"
      }
    }
  ]
}
```

## API 端点

插件启动后暴露以下端点：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/.well-known/agent.json` | GET | A2A Agent Card（自动发现） |
| `/a2a/jsonrpc` | POST | JSON-RPC 2.0 端点（`message/send` 等） |
| `/a2a/rest` | POST | REST 风格端点 |

## 向对等节点发送消息

从任何 OpenClaw 智能体会话中，使用网关方法：

```
a2a.send({ peer: "research-agent", message: { text: "总结近期 LLM 相关论文" } })
```

客户端会依次：
1. 获取对等节点的 Agent Card，发现其服务 URL
2. 发送带有认证头的 JSON-RPC `message/send` 请求
3. 返回响应结果

## 智能体调度流程

当收到入站 A2A 消息时：

1. **Gateway RPC（首选）** — 通过 WebSocket 连接 OpenClaw 网关，解析目标智能体会话，调度消息。等待智能体响应后作为已完成的 A2A 任务返回。

2. **Legacy 桥接（降级）** — Gateway RPC 失败时，降级到 `dispatchToAgent` API。

3. **Hooks Wake（兜底）** — 所有方式都失败时，发送 wake 事件到 `/hooks/wake`，让智能体异步处理消息。

## 内部模块

`src/internal/` 目录包含超出 A2A 规范的网关间通信扩展：

| 模块 | 用途 |
|------|------|
| `envelope.ts` | 自定义 `a2a/v1` 信封格式，支持 TTL、跳数限制和环路检测 |
| `transport.ts` | HTTP 传输层，`/a2a/v1/inbox` 端点 |
| `security.ts` | HMAC-SHA256 签名、Nonce 缓存和对等节点注册表 |
| `routing.ts` | 基于 route-key 和 agent-id 的消息路由 |
| `outbox.ts` | 可靠的至少一次投递，指数退避重试 |
| `idempotency.ts` | SHA-256 负载指纹识别和去重 |
| `metrics.ts` | 协议指标收集和结构化日志 |

## 测试

```bash
npm test
```

使用 Node.js 内置测试运行器（`node:test`）执行测试套件。测试覆盖：
- Agent Card 生成（A2A v0.3.0 字段验证）
- Gateway RPC 调度（首选路径）
- Legacy 桥接降级
- 任务取消生命周期
- 通过 `a2a.send` 向对等节点发送消息

## 技术栈

- **运行时:** Node.js（ESM）
- **A2A SDK:** `@a2a-js/sdk` v0.3.0
- **HTTP 框架:** Express 4
- **开发语言:** TypeScript 5
- **测试:** `node:test` + `supertest`

## 许可证

MIT

## 相关链接

- [A2A 协议规范](https://google.github.io/A2A/)
- [OpenClaw 文档](https://docs.openclaw.ai)
- [@a2a-js/sdk](https://www.npmjs.com/package/@a2a-js/sdk)
