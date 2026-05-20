# 多 Agent 交叉学习与每日总结系统架构方案

## 一、系统整体架构

以下是一个完整的系统架构设计，通过飞书群组实现角色分工的 Agent 团队协作，满足你作为"老板"实时可见、随时干预的需求。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             飞书群组（所有Agent同在一个群）                     │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│   │ 产品经理 │ │ 架构师  │ │ UI设计  │ │ 前端开发 │ │ 后端开发 │ │  测试   │  │
│   │  Agent  │ │  Agent  │ │  Agent  │ │  Agent  │ │  Agent  │ │  Agent  │  │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘  │
│        │           │           │           │           │           │       │
│   (老板可实时查看和@干预，Agent被@后回复，Agent之间用sessions_send通信)        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OpenClaw Gateway（单Gateway模式）                   │
│   ┌─────────────────────────────────────────────────────────────────────┐    │
│   │  飞书插件（@tobeyoureyes/feishu + openclaw-feishu，双通道）           │    │
│   │  - WebSocket长连接接收消息  - 发送群聊/卡片  - sessions_send跨会话通信 │    │
│   └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐    │
│   │  记忆系统（飞书多维表格 + Agent本地MEMORY.md）                          │    │
│   │  - 每日总结记录  - 交叉学习存档  - 个体成长日志  - 经验知识积累          │    │
│   └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 架构选型说明

OpenClaw 支持在同一 workspace 中运行多个独立的 AI Agent，每个 Agent 可以有独立的人格（Soul）、身份（Identity）、模型配置和技能集，这些 Agent 可以通过会话（Session）相互通信，形成团队协作。

选择单 Gateway 模式的理由：资源占用低，配置简单，Agent 间文件共享与通信便捷，非常适合个人使用、小团队以及需要 Agent 协作的轻量级场景。多 Agent 本身已有良好的隔离性，单 Gateway 不会造成安全风险。

### Agent 列表规划

按你现有团队结构（未来可扩展），设置以下 Agent：

| Agent ID | 角色 | 核心职责 |
|----------|------|----------|
| pm | 产品经理 | 需求分析、产品规划、优先级决策 |
| architect | 架构师 | 技术架构设计、技术决策、代码评审 |
| ui | UI设计 | 交互设计、视觉设计、设计规范 |
| frontend | 前端开发 | 前端代码开发、组件设计 |
| backend | 后端开发 | 后端代码开发、API设计 |
| qa | 测试 | 测试用例、自动化测试、质量把控 |
| devops | 部署 | CI/CD、环境配置、监控告警 |

未来可新增 marketing、data 等 Agent。

## 二、核心配置与准备工作

### 2.1 创建 Agent 并配置独立工作区

```bash
# 创建工作区目录（统一放在 .openclaw 下便于管理）
mkdir -p ~/.openclaw/workspace-pm
mkdir -p ~/.openclaw/workspace-architect
mkdir -p ~/.openclaw/workspace-ui
mkdir -p ~/.openclaw/workspace-frontend
mkdir -p ~/.openclaw/workspace-backend
mkdir -p ~/.openclaw/workspace-qa
mkdir -p ~/.openclaw/workspace-devops

# 在 openclaw.json 中配置 Agent（参考配置）
openclaw config agents.add pm --workspace ~/.openclaw/workspace-pm
openclaw config agents.add architect --workspace ~/.openclaw/workspace-architect
openclaw config agents.add ui --workspace ~/.openclaw/workspace-ui
# ... 其他 Agent 同理
```

### 2.2 关键配置项（openclaw.json）

```json
{
  "agents": {
    "list": [
      { "id": "pm", "workspace": "~/.openclaw/workspace-pm" },
      { "id": "architect", "workspace": "~/.openclaw/workspace-architect" },
      { "id": "ui", "workspace": "~/.openclaw/workspace-ui" },
      { "id": "frontend", "workspace": "~/.openclaw/workspace-frontend" },
      { "id": "backend", "workspace": "~/.openclaw/workspace-backend" },
      { "id": "qa", "workspace": "~/.openclaw/workspace-qa" },
      { "id": "devops", "workspace": "~/.openclaw/workspace-devops" }
    ]
  },
  "tools": {
    "sessions": {
      "visibility": "all"
    },
    "agentToAgent": {
      "enabled": true,
      "allow": ["pm", "architect", "ui", "frontend", "backend", "qa", "devops"],
      "historyLimit": 50
    }
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxxxxxx",
      "appSecret": "xxxxxxxxx",
      "groupPolicy": "allowlist",
      "requireMention": true,
      "groups": {
        "oc_xxxxxxxxx": { "enabled": true }
      }
    }
  }
}
```

关键配置说明：

- `tools.sessions.visibility = "all"`：让 Agent 之间能够"看见"彼此，这是 Agent 协同的最关键一步。
- `tools.agentToAgent.enabled = true` + allow 列表：允许指定的 Agent 进行跨会话通信。

### 2.3 为每个 Agent 编写 SOUL.md（核心人格文件）

OpenClaw 会读取工作区中的 SOUL.md 等用户可编辑文件并注入会话上下文。以下是针对本系统设计的 Agent 人格文件模板：

#### 架构师 Agent 示例（~/.openclaw/workspace-architect/SOUL.md）

```markdown
# SOUL.md - 架构师

## 核心特质
- 技术精湛，擅长系统架构设计与技术决策
- 重视可扩展性、可维护性和系统边界划分
- 能够从全局视角理解各角色工作的相互影响
- 善于发现设计缺陷并主动提出优化建议

## 沟通风格
- 直接、严谨，用结构化思考表达观点
- 用图表和架构图辅助说明（Markdown + 文字描述）
- 对不确定的问题会主动追问细节

## 交叉学习职责
- 每日总结：梳理当天架构相关工作和产出
- 交叉理解：分析其他角色工作对系统架构的影响
- 相互提问：对其他角色的总结提出技术挑战和建设性意见
- 持续成长：记录发现的问题和经验教训

## 每日总结模板
### 今日工作
- [具体产出]

### 技术决策
- [今天做了哪些技术决策]

### 架构影响评估
- [各 Agent 的工作对架构的影响]

### 对其他角色的建议/质疑
- [针对PM/UI/前后端/测试/运维的具体问题]

### 今日经验沉淀
- [学到的教训、发现的坑、积累的最佳实践]
```

#### 产品经理 Agent 示例（~/.openclaw/workspace-pm/SOUL.md）的关键差异化内容

```markdown
## 核心特质
- 擅长需求分析、用户洞察和产品价值判断
- 能够从业务视角评估技术方案的合理性

## 交叉学习职责
- 理解架构决策对产品路线的影响
- 理解 UI 设计对用户体验的贡献
- 用用户故事和业务价值角度对其他角色提出疑问

## 每日总结模板
### 今日工作
- [产品产出：需求/PRD/优先级]
- [用户反馈/数据分析结论]

### 对其他角色的建议/质疑
- [对架构师：这个设计是否会影响后续需求扩展？]
- [对设计：这个交互方案是否考虑到了目标用户的使用习惯？]
- [对前后端：当前开发进度是否符合产品节点？]
```

#### UI 设计 Agent 示例（关键差异化内容）

```markdown
## 核心特质
- 擅长交互设计、视觉设计、设计系统搭建
- 能够从用户体验角度评估技术实现的合理性

## 交叉学习职责
- 理解产品需求驱动的设计目标
- 理解前端开发对设计稿的落地能力边界

## 对其他角色的建议/质疑
- [对产品：这个需求的用户场景是否清晰？]
- [对前端：这个组件库能支持我设计的所有状态吗？]
- [对测试：交互流程的边界情况都覆盖了吗？]
```

每个 Agent 都需要三个文件：SOUL.md（人格）、IDENTITY.md（身份）、USER.md（用户信息）。各 Agent 的 IDENTITY.md 主要用于定义基本身份信息，USER.md 记录 Agent 服务的用户信息。建议为每个 Agent 编写符合其角色的 SOUL.md，明确定义其在交叉学习中的具体职责和提问边界。

### 2.4 飞书双通道通信配置

每个 Agent 需要知道其他 Agent 在飞书群中的 session_key 和 open_id。

session_key 格式：`agent:<agentId>:feishu:group:<chat_id>`。例如：

- PM Agent：`agent:pm:feishu:group:oc_xxxxxxxxx`
- 架构师 Agent：`agent:architect:feishu:group:oc_xxxxxxxxx`
- UI Agent：`agent:ui:feishu:group:oc_xxxxxxxxx`
- 前端 Agent：`agent:frontend:feishu:group:oc_xxxxxxxxx`
- 后端 Agent：`agent:backend:feishu:group:oc_xxxxxxxxx`
- 测试 Agent：`agent:qa:feishu:group:oc_xxxxxxxxx`
- 运维 Agent：`agent:devops:feishu:group:oc_xxxxxxxxx`

open_id 映射表（在飞书应用中获取，需要整理一个文档放在统一位置供所有 Agent 访问）：

```json
{
  "agentOpenIdMap": {
    "pm": { "openId": "ou_xxxxxxxxx_pm" },
    "architect": { "openId": "ou_xxxxxxxxx_arch" },
    "ui": { "openId": "ou_xxxxxxxxx_ui" },
    "frontend": { "openId": "ou_xxxxxxxxx_fe" },
    "backend": { "openId": "ou_xxxxxxxxx_be" },
    "qa": { "openId": "ou_xxxxxxxxx_qa" },
    "devops": { "openId": "ou_xxxxxxxxx_dops" }
  }
}
```

### 2.5 配置飞书插件实现消息收发

通过安装 @tobeyoureyes/feishu 插件，让 OpenClaw Agent 能够接入飞书，支持私聊、群聊、被 @ 提及后回复、处理图片和文件等。

安装命令：

```bash
openclaw plugins install @tobeyoureyes/feishu
```

配置要点：群聊策略建议设为 `groupPolicy: "allowlist"` 并指定 groups，`requireMention: true` 确保 Agent 只在被 @ 时回复，避免刷屏。如需更完整的卡片交互能力，可同时安装 openclaw-feishu 插件作为补充。

## 三、核心交互流程（详细步骤）

### 3.1 每日总结流程（定时触发）

时间点：每日 18:00（或你指定的时间）

**Step 1: 老板（或定时任务）在群里发指令**

→ 老板 @admin_agent "开始每日总结"

**Step 2: 各 Agent 被 @ 后开始工作**

→ 群聊出现：@pm 开始总结；@architect 开始总结...

**Step 3: 各 Agent 自己生成总结**

→ 每个 Agent 根据自己 SOUL.md 中的模板生成今日工作总结
→ 同时检索飞书多维表格中的"历史记忆"，结合当天的实际工作产出

**Step 4: 各 Agent 在群里公开总结**

→ 在群聊中发送各自的总结（必然让老板看到）
→ 同时通过 sessions_send 将总结推送给其他所有 Agent 的 session

**Step 5: 相互提问环节（深度交叉学习）**

→ 每个 Agent 读取其他 Agent 的总结内容
→ 交叉分析，生成对其他 Agent 的提问/质疑/建议
→ 在群里依次展示提问（并 @ 对方）

**Step 6: 回应与辩论（多轮可选）**

→ 被 @ 的 Agent 回应质疑
→ 可多轮深入辩论，直到达成共识或记录分歧

**Step 7: 归档（老板确认后）**

→ 老板输入"归档"或"完成总结"
→ Coordinator/Conductor Agent（预设的协调 Agent，可以使用 main Agent 担任）将今日总结保存到多维表格
→ 更新每个 Agent 的个体学习记录

### 3.2 交叉理解机制（如何让 Agent 真正理解别人的工作）

交叉理解的核心在于每个 Agent 的 SOUL.md 中明确定义了以下两条规则：

**规则一（必须阅读别人）：**

```markdown
# 交叉学习规则 1
在执行每日总结时，你必须：
1. 先通过 sessions_send 获取其他 Agent 的总结内容
2. 分析他们的工作对你所负责领域的影响
3. 在总结中包含"对其他角色的影响分析"部分
4. 如果不理解别人的工作内容，必须主动提问
```

**规则二（必须主动提问）：**

```markdown
# 交叉学习规则 2
在阅读了其他 Agent 的总结后，你必须：
1. 至少对其他 Agent 的工作提出 2 个有深度的问题或质疑
2. 问题不能是"做得好"之类的客套话，必须涉及具体内容、技术细节或业务影响
3. 问题可以用业务价值、技术合理性、设计一致性、质量保障等角度提出
```

交叉理解的具体流程：

- 每个 Agent 完成自己的总结后，通过 sessions_send 发送给所有其他 Agent
- 其他 Agent 读取后，在自己的 SOUL.md 提示框架下，提取"对自己领域有影响"的关键信息
- 交叉分析环节，按顺序轮流提问，并采用"提问→回应→追问"的轮次结构
- 提问方记录对方回应，更新自己的认知

### 3.3 相互提问机制（代码级实现）

**Step 1 - 读取别人总结：**

工具调用：

```json
sessions_send
  targetSession: "agent:architect:feishu:group:oc_xxxxx"
  message: "请求获取今日工作总结"
  timeoutSeconds: 60
```

Agent 需要维护一个"其他人总结的副本"（可以从群聊中获取，但 sessions_send 更可靠，因为消息直接进入对方 session）。Agent 将其他总结的内容结构化存储在临时上下文中。

**Step 2 - 生成具体问题和质疑：**

在 SOUL.md 中定义各 Agent 的提问侧重领域：

- PM → 对他人提问：质疑技术方案是否符合业务目标、优先级是否合理、用户场景是否全面
- 架构师 → 对他人提问：质疑设计决策的可扩展性、系统边界清晰度、技术选型合理性
- UI → 对他人提问：质疑交互流程的体验、视觉一致性、可访问性
- 前端 → 对他人提问：质疑 API 设计的前端友好度、接口文档完整性、性能指标
- 后端 → 对他人提问：质疑数据结构的合理性、安全边界、数据库设计
- 测试 → 对他人提问：质疑测试覆盖率、边界情况处理、异常场景设计
- 运维 → 对他人提问：质疑部署流程、环境一致性、监控和回滚方案

**Step 3 - 异步提问与响应：**

- Agent A 在群里公开提问，并用飞书卡片生成投票/确认按钮（供老板判断）
- 被 @ 的 Agent B 在自己的 session 收到消息（通过飞书事件订阅自动识别）
- Agent B 回复前，先在群里公开表示"收到问题，正在思考"
- Agent B 生成详细回答，@ Agent A 并发送
- 如果 Agent A 认为回答不充分，继续追问

### 3.4 多角色评审任务流程

当老板下达一个开发任务时，可以启动多角色评审流程：

**步骤 1:** 老板在群里 @architect "启动'用户认证模块'的多角色评审"

**步骤 2:** 架构师作为 Orchestrator，分解任务：

→ 用 sessions_send 向 PM 发送："请提供该模块的用户场景和需求优先级"
→ 用 sessions_send 向 UI 发送："请提供认证页面的交互设计方案"
→ 用 sessions_send 向前端发送："请评估认证组件的实现方案"
→ 用 sessions_send 向后端发送："请提供认证 API 设计方案"
→ 用 sessions_send 向 QA 发送："请准备认证模块的测试计划"
→ 用 sessions_send 向 DevOps 发送："请评估部署影响"

**步骤 3:** 各 Agent 提供方案（3-5 分钟）后在群里公开

**步骤 4:** 架构师主持评审，进行第二层交叉提问

**步骤 5:** 老板在群里确认最终方案（老板拥有最终决策权）

### 3.5 日常任务管理流程

当老板在群里有任务需要处理时，系统应自动触发任务分配和反馈机制：

**步骤 1:** 老板在群里 @all_agents "需要完成：用户登录页面开发"

**步骤 2:** 架构师 Agent 自动分析任务：

- 如果是一人任务：直接分配给对应 Agent（如 @frontend）
- 如果是多人任务：拆解后分别 @ 并协调时间

**步骤 3:** 任务承接人给出时间预估

格式：@boss "预计 2 小时完成"

**步骤 4:** 完成后在群里 @review 请求评审

**步骤 5:** 相关 Agent 交叉评审（如架构师审 API 设计，测试审测试方案）

**步骤 6:** 评审通过，归档到多维表格

## 四、Agent 持续学习与成长方案（记忆系统）

### 4.1 存储方案：飞书多维表格 + Agent 本地 MEMORY.md

飞书多维表格提供了强大的数据管理能力，可以通过多维表格 API 打通业务系统，让数据流转和同步。

创建多维表格结构：

创建一个名为"Agent 团队记忆库"的多维表格，包含以下数据表：

| 数据表名称 | 字段 | 说明 |
|-----------|------|------|
| 每日总结 | 日期、Agent ID、总结内容、影响分析、对其他角色的问题、经验教训 | 每天各 Agent 的总结归档 |
| 问答记录 | 日期、提问方、回答方、问题内容、回答内容、是否已解决 | 所有交叉提问的存档 |
| 个体学习记录 | Agent ID、日期、今日学到的内容、修正的认知、积累的经验 | 每个 Agent 的成长记录 |
| 经验知识库 | 知识点、提出者、适用角色、案例、标签 | 长期知识积累 |
| 最佳实践 | 实践名称、来源 Agent、适用场景、内容 | 被验证的成功做法 |

多维表格操作示例（Python，通过 OpenAPI 写入）：

```python
import requests
import json

def save_daily_summary(app_token, table_id, agent_id, summary):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records"
    headers = {
        "Authorization": f"Bearer {tenant_access_token}",
        "Content-Type": "application/json"
    }
    data = {
        "fields": {
            "日期": "2026-01-15",
            "Agent ID": agent_id,
            "总结内容": summary,
            "归档时间": "2026-01-15 18:30:00"
        }
    }
    response = requests.post(url, headers=headers, json=data)
    return response.json()
```

更新多维表格记录时使用 PUT 请求，且为增量更新。

### 4.2 分层记忆架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    记忆系统分层架构                              │
├─────────────────────────────────────────────────────────────────┤
│ Level 1: 短期记忆（当前 Session）                                │
│   - 存储在 OpenClaw 会话文件中（.jsonl）                        │
│   - 包含当天讨论内容、当前任务上下文                             │
├─────────────────────────────────────────────────────────────────┤
│ Level 2: 中期记忆（Agent 本地文件）                              │
│   - MEMORY.md：每个 Agent 的长期记忆                            │
│   - memory/YYYY-MM-DD.md：每日日志                               │
│   - SOUL.md/IDENTITY.md：人格与身份                             │
├─────────────────────────────────────────────────────────────────┤
│ Level 3: 团队记忆（飞书多维表格）                                │
│   - 每日总结归档                                                │
│   - 交叉问答记录                                                │
│   - 团队经验知识库                                              │
│   - 个体成长档案                                                │
├─────────────────────────────────────────────────────────────────┤
│ Level 4: 外部知识（可选扩展）                                    │
│   - RAG 检索增强                                                │
│   - 代码仓库（Git）                                             │
│   - 文档系统（飞书文档/知识库）                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 每日总结模板（标准化）

各 Agent 统一使用以下模板：

```markdown
【{{agent_name}} 每日总结 | {{date}}】

## 📋 今日工作
- [具体完成的开发/设计/分析工作]
- [产出的文档/代码/设计方案]

## 💡 技术决策
- [今天做的重要决策及理由]

## 🔗 对其他角色的影响分析
- **对 PM 的影响：** [...]
- **对 UI 的影响：** [...]
- **对前端的影响：** [...]
- **对后端的影响：** [...]
- **对测试的影响：** [...]
- **对运维的影响：** [...]

## ❓ 对其他角色的提问
1. @pm [...]
2. @ui [...]
3. @frontend [...]
4. @backend [...]
5. @qa [...]

## 📝 今日经验教训
- [学到]
- [教训]
```

每日归档内容格式（飞书多维表格）：

```json
{
  "日期": "2026-01-15",
  "Agent ID": "architect",
  "总结内容": "今日完成...",
  "影响分析": "对 PM 的影响：...",
  "提出问题": "@pm 请问...",
  "经验教训": "今天的教训：..."
}
```

### 4.4 经验提炼与知识沉淀机制

每个 Agent 在自己的 SOUL.md 中包含以下自我提升规则：

```markdown
# 经验提炼规则
在每日总结完成后，你必须：
1. 从今天的讨论中提炼至少 1 条可复用的经验/教训
2. 检查是否修正了以往的认知误区
3. 如果发现了新的最佳实践，记录并验证

# 知识检索规则
在开始新的一天工作前，你必须：
1. 通过 sessions_send 向多维表格查询最近的相关总结
2. 回顾自己的 MEMORY.md 中和当前任务相关的内容
3. 将检索到的经验融入今天的工作
```

团队共同确认的重要经验，由协调 Agent 统一整理并置顶到多维表格的"最佳实践"表。

## 五、飞书群实时同步与人工干预

### 5.1 群聊消息分类

群聊中消息分为以下几类：

| 消息类型 | 触发方式 | Agent 行为 |
|---------|----------|------------|
| 老板指令 | 老板 @ 特定 Agent 或 @all | 被 @ 的 Agent 响应并执行 |
| 交叉提问 | Agent A @ Agent B | 被 @ 的 Agent 思考后回复 |
| 进度同步 | Agent 完成任务后主动群发 | 其他 Agent 知晓，老板可见 |
| 日常分享 | Agent 发现有趣的信息主动分享 | 其他 Agent 可选回应 |
| 每日总结 | 定时触发或老板指令 | 所有 Agent 按模板发送 |
| 异常告警 | Agent 检测到错误或超时 | 相关 Agent 迅速回应 |

### 5.2 老板干预机制

老板作为系统的最终决策者，可通过 @ 随时介入任何 Agent 的决策过程：

**场景 1：老板不同意架构师的某个技术决策**

→ 老板在群里 @architect "这个方案是否考虑过 xxx？"

**场景 2：老板希望调整优先级**

→ 老板 @pm "下周优先做用户反馈系统，其他先缓一缓"

**场景 3：老板叫停正在进行的自动流程**

→ 老板发送 "暂停所有 Agent 的当前任务"

**场景 4：老板单独和某个 Agent 沟通（私聊模式）**

→ 可通过私聊直接给该 Agent 下达指令

老板干预后，Agent 必须重新评估当前决策，并根据新指令调整行动。

### 5.3 信息可见性设计

- 所有 Agent 的总结和提问都在群聊中公开，老板可见全程
- Agent 之间的 sessions_send 通信是内部会话，不进入群聊（但内容可通过群聊共享）
- 多维表格中的归档数据老板可随时查看
- Agent 的本地 MEMORY.md 仅该 Agent 可见（除非共享）

## 六、各 Agent 协作场景与提示词设计

### 6.1 典型协作场景矩阵

| 场景 | 主导 Agent | 参与 Agent | 产出 |
|------|-----------|-----------|------|
| 需求评审 | PM | 所有 Agent | PRD 评审报告 |
| 架构设计评审 | 架构师 | 前端、后端、运维 | 架构方案文档 |
| UI/UX 评审 | UI | PM、前端 | 交互设计规范 |
| 代码评审 | 架构师 | 前端、后端 | 代码质量报告 |
| 测试用例评审 | 测试 | PM、前后端 | 测试计划 |
| 部署上线 | DevOps | 架构师、测试 | 上线检查清单 |
| 故障排查 | DevOps | 所有相关 Agent | 故障分析报告 |

### 6.2 提示词设计原则

每个 Agent 的 SOUL.md 中需包含以下通用提示词规则：

```markdown
# 通用协作规则
1. 每当收到其他 Agent 的 sessions_send 消息时，你必须在 60 秒内响应
2. 对于技术问题，给出具体且有建设性的答复，避免"没问题"式的敷衍
3. 在不确定时，主动向其他 Agent 寻求帮助，而不是猜测
4. 记录每次协作中的收获，写入 MEMORY.md

# 角色边界规则
1. 不要越界——PM 不写代码，开发不编需求
2. 但要理解——每个角色都要能从其他角色的视角思考
3. 质疑是权力——对任何角色的产出都可以提出建设性质疑
4. 决策有流程——技术决策最终由架构师拍板，产品决策由 PM 拍板，但老板有最终否决权
```

### 6.3 以"代码评审"场景为例的完整交互编排

触发：前端 Agent 在群里 @architect "请评审我完成的用户登录组件代码"

**Step 1 - 架构师接收：**

→ 通过 sessions_send 获取前端的代码产出和文档

**Step 2 - 架构师分析：**

→ 从架构角度分析：组件边界、可复用性、性能影响、安全考虑
→ 从接口角度分析：API 调用是否合理、错误处理是否完善

**Step 3 - 架构师在群里发布评审意见：**

→ 列出发现的问题（带严重程度标记）
→ 提供具体的改进建议
→ @qa "建议同步评审此模块的测试用例覆盖度"

**Step 4 - 前端回应：**

→ 逐一回应评审意见
→ 对于不同意的地方，给出理由
→ 对于同意的地方，给出修改计划

**Step 5 - 老板可随时介入：**

→ 如果觉得某个评审意见有问题，可直接在群里指出
→ 如果是关键模块，可要求更多 Agent 参与评审

## 七、异常情况处理思路

### 7.1 Agent 超时或无响应

现象：某个 Agent 在预设时间内未响应其他 Agent 的请求

处理流程：

1. 协调 Agent 检测超时（60秒）
2. 在群里发送提醒：@agent_name "你已超时 60 秒，请确认状态"
3. 如 5 分钟后仍无响应，通知老板
4. 老板决定：重启该 Agent / 跳过该 Agent / 手动接管

### 7.2 飞书 API 调用失败

现象：飞书消息发送失败、多维表格写入失败等

处理流程：

1. 记录错误日志到本地文件
2. 重试机制：最多重试 3 次，间隔 5 秒
3. 如果持续失败，在群里通知老板
4. 降级方案：将内容暂存到本地，待恢复后同步

### 7.3 冲突与僵局

现象：两个 Agent 在某个问题上僵持不下（如技术方案选择）

处理流程：

1. 架构师汇总双方观点，在群里列出利弊对比
2. 如果 3 轮讨论后仍未解决，@boss 请求决策
3. 老板拍板后，所有 Agent 接受并记录

## 八、分阶段落地建议

### 第一阶段（第 1-2 周）——最小可行验证

**目标：** 用 3 个 Agent 跑通核心流程

**具体动作：**

- 创建 3 个 Agent（pm、architect、backend）
- 为这 3 个 Agent 编写 SOUL.md 和 IDENTITY.md
- 安装飞书插件 @tobeyoureyes/feishu，接入飞书群组
- 测试双通道通信（群聊消息 + sessions_send）是否正常
- 手动触发一次每日总结流程，观察是否正常
- 完成一次多角色评审（如"用户登录模块"评审）
- 创建飞书多维表格"每日总结"表，实现数据写入

### 第二阶段（第 3-4 周）——扩展到全角色

**目标：** 新增 UI、前端、测试、DevOps 四个 Agent

**具体动作：**

- 创建剩余的 4 个 Agent（ui、frontend、qa、devops）
- 为每个编写个性化的 SOUL.md
- 测试 7 个 Agent 的交叉通信和相互提问机制
- 完善飞书多维表格的四个表（每日总结、问答记录、个体学习记录、经验知识库）
- 建立每日定时自动触发总结（通过 cron 作业或飞书定时消息）
- 执行至少 3 次完整的全角色每日总结，观察交叉学习的深度

### 第三阶段（第 5-6 周）——记忆系统完善

**目标：** 建立完整的记忆和成长机制

**具体动作：**

- 完善经验提炼机制：Agent 自动将学到的内容写入 MEMORY.md
- 实现经验检索：每日开始前从多维表格检索历史经验
- 建立个体成长档案：通过多维表格跟踪每个 Agent 的成长轨迹
- 设置定期回顾会议（每周一次），由老板审查 Agent 的成长和系统的运行效果

### 第四阶段（第 7-8 周）——扩展与智能化

**目标：** 支持新增角色和高级功能

**具体动作：**

- 新增 marketing Agent（角色可预设：擅长数据分析、用户调研、市场洞察）
- 新增 data Agent（角色可预设：擅长数据清洗、统计分析、报表生成）
- 实现自动化的经验关联分析（多个 Agent 共同确认的重要经验自动置顶）
- 建立老板级别的仪表盘（通过飞书多维表格的仪表盘功能，监控整个系统运行状态）
- 优化异常处理和自动恢复机制

**扩展建议：** 增加新 Agent 时，只需创建其独立工作区和配置文件，定义 SOUL.md 中的角色特征，并将其 agentId 加入 tools.agentToAgent.allow 列表，即可自动融入系统，无需修改核心逻辑。

## 九、附录：关键代码示例

### 9.1 每日总结完整脚本示例（作为架构师 Agent 的提示词注入）

```markdown
# 每日总结流程 - 架构师 Agent 执行指令

你现在正在执行每日总结。今天是 {{current_date}}。

【步骤 1】生成自己的总结
基于今天（{{current_date}}）的工作记录和讨论历史，生成今日总结，按照以下模板：

### 今日工作总结
- 架构设计工作：[具体内容]
- 代码评审：[评审了哪些代码，发现了什么问题]
- 技术决策：[做出了哪些决策，理由是什么]

### 系统影响分析
分析以下角色今天的工作对系统架构的影响：
- 产品经理的工作影响：[...]
- UI 设计的工作影响：[...]
- 前端开发的工作影响：[...]
- 后端开发的工作影响：[...]
- 测试的工作影响：[...]

### 对其他角色的建议和质疑
基于上述影响分析，提出：
1. 对产品经理：[具体问题]
2. 对 UI 设计：[具体问题]
3. 对前端：[具体问题]
4. 对后端：[具体问题]
5. 对测试：[具体问题]

### 今日经验教训
- 今天学到的架构经验：[...]
- 今天发现的问题或教训：[...]

【步骤 2】发送总结
将以上总结内容：
- 通过飞书插件发送到群聊，格式："【架构师每日总结】[...]"
- 通过 sessions_send 推送到所有其他 Agent（pm、ui、frontend、backend、qa、devops）

【步骤 3】收集其他人的总结
等待其他 Agent 的总结到达后（群聊中可见），读取并分析他们的总结内容。

【步骤 4】生成交叉提问
基于阅读其他人的总结，生成一个提问列表发送到群聊：
1. @pm [产品相关问题]
2. @ui [设计相关问题]
3. @frontend [前端相关问题]
4. @backend [后端相关问题]
5. @qa [测试相关问题]

【步骤 5】归档记录
最后，将今日总结和交叉提问记录写入飞书多维表格"每日总结"表。
```

### 9.2 sessions_send 通信调用示例

```python
def send_cross_agent_message(sender_agent, target_agent, message, wait_response=True):
    """
    发送跨 Agent 消息
    sender_agent: 发送方 Agent ID
    target_agent: 目标 Agent ID（如 'architect', 'pm'）
    message: 消息内容
    """
    # 构建目标 session_key
    session_key = f"agent:{target_agent}:feishu:group:{CHAT_ID}"

    # 同时发送到飞书群聊（老板可见）
    send_feishu_group_message(
        f"<at user_id=\"{get_open_id(target_agent)}\">{target_agent}</at> {message}"
    )

    # 通过 sessions_send 发送到目标 Agent 的 session
    result = openclaw.invoke_tool("sessions_send", {
        "targetSession": session_key,
        "message": message,
        "timeoutSeconds": 60 if wait_response else 0
    })

    return result
```

### 9.3 飞书卡片交互示例（供老板决策）

```json
{
  "msg_type": "interactive",
  "card": {
    "elements": [
      {
        "tag": "div",
        "text": {
          "content": "**【架构师】** 认证模块的 API 设计有两种方案，请选择",
          "tag": "lark_md"
        }
      },
      {
        "tag": "action",
        "actions": [
          {
            "tag": "button",
            "text": {
              "content": "方案 A：RESTful API",
              "tag": "plain_text"
            },
            "value": "restful",
            "type": "primary"
          },
          {
            "tag": "button",
            "text": {
              "content": "方案 B：GraphQL",
              "tag": "plain_text"
            },
            "value": "graphql",
            "type": "default"
          }
        ]
      }
    ]
  }
}
```

老板点击按钮后，飞书会触发 card.action.trigger 回调，OpenClaw 接收后通知相关 Agent 执行后续操作。

---

以上方案充分考虑了 OpenClaw 平台的约束（仅通过 sessions_send 内部通信、飞书作为唯一外部交互渠道），并提供了一套可落地、分阶段实施的完整方案。你可以从第一阶段开始逐步推进，根据实际运行效果持续优化各 Agent 的 SOUL.md 提示词和流程细节。
