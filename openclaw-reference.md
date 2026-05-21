# OpenClaw 项目参考文档

本文档是 OpenClaw 项目的配置和结构参考。修改 OpenClaw 配置前必须先阅读本文档。

> 最后同步：2026-05-20（基于当前运行中的 openclaw.json 配置）

## 版本信息

- OpenClaw 版本：2026.5.18
- 安装方式：npm 全局安装（`npm install -g openclaw@latest`）
- 二进制路径：`/usr/bin/openclaw`
- 数据目录：`/root/.openclaw`
- 运行环境：Linux，Node.js

---

## 目录结构

```
/root/.openclaw/
├── openclaw.json              # 核心配置文件（最重要）
├── docs/                      # 项目文档
│   ├── MAINTENANCE.md         #   运维手册
│   ├── openclaw-reference.md  #   配置参考文档
│   └── MULTI-AGENT-CROSS-LEARNING-v2.md  # 多Agent交叉学习方案
├── update-check.json          # 版本更新检查记录
├── agents/                    # Agent 运行时数据
│   ├── main/                  # main agent（程一）
│   │   ├── agent/
│   │   │   └── auth-profiles.json  #   该 agent 的 API 认证密钥
│   │   └── sessions/          #   会话记录（对话历史、轨迹）
│   └── finance/               # finance agent（程二）
│       ├── agent/
│       └── sessions/
├── workspace/                 # main agent 的工作空间（git 仓库）
│   ├── AGENTS.md              #   Agent 行为规范
│   ├── SOUL.md                #   Agent 性格/人格定义
│   ├── IDENTITY.md            #   Agent 身份信息
│   ├── USER.md                #   用户信息
│   ├── TOOLS.md               #   工具/设备笔记
│   ├── HEARTBEAT.md           #   心跳检查配置
│   ├── MEMORY.md              #   长期记忆
│   └── memory/                #   Agent 记忆存储目录
├── workspace-finance/         # finance agent 的工作空间（git 仓库）
├── identity/                  # 设备身份（device.json，用于配对）
├── cron/                      # 定时任务（jobs.json）
├── tasks/                     # 后台任务 SQLite 数据库
├── plugins/                   # 插件安装记录（installs.json）
├── plugin-skills/             # 插件 skill 符号链接
├── canvas/                    # Canvas Web UI
├── completions/               # Shell 自动补全脚本
├── logs/                      # Gateway 运行日志
├── credentials/               # 飞书配对凭证数据
├── memory/                    # SQLite 记忆数据库
├── wiki/                      # memory-wiki 数据
├── extensions/                # 插件安装目录（如 openclaw-lark）
└── .claude/                   # Claude Code 配置
```

---

## 核心配置文件：openclaw.json

这是 OpenClaw 最重要的配置文件，所有运行时行为都由它控制。

### 顶层结构

```json
{
  "agents": {},       // Agent 定义和默认配置
  "models": {},       // 模型 Provider 和模型列表定义（内联在配置中）
  "gateway": {},      // 网关服务配置
  "channels": {},     // 聊天通道（飞书、Telegram 等）
  "bindings": [],     // 路由绑定（通道 → Agent 映射）
  "plugins": {},      // 插件配置
  "auth": {},         // 认证配置
  "tools": {},        // 工具配置
  "session": {},      // 会话管理
  "messages": {},     // 消息格式配置
  "commands": {},     // 命令/权限配置
  "wizard": {},       // 安装向导状态（只读，不要手动改）
  "meta": {}          // 元数据（只读）
}
```

### agents 配置（当前实际值）

```json
{
  "agents": {
    "defaults": {
      "workspace": "/root/.openclaw/workspace",
      "model": {
        "primary": "volcengine-plan/ark-code-latest"
      },
      "models": {
        "volcengine-plan/ark-code-latest": {},
        "volcengine-plan/doubao-seed-code": {},
        "volcengine-plan/glm-5.1": {},
        "volcengine-plan/deepseek-v3.2": {},
        "volcengine-plan/doubao-seed-2.0-code": {},
        "volcengine-plan/doubao-seed-2.0-pro": {},
        "volcengine-plan/doubao-seed-2.0-lite": {},
        "volcengine-plan/minimax-latest": {},
        "volcengine-plan/kimi-k2.6": {},
        "deepseek/deepseek-v4-pro": {}
      }
    },
    "list": [
      {
        "id": "main",
        "name": "程一-秘书",
        "workspace": "/root/.openclaw/workspace",
        "agentDir": "/root/.openclaw/agents/main/agent",
        "model": "deepseek/deepseek-v4-pro",
        "identity": {
          "name": "程一",
          "emoji": "📋"
        }
      },
      {
        "id": "finance",
        "name": "finance",
        "workspace": "/root/.openclaw/workspace-finance",
        "agentDir": "/root/.openclaw/agents/finance/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": {
          "name": "程二",
          "emoji": "💰"
        }
      }
    ]
  }
}
```

**当前 Agent 一览：**

| Agent ID | 名称 | 模型 | 工作空间 |
|----------|------|------|----------|
| `main` | 程一-秘书 | deepseek/deepseek-v4-pro | /root/.openclaw/workspace |
| `finance` | 程二-财务助手 | volcengine-plan/glm-5.1 | /root/.openclaw/workspace-finance |

**重要规则：**
- `defaults` 中定义所有 agent 共享的默认配置
- `list` 中的 agent 会继承 `defaults`，可单独覆盖（如 model）
- `model.primary` 格式必须是 `provider-id/model-id`
- 多 agent 场景：每个 agent 需要独立的 `workspace` 和 `agentDir`
- `identity.name` 和 `identity.emoji` 定义 agent 的显示身份

### gateway 配置（当前实际值）

```json
{
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "<自动生成>"
    },
    "port": 18789,
    "bind": "loopback",
    "trustedProxies": ["127.0.0.1/32"],
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    },
    "controlUi": {
      "allowInsecureAuth": true,
      "allowedOrigins": ["https://openclaw.huiyiyou.cloud"]
    },
    "nodes": {
      "denyCommands": [
        "camera.snap", "camera.clip", "screen.record",
        "contacts.add", "calendar.add", "reminders.add",
        "sms.send", "sms.search"
      ]
    }
  }
}
```

**启动命令：** `openclaw gateway --port 18789`

### channels 配置（当前实际值）

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_a932a2224f39dbce",
      "appSecret": "<secret>",
      "domain": "feishu",
      "connectionMode": "websocket",
      "requireMention": false,
      "streaming": true,
      "accounts": {
        "default": {},
        "finance": {
          "enabled": true,
          "name": "程二-财务助手",
          "appId": "cli_a9340f9ff178dbc6",
          "appSecret": "<secret>",
          "domain": "feishu",
          "connectionMode": "websocket",
          "requireMention": false,
          "streaming": true
        }
      }
    }
  }
}
```

**当前飞书账户：**

| 账户 | App ID | Agent |
|------|--------|-------|
| default | cli_a932a2224f39dbce | main（程一） |
| finance | cli_a9340f9ff178dbc6 | finance（程二） |

### bindings 配置（当前实际值）

```json
{
  "bindings": [
    {
      "type": "route",
      "agentId": "main",
      "match": { "channel": "feishu", "accountId": "default" }
    },
    {
      "type": "route",
      "agentId": "finance",
      "match": { "channel": "feishu", "accountId": "finance" }
    }
  ]
}
```

**规则：** 每个 binding 将一个通道账户路由到一个 agent，必须一一对应。

### plugins 配置（当前实际值）

```json
{
  "plugins": {
    "entries": {
      "volcengine": { "enabled": true },     // 火山引擎模型插件
      "deepseek": { "enabled": true },       // DeepSeek 模型插件
      "tavily": { "enabled": true },         // Tavily 搜索插件
      "openclaw-lark": { "enabled": true },  // 飞书通道插件
      "memory-core": { "enabled": true },    // 记忆核心插件（含 Dreaming）
      "memory-wiki": { "enabled": true }     // 记忆 Wiki 插件
    }
  }
}
```

### tools 配置（当前实际值）

```json
{
  "tools": {
    "profile": "full",             // 工具配置文件：full（全量工具）
    "web": {
      "search": {
        "provider": "tavily",
        "enabled": true
      }
    }
  }
}
```

### session / messages / commands 配置（当前实际值）

```json
{
  "session": {
    "dmScope": "per-channel-peer"       // DM 会话隔离策略
  },
  "messages": {
    "groupChat": {
      "visibleReplies": "message_tool"  // 群聊可见回复模式
    }
  },
  "commands": {
    "ownerAllowFrom": [
      "feishu:ou_647da8580f0ba18957271f288a5688ae"  // Owner 白名单
    ]
  }
}
```

### auth 配置（当前实际值）

```json
{
  "auth": {
    "profiles": {
      "volcengine:default": {
        "provider": "volcengine",
        "mode": "api_key"
      }
    }
  }
}
```

**说明：** 所有通过火山引擎代理的模型共用这一个 API Key 认证。

---

## 模型配置

模型信息定义在 openclaw.json 的 `models.providers` 节中（内联于主配置文件）。

### 当前已注册的 Provider

| Provider ID | API 类型 | 基础 URL | 说明 |
|---|---|---|---|
| `volcengine-plan` | openai-completions | `ark.cn-beijing.volces.com/api/coding/v3` | 火山引擎 Coding 计划（代理多模型） |
| `deepseek` | openai-completions | `api.deepseek.com` | DeepSeek 官方 API |

### volcengine-plan 下的模型列表

| 模型 ID | 上下文窗口 | 最大输出 | 输入类型 |
|---------|-----------|----------|---------|
| ark-code-latest | 256K | 32K | text, image |
| doubao-seed-code | 256K | 32K | text, image |
| doubao-seed-2.0-code | 256K | 128K | text, image |
| doubao-seed-2.0-pro | 256K | 128K | text, image |
| doubao-seed-2.0-lite | 256K | 128K | text, image |
| glm-5.1 | 200K | 128K | text |
| deepseek-v3.2 | 128K | 32K | text |
| minimax-latest | 200K | 128K | text |
| kimi-k2.6 | 256K | 32K | text, image |

### deepseek 下的模型列表

| 模型 ID | 上下文窗口 | 最大输出 | 输入类型 |
|---------|-----------|----------|---------|
| deepseek-v4-pro | 128K | 32K | text, image |

### 认证配置

```json
{
  "profiles": {
    "volcengine:default": {
      "type": "api_key",
      "provider": "volcengine",
      "key": "<api-key>"
    }
  }
}
```

**注意：** 当前所有通过火山引擎代理的模型共用同一 API Key，DeepSeek 直连使用独立 API Key。

---

## 常用操作指南

### 启动/停止 Gateway

```bash
# 启动
openclaw gateway --port 18789

# 强制启动（杀掉占用端口的进程）
openclaw gateway --force

# 后台运行（systemd）
systemctl --user restart openclaw-gateway

# 查看日志
journalctl --user -u openclaw-gateway -f

# 检查状态
openclaw doctor
```

### 添加新 Agent

1. 在 `openclaw.json` 的 `agents.list` 中添加新条目
2. 创建对应的目录：`agents/<id>/` 和 `workspace-<id>/`
3. 如有飞书多账户，在 `channels.feishu.accounts` 添加账户
4. 在 `bindings` 中添加路由

### 添加新模型

1. 在 `openclaw.json` 的 `models.providers` 下找到对应 provider，添加模型定义
2. 在 `openclaw.json` 的 `agents.defaults.models` 中添加 `provider/model-id: {}`
3. 如需设为全局默认，修改 `agents.defaults.model.primary`
4. 如需设为某 agent 专用，修改 `agents.list[].model`

### 切换 Agent 模型

修改 `agents.list` 中对应 agent 的 `model` 字段。示例：

```json
// 将 main agent 切换为豆包：在 agents.list 中将 main 的 model 改为
"model": "volcengine-plan/doubao-seed-2.0-pro"
```

### 更新 OpenClaw

```bash
npm update -g openclaw
openclaw setup --wizard    # 运行向导检查兼容性
```

---

## 官方文档参考

- **文档首页**：https://docs.openclaw.ai/
- **文档索引**：https://docs.openclaw.ai/llms.txt（完整的页面列表，查资料前先看这个）
- **配置参考**：https://docs.openclaw.ai/gateway/configuration-reference
- **Agent 配置**：https://docs.openclaw.ai/gateway/config-agents
- **通道配置**：https://docs.openclaw.ai/gateway/config-channels
- **工具和自定义 Provider**：https://docs.openclaw.ai/gateway/config-tools
- **飞书通道**：https://docs.openclaw.ai/channels/feishu
- **模型 Provider**：https://docs.openclaw.ai/providers/index
- **火山引擎 Provider**：https://docs.openclaw.ai/providers/volcengine
- **多 Agent 路由**：https://docs.openclaw.ai/concepts/multi-agent
- **Agent 工作空间**：https://docs.openclaw.ai/concepts/agent-workspace
- **会话管理**：https://docs.openclaw.ai/concepts/session
- **SOUL.md 人格指南**：https://docs.openclaw.ai/concepts/soul
- **记忆系统**：https://docs.openclaw.ai/concepts/memory
- **定时任务**：https://docs.openclaw.ai/automation/cron-jobs
- **CLI 命令参考**：https://docs.openclaw.ai/cli/index

---

## 注意事项

1. **修改 openclaw.json 后需要重启 gateway 才能生效**
2. **API Key 安全**：openclaw.json、auth-profiles.json 中都包含明文密钥，注意权限
3. **多 Agent 隔离**：每个 agent 的 workspace 是独立 git 仓库，互不影响
4. **飞书 ou_id 获取**：使用 `openclaw directory` 命令
5. **会话数据**：`agents/<id>/sessions/` 下的文件是二进制会话记录，不要手动编辑
6. **provider 命名**：引用模型时格式为 `provider-id/model-id`，如 `volcengine-plan/ark-code-latest`
7. **模型层级**：全局默认（`defaults.model.primary`）→ Agent 覆盖（`list[].model`）。
   例如：全局默认是 ark-code-latest，但 main agent 覆盖为 deepseek-v4-pro
