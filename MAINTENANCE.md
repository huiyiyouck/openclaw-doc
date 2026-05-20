# OpenClaw 本地维护手册

> 本手册面向 AI agent 和人工维护者，提供 OpenClaw 部署的架构、配置和运维参考。
> 官方文档：https://docs.openclaw.ai/ | 完整索引：https://docs.openclaw.ai/llms.txt
> 最后更新：2026-05-19

---

## 1. 系统架构概览

### 核心概念

| 概念 | 说明 |
|------|------|
| **Gateway** | 核心网关进程，监听端口 18789，连接通道与 AI agent |
| **Agent** | AI 代理实例，每个有独立的模型、workspace、记忆 |
| **Channel** | 消息通道（飞书等），将用户消息路由到 agent |
| **Plugin** | 扩展插件，提供工具和技能 |
| **Skill** | 技能模块（SKILL.md），教会 agent 使用特定工具 |
| **Binding** | 路由规则，将 channel+account 映射到 agent |

### 目录结构

```
/root/.openclaw/
├── openclaw.json              # 主配置文件（唯一配置源）
├── openclaw.json.last-good    # 最近一次正常配置备份
├── docs/                      # 文档（本手册）
├── doc/                       # 官方参考文档
│   └── openclaw-reference.md
├── agents/                    # Agent 定义目录
│   ├── main/
│   │   ├── agent/             # main agent 配置
│   │   │   ├── models.json    # 模型 provider 和模型列表
│   │   │   ├── auth-profiles.json  # API 密钥
│   │   │   └── auth-state.json     # 认证状态
│   │   └── sessions/          # 会话数据
│   └── finance/
│       ├── agent/             # finance agent 配置
│       │   ├── models.json    # 模型配置（继承 main 的）
│       │   └── auth-profiles.json
│       └── sessions/
├── workspace/                 # main agent (程一) workspace
│   ├── IDENTITY.md, SOUL.md, TOOLS.md, USER.md, AGENTS.md, ...
│   ├── memory/
│   └── .openclaw/
├── workspace-finance/         # finance agent (程二) workspace
│   ├── IDENTITY.md, SOUL.md, TOOLS.md, USER.md, AGENTS.md, ...
│   ├── memory/
│   └── .openclaw/
├── identity/                  # 设备密钥
│   └── device.json
├── logs/                      # 运行日志
├── plugins/                   # 插件数据
├── extensions/                # 外部插件安装目录
│   └── openclaw-lark/         # 飞书官方插件
├── plugin-skills/             # 插件附带的技能
├── completions/               # 补全数据
├── cron/                      # 定时任务
├── canvas/                    # Canvas 数据
├── tasks/                     # 任务数据
├── update-check.json          # 版本检查记录
└── .claude/                   # Claude Code 配置
    └── settings.local.json    # 权限白名单
```

---

## 2. 核心配置文件

### 2.1 openclaw.json（主配置）

路径：`/root/.openclaw/openclaw.json`

**当前完整配置：**

```json
{
  "agents": {
    "defaults": {
      "workspace": "/root/.openclaw/workspace",
      "models": { "volcengine-plan/ark-code-latest": {} },
      "model": { "primary": "volcengine-plan/ark-code-latest" }
    }
  },
  "gateway": {
    "mode": "local",
    "auth": { "mode": "token", "token": "<token>" },
    "port": 18789,
    "bind": "loopback"
  },
  "session": { "dmScope": "per-channel-peer" },
  "tools": {
    "profile": "coding",
    "web": { "search": { "provider": "tavily", "enabled": true } }
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "domain": "feishu",
      "connectionMode": "websocket",
      "requireMention": false,
      "streaming": true,
      "accounts": {
        "default": {
          "enabled": true, "name": "程一-秘书",
          "appId": "cli_a932a2224f39dbce", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket", "streaming": true
        },
        "finance": {
          "enabled": true, "name": "程二-财务助手",
          "appId": "cli_a9340f9ff178dbc6", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket", "streaming": true
        }
      }
    }
  },
  "bindings": [
    { "type": "route", "agentId": "main", "match": { "channel": "feishu", "accountId": "default" } },
    { "type": "route", "agentId": "finance", "match": { "channel": "feishu", "accountId": "finance" } }
  ],
  "plugins": {
    "entries": {
      "volcengine": { "enabled": true },
      "tavily": { "enabled": true },
      "openclaw-lark": { "enabled": true }
    }
  },
  "auth": {
    "profiles": {
      "volcengine:default": { "provider": "volcengine", "mode": "api_key" }
    }
  },
  "commands": {
    "ownerAllowFrom": ["feishu:ou_647da8580f0ba18957271f288a5688ae"]
  }
}
```

**主要字段说明：**

| 字段 | 说明 |
|------|------|
| `agents.defaults` | 默认 agent 配置（workspace、模型） |
| `gateway` | 网关设置（模式、端口、认证、绑定） |
| `channels.feishu` | 飞书通道配置（appId、WebSocket 连接） |
| `bindings` | 路由绑定（通道 → agent 映射） |
| `session.dmScope` | 会话作用域：per-channel-peer（每通道每对等方独立会话） |
| `tools.profile` | 工具配置集：coding |
| `tools.web.search` | 网络搜索：tavily |
| `plugins.entries` | 插件开关 |
| `auth.profiles` | 认证配置（API key 模式） |
| `commands.ownerAllowFrom` | 命令所有者（飞书配对用户） |

**Gateway 详细配置：**

| 参数 | 值 | 说明 |
|------|---|------|
| mode | local | 本地运行模式 |
| auth.mode | token | Token 认证 |
| port | 18789 | 监听端口 |
| bind | loopback | 仅监听本地回环 |
| tailscale.mode | off | Tailscale 关闭 |
| controlUi.allowInsecureAuth | true | 允许非安全认证 |
| nodes.denyCommands | [camera.*, screen.record, ...] | 节点禁止命令列表 |

### 2.2 agents/main/agent/models.json（模型配置）

定义了 4 个 provider，详见第 4 章。

### 2.3 agents/main/agent/auth-profiles.json（认证配置）

```json
{
  "profiles": {
    "xai:default": { "type": "api_key", "provider": "xai", "key": "<key>" },
    "volcengine:default": { "type": "api_key", "provider": "volcengine", "key": "<key>" }
  }
}
```

---

## 3. Agent 管理

### 3.1 当前 Agent

| ID | 名称 | Emoji | 默认模型 | Workspace | Agent Dir | 说明 |
|----|------|-------|---------|-----------|-----------|------|
| main | 程一 | 📋 | deepseek/deepseek-v4-pro | /root/.openclaw/workspace | /root/.openclaw/agents/main/agent | 一人公司秘书 |
| finance | 程二 | 💰 | volcengine-plan/glm-5.1 | /root/.openclaw/workspace-finance | /root/.openclaw/agents/finance/agent | 财务助手（消费记录） |

> 注：`agents.defaults.model.primary` 为 `volcengine-plan/ark-code-latest`，但两个 agent 各自在 `agents.list` 中覆盖了默认模型。

### 3.2 Workspace 文件

| 文件 | 说明 |
|------|------|
| IDENTITY.md | Agent 身份定义（名称、形象、风格） |
| SOUL.md | Agent 性格和行为准则 |
| TOOLS.md | 可用工具描述 |
| USER.md | 用户信息 |
| AGENTS.md | Agent 列表（多 agent 时使用） |
| BOOTSTRAP.md | 引导流程配置 |
| HEARTBEAT.md | 心跳/在线状态配置 |

### 3.3 路由绑定（Bindings）

| 通道 | 账户 | Agent | 说明 |
|------|------|-------|------|
| feishu | default | main (程一) | 程一飞书机器人 |
| feishu | finance | finance (程二) | 程二飞书机器人 |

---

## 4. 模型管理

### 4.1 Provider 总览

两个 agent 各有独立的 `models.json`，provider 数量不同：

| Provider | baseUrl | API 协议 | 程一 (main) | 程二 (finance) | 状态 |
|----------|---------|---------|:-----------:|:--------------:|------|
| volcengine-plan | https://ark.cn-beijing.volces.com/api/coding/v3 | openai-completions | 9 个 | 6 个 | 在用 |
| deepseek | https://api.deepseek.com | openai-completions | 1 个 | 无 | 在用（程一默认） |
| xai | https://api.x.ai/v1 | openai-responses | 14 个 | 14 个 | 待清理 |
| x-ai | https://api.x.ai/v1 | openai-responses | 14 个 | 14 个 | 待清理（xai 别名） |
| volcengine | https://ark.cn-beijing.volces.com/api/v3 | openai-completions | 5 个 | 5 个 | 待清理（未使用） |

> 注：xai/x-ai/volcengine 三个 provider 仍存在于 models.json 中但未实际使用，建议清理。

### 4.2 volcengine-plan 模型

**程一（main）— 9 个模型：**

| 模型 ID | 上下文 | 输出 | 输入 |
|---------|--------|------|------|
| ark-code-latest | 256K | 32K | text, image |
| doubao-seed-code | 256K | 32K | text, image |
| glm-5.1 | 200K | 128K | text |
| deepseek-v3.2 | 128K | 32K | text |
| doubao-seed-2.0-code | 256K | 128K | text, image |
| doubao-seed-2.0-pro | 256K | 128K | text, image |
| doubao-seed-2.0-lite | 256K | 128K | text, image |
| minimax-latest | 200K | 128K | text |
| kimi-k2.6 | 256K | 32K | text, image |

**程二（finance）— 6 个模型：**

| 模型 ID | 上下文 | 输出 | 输入 |
|---------|--------|------|------|
| ark-code-latest | 256K | 4K | text |
| doubao-seed-code | 256K | 4K | text |
| glm-4.7 | 200K | 4K | text |
| kimi-k2-thinking | 256K | 4K | text |
| kimi-k2.5 | 256K | 4K | text |
| doubao-seed-code-preview-251028 | 256K | 4K | text |

### 4.3 xai / x-ai 模型（Grok 系列）

两个 provider 模型列表相同，程一和程二共享。

| 模型 ID | 上下文 | 输出 | 推理 | 输入 |
|---------|--------|------|------|------|
| grok-3 | 131K | 8K | 否 | text |
| grok-3-fast | 131K | 8K | 否 | text |
| grok-3-mini | 131K | 8K | 是 | text |
| grok-3-mini-fast | 131K | 8K | 是 | text |
| grok-4.3 | 1M | 64K | 是 | text, image |
| grok-4 | 256K | 64K | 是 | text |
| grok-4-0709 | 256K | 64K | 否 | text |
| grok-4-fast | 2M | 30K | 是 | text, image |
| grok-4-fast-non-reasoning | 2M | 30K | 否 | text, image |
| grok-4-1-fast | 2M | 30K | 是 | text, image |
| grok-4-1-fast-non-reasoning | 2M | 30K | 否 | text, image |
| grok-4.20-beta-latest-reasoning | 2M | 30K | 是 | text, image |
| grok-4.20-beta-latest-non-reasoning | 2M | 30K | 否 | text, image |
| grok-code-fast-1 | 256K | 10K | 是 | text |

### 4.4 volcengine 模型（标准 API）

程一和程二共享，5 个模型：

| 模型 ID | 上下文 | 输出 | 输入 |
|---------|--------|------|------|
| doubao-seed-code-preview-251028 | 256K | 4K | text, image |
| doubao-seed-1-8-251228 | 256K | 4K | text, image |
| kimi-k2-5-260127 | 256K | 4K | text, image |
| glm-4-7-251222 | 200K | 4K | text, image |
| deepseek-v3-2-251201 | 128K | 4K | text, image |

### 4.5 deepseek 模型（官方 API）

仅程一（main）配置，1 个模型：

| 模型 ID | 上下文 | 输出 | 输入 |
|---------|--------|------|------|
| deepseek-v4-pro | 128K | 32K | text, image |

### 4.6 当前模型分配

| Agent | 默认模型 | 配置位置 |
|-------|---------|---------|
| main (程一) | deepseek/deepseek-v4-pro | agents.list[0].model 覆盖 defaults |
| finance (程二) | volcengine-plan/glm-5.1 | agents.list[1].model 覆盖 defaults |

> 注：`agents.defaults.model.primary` 为 `volcengine-plan/ark-code-latest`，但两个 agent 都在 `agents.list` 中指定了自己的默认模型。

### 4.7 模型切换方法

1. 编辑 `openclaw.json`，修改 `agents.defaults.model.primary`
2. 重启 gateway：`openclaw gateway restart --force`
3. 验证：`systemctl --user status openclaw-gateway`

---

## 5. Gateway 运维

### 5.1 服务信息

| 参数 | 值 |
|------|---|
| 服务名 | openclaw-gateway.service |
| 服务类型 | systemd user service |
| 服务版本 | 2026.5.18 |
| 安装版本 | 2026.5.18 |
| 端口 | 18789 |
| 执行命令 | node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789 |
| 重启策略 | Restart=always, RestartSec=5 |
| 配置文件 | ~/.config/systemd/user/openclaw-gateway.service |

### 5.2 常用命令速查

```bash
# 查看状态
systemctl --user status openclaw-gateway

# 重启
systemctl --user restart openclaw-gateway

# 停止
systemctl --user stop openclaw-gateway

# 查看实时日志
journalctl --user -u openclaw-gateway -f

# 查看最近 100 行日志
journalctl --user -u openclaw-gateway -n 100

# 健康检查
openclaw doctor

# 重新加载 systemd 配置（修改 service 文件后）
systemctl --user daemon-reload
```

### 5.3 环境变量

服务通过 Environment 注入：
- `HOME=/root`
- `OPENCLAW_GATEWAY_PORT=18789`
- `OPENCLAW_SYSTEMD_UNIT=openclaw-gateway.service`
- `OPENCLAW_SERVICE_VERSION=2026.5.18`

---

## 6. 飞书通道配置

### 6.1 账号信息

飞书通道配置为多账户模式，两个独立的飞书机器人应用：

| 参数 | 程一 (default) | 程二 (finance) |
|------|----------------|----------------|
| App ID | cli_a932a2224f39dbce | cli_a9340f9ff178dbc6 |
| 域名 | feishu | feishu |
| 连接方式 | WebSocket | WebSocket |
| requireMention | false | false |
| streaming | true | true |
| 绑定 Agent | main | finance |

### 6.2 配对和权限

- 使用飞书配对码模式（非 allowFrom 模式）
- 已配对用户：`ou_647da8580f0ba18957271f288a5688ae`
- 命令所有者：`commands.ownerAllowFrom: ["feishu:ou_647da8580f0ba18957271f288a5688ae"]`

### 6.3 路由绑定

- 飞书 default 账户 → main agent（程一）
- 飞书 finance 账户 → finance agent（程二）

### 6.4 插件

使用飞书官方插件 `openclaw-lark`（非内置 feishu 插件），安装路径：
`/root/.openclaw/extensions/openclaw-lark/`

安装/更新命令：`npx -y @larksuite/openclaw-lark install`

### 6.5 注意事项

- 飞书开放平台中，应用的订阅方式必须设为"使用长连接接收事件/回调"
- 内置 `feishu` 插件已禁用并从 plugins.entries 中移除，避免与 `openclaw-lark` 冲突

---

## 7. 插件系统

### 7.1 已启用插件

| 插件 | 说明 |
|------|------|
| volcengine | 火山引擎集成（模型 provider） |
| tavily | 网络搜索（`tools.web.search.provider`） |
| openclaw-lark | 飞书官方插件（通道 + 工具） |
| memory-core | 记忆核心（含 dreaming 梦境整理） |
| memory-wiki | 记忆 wiki 系统 |

### 7.2 认证配置

通过 `auth.profiles` 管理 API 密钥：
- `volcengine:default` — 火山引擎 API key
- `xai:default` — xAI API key（在 auth-profiles.json 中）

---

## 8. Skills 系统

openclaw-lark 插件提供的 skills（安装在 `plugin-skills/` 目录）：

| Skill | 说明 |
|-------|------|
| feishu-bitable | 飞书多维表格的增删改查 |
| feishu-calendar | 飞书日历管理 |
| feishu-fetch-doc | 获取飞书文档内容 |
| feishu-create-doc | 创建飞书文档 |
| feishu-update-doc | 更新飞书文档 |
| feishu-im-read | 读取飞书消息 |
| feishu-task | 飞书任务管理 |
| feishu-channel-rules | 飞书通道规则 |
| feishu-troubleshoot | 飞书排障 |

### 加载优先级（从高到低）

1. Workspace skills（`/skills`）
2. Project agent skills（`/.agents/skills`）
3. Personal agent skills（`~/.agents/skills`）
4. Managed/local skills（`~/.openclaw/skills`）
5. Bundled skills（随安装包发布）
6. Extra skill folders（`skills.load.extraDirs`）

### Skill 管理命令

```bash
# 安装 skill
openclaw skills install <skill-name>

# 更新所有已安装 skills
openclaw skills update --all

# 查看 ClawHub 上的 skills
clawhub search <query>
```

---

## 9. 记忆系统

### 9.1 已启用插件

| 插件 | 说明 |
|------|------|
| memory-core | 记忆核心，含 dreaming（梦境整理）功能 |
| memory-wiki | 记忆 wiki 系统 |

### 9.2 Dreaming 配置

```json
{
  "dreaming": {
    "enabled": true,
    "frequency": "0 3 * * *",
    "timezone": "Asia/Shanghai"
  }
}
```

每天凌晨 3:00（Asia/Shanghai）自动触发，生成 light（浅层）和 rem（深层）梦境，整理对话记忆。

### 9.3 数据存储

| 文件 | 说明 |
|------|------|
| memory/main.sqlite | 程一的记忆数据库 |
| memory/finance.sqlite | 程二的记忆数据库 |
| workspace/memory/ | 程一的 workspace 记忆文件 |
| workspace-finance/memory/ | 程二的 workspace 记忆文件 |

---

## 10. 设备和安全

### 10.1 设备密钥

`/root/.openclaw/identity/device.json` — 本机设备标识。

### 10.2 Gateway 认证

- 模式：token
- 绑定：loopback（仅本地访问）
- Tailscale：关闭

### 10.3 节点安全

Gateway 禁止远程节点执行以下命令：
- camera.snap, camera.clip
- screen.record
- contacts.add, calendar.add, reminders.add
- sms.send, sms.search

---

## 11. 常见运维操作

### 11.1 配置修改标准流程

```bash
# 1. 备份当前配置
cp openclaw.json openclaw.json.backup.$(date +%Y%m%d%H%M%S)

# 2. 编辑配置（使用编辑器或 AI agent）

# 3. 验证 JSON 格式
python3 -c "import json; json.load(open('openclaw.json')); print('JSON OK')"

# 4. 重启 gateway
systemctl --user restart openclaw-gateway

# 5. 检查启动是否正常
systemctl --user status openclaw-gateway
journalctl --user -u openclaw-gateway -n 30
```

> 注意：openclaw 会自动保存 `openclaw.json.last-good` 作为最近一次正常配置的备份。

### 11.2 日志查看

```bash
# 查看日志文件（主要日志位置）
ls /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 实时跟踪
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 搜索错误
grep '"logLevelName":"ERROR"' /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 按关键字搜索（如飞书消息处理）
grep "feishu\[default\]" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -50

# systemd 日志（启动/停止事件）
journalctl --user -u openclaw-gateway -n 100
```

### 11.3 健康检查

```bash
openclaw doctor
```

### 11.4 备份关键文件

```bash
tar czf /root/openclaw-backup-$(date +%Y%m%d).tar.gz \
  -C /root \
  .openclaw/openclaw.json \
  .openclaw/agents/ \
  .openclaw/workspace/ \
  .openclaw/workspace-finance/ \
  .openclaw/identity/ \
  .openclaw/doc/ \
  .config/systemd/user/openclaw-gateway.service
```

### 11.5 新增 Agent

使用 CLI 标准流程：

```bash
# 1. 添加 agent
openclaw agents add <id> --workspace /root/.openclaw/workspace-<id> --non-interactive

# 2. 设置身份
openclaw agents set-identity --agent <id> --name "显示名" --emoji "📋"

# 3. 如需飞书通道，在 openclaw.json 的 channels.feishu.accounts 中添加账户

# 4. 绑定路由
openclaw agents bind --agent <id> --bind feishu:<accountId>

# 5. 创建 workspace 文件（IDENTITY.md、SOUL.md 等）
# 6. 按标准流程重启 gateway
```

### 11.6 新增飞书通道账户

1. 在 `openclaw.json` 的 `channels.feishu.accounts` 中添加新账户
2. 配置 enabled、name、appId、appSecret、domain、connectionMode、streaming
3. 使用 `openclaw agents bind` 绑定到对应 agent
4. 按标准流程重启 gateway

### 11.7 版本升级

```bash
npm install -g openclaw@latest
openclaw doctor                    # 检查兼容性
systemctl --user restart openclaw-gateway
```

---

## 12. 排障指南

### 12.1 Gateway 无法启动

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| 服务启动后立即退出 | JSON 配置语法错误 | `python3 -c "import json; json.load(open('openclaw.json'))"` |
| 端口被占用 | 18789 端口冲突 | `netstat -tuln \| grep 18789` 检查并 kill |
| Restart 循环 | 依赖问题 | `journalctl --user -u openclaw-gateway -n 50` |

### 12.2 模型调用失败

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| 429 Too Many Requests | API 限流 | 等待或调整调用频率 |
| 403 Forbidden | API Key 无效 | 检查 models.json 和 auth-profiles.json 中的 key |
| Timeout | 网络问题 | 检查网络连接 |
| 上下文超限 | 输入超过 contextWindow | 减少输入或切换更大窗口模型 |

### 12.3 通道连接问题

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| 消息不响应 | 通道未配置或连接断开 | 检查 channels 配置，重启 gateway |
| 权限错误 | appId/appSecret 错误 | 验证凭证，重新获取 |

### 12.4 飞书卡片卡在"回复中"

**症状：** 飞书上给 agent 发消息后，卡片一直显示"回复中"，后续消息全部不响应。

**诊断方法：**

```bash
# 查看是否有卡在 streaming 的卡片（无 completed 日志）
grep "phase transition.*streaming.*completed" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -5

# 查看是否有 queued 的消息
grep "queued" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -5
```

**根因：** CardKit 卡片在 streaming 模式下更新未正常完成（`onIdle` 回调未触发），导致卡片永远不会 transition 到 completed 状态。后续所有新消息都被标记为 queued，无法处理。

**解决方案：**

```bash
# 重启 gateway 清除卡住的 session 状态
openclaw gateway restart --force
```

> 已知触发条件：回复内容过长时更容易出现（如 seq > 200 次 cardElement 更新）。

---

## 附录：官方文档索引

### 核心文档
- 官方文档首页：https://docs.openclaw.ai/
- 完整页面索引：https://docs.openclaw.ai/llms.txt
- 快速入门：https://docs.openclaw.ai/getting-started

### 架构与概念
- 架构概览：https://docs.openclaw.ai/concepts/architecture
- Workspace：https://docs.openclaw.ai/concepts/workspace
- Session：https://docs.openclaw.ai/concepts/session
- Presence：https://docs.openclaw.ai/concepts/presence
- Memory：https://docs.openclaw.ai/concepts/memory

### Agent 配置
- Agent 配置：https://docs.openclaw.ai/agents/configuration
- ACP Agents：https://docs.openclaw.ai/tools/acp-agents
- Sub-agents：https://docs.openclaw.ai/tools/subagents

### 模型
- 模型配置：https://docs.openclaw.ai/models
- Providers：https://docs.openclaw.ai/models/providers

### Gateway
- 配置参考：https://docs.openclaw.ai/gateway/configuration-reference
- 健康检查：https://docs.openclaw.ai/gateway/health
- 安全：https://docs.openclaw.ai/gateway/security
- 沙箱：https://docs.openclaw.ai/gateway/sandboxing
- 排障：https://docs.openclaw.ai/gateway/troubleshooting

### 通道
- 通道总览：https://docs.openclaw.ai/channels
- 飞书：https://docs.openclaw.ai/channels/feishu
- 路由：https://docs.openclaw.ai/channels/channel-routing

### 工具与技能
- 工具总览：https://docs.openclaw.ai/tools
- Skills：https://docs.openclaw.ai/tools/skills
- Skills 配置：https://docs.openclaw.ai/tools/skills-config
- 创建 Skills：https://docs.openclaw.ai/tools/creating-skills
- ClawHub：https://docs.openclaw.ai/tools/clawhub
- 插件：https://docs.openclaw.ai/tools/plugin

### 多 Agent
- 多 Agent 沙箱和工具：https://docs.openclaw.ai/tools/multi-agent-sandbox-tools
