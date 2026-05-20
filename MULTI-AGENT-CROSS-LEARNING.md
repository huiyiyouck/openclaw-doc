# 多 Agent 交叉学习与每日总结系统架构方案（Coordinator 编排模式）

> 版本：v1.0 | 日期：2026-05-20
> 基于平台：OpenClaw 2026.5.18 + 飞书开放平台

## 背景

在《多 Agent 交叉学习与每日总结系统架构方案》初版中，我们设计了基于 P2P 模式的 Agent 间直接通信方案——每个 Agent 通过 `sessions_send` 直接向其他 Agent 发送消息，各自独立发起总结和提问。

在实际落地评估中发现以下问题：

1. **通信复杂度高**：7 个 Agent 之间 P2P 通信需要 N×(N-1) 条链路，新增角色时所有 Agent 都需调整
2. **流程无法统一调度**：没有全局协调者，各 Agent 独立触发总结，难以保证"先收集→再交叉审阅→再讨论"的有序推进
3. **讨论轮次不可控**：P2P 模式下多轮讨论容易发散，缺乏统一的超时和共识判定机制
4. **人类干预入口分散**：老板需要 @ 不同 Agent 才能干预，无法通过统一入口操作

基于以上问题，本方案引入 **Coordinator（协调者）Agent**，采用集中编排模式重新设计交互流程。Coordinator 作为唯一调度中心，负责流程编排、消息分发、轮次控制和结果归档，各角色 Agent 只需响应 Coordinator 的请求，无需直接感知其他 Agent。

两份文档的关系：
- 《多 Agent 交叉学习与每日总结系统架构方案》— 初版方案，P2P 模式，包含完整的 Agent SOUL.md 模板、协作场景矩阵和代码示例
- 本文档（Coordinator 编排模式）— 迭代方案，解决初版的落地问题，聚焦编排流程、交叉审阅机制、记忆体系和分阶段实施

---

## 1. 系统整体架构

### 1.1 架构总览

```
                        ┌──────────────────────────────┐
                        │        飞书群（共享空间）        │
                        │  ┌─────┐ ┌─────┐ ┌─────┐    │
                        │  │老板  │ │卡片  │ │消息流│    │
                        │  │(你)  │ │消息  │ │     │    │
                        │  └──┬──┘ └──▲──┘ └──▲──┘    │
                        │     │@mention│       │        │
                        │     │   回复  │  实时推送       │
                        └─────┼────────┼───────┼────────┘
                              │        │       │
            ┌─────────────────┼────────┼───────┼──────────────┐
            │  OpenClaw Gateway (port 18789)                   │
            │                                                   │
            │  ┌──────────────────────────────────────────┐    │
            │  │           Coordinator (协调者 Agent)       │    │
            │  │  - 编排每日 review 流程                    │    │
            │  │  - 收集/分发总结                           │    │
            │  │  - 管理讨论轮次                            │    │
            │  │  - 读写共享记忆 (Bitable)                  │    │
            │  │  - 处理人类干预                            │    │
            │  └────┬─────┬─────┬─────┬─────┬─────┬─────┘    │
            │       │     │     │     │     │     │            │
            │  sessions_send (内部通信)                       │
            │       │     │     │     │     │     │            │
            │  ┌────▼──┐┌──▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼───┐    │
            │  │ 产品  ││架构 ││ UI ││前端││后端││测试  │    │
            │  │ 经理  ││ 师  ││设计││ 开发││开发││  师  │    │
            │  └───┬───┘└──┬──┘└─┬──┘└─┬──┘└─┬──┘└──┬──┘    │
            │      │       │     │     │     │      │        │
            │  ┌───▼───────▼─────▼─────▼─────▼──────▼──┐     │
            │  │       各 Agent 独立 Workspace            │     │
            │  │  MEMORY.md / memory/*.md / SOUL.md       │     │
            │  └─────────────────────────────────────────┘     │
            │                                                   │
            │  ┌─────────────────────────────────────────┐     │
            │  │       共享记忆 (飞书多维表格 Bitable)      │     │
            │  │  - daily_summaries (每日总结)             │     │
            │  │  - cross_reviews (交叉评审记录)           │     │
            │  │  - learnings (学习笔记)                   │     │
            │  │  - action_items (待改进项)                │     │
            │  │  - knowledge_base (共享知识库)            │     │
            │  └─────────────────────────────────────────┘     │
            └───────────────────────────────────────────────────┘
```

### 1.2 Agent 角色定义

| Agent ID | 角色 | Emoji | 职责 |
|----------|------|-------|------|
| `coordinator` | 协调者 | 🎯 | 编排每日 review，管理讨论流程，读写共享记忆 |
| `pm` | 产品经理 | 📦 | 需求分析、PRD、优先级、用户故事 |
| `architect` | 架构师 | 🏗️ | 系统设计、技术选型、接口规范、性能规划 |
| `ui-designer` | UI 设计 | 🎨 | 交互设计、视觉规范、组件库、设计系统 |
| `frontend` | 前端开发 | 💻 | 页面实现、组件开发、状态管理、性能优化 |
| `backend` | 后端开发 | ⚙️ | API 实现、数据模型、业务逻辑、服务治理 |
| `tester` | 测试工程师 | 🔍 | 测试用例、质量标准、缺陷分析、回归策略 |
| `devops` | 部署运维 | 🚀 | CI/CD、部署策略、监控告警、基础设施 |

> **可扩展设计**：新增角色只需在 `openclaw.json` 的 `agents.list` 中添加条目、创建 workspace、注册到 coordinator 的 agent 列表。核心流程无需改动。

### 1.3 核心设计原则

1. **Coordinator 编排模式**：用一个专职协调者 Agent 管理整个流程，而非 P2P 通信。原因：
   - `sessions_send` 不支持广播，P2P 需要 N×N 通信对
   - 协调者可以做全局调度、冲突检测、进度跟踪
   - 新增 Agent 只需向协调者注册

2. **内部通信用 `sessions_send`，外部输出走飞书**：
   - Agent 间通过 `sessions_send` 同步交流（带 timeout 等待回复）
   - 所有产出通过协调者统一推送到飞书群（卡片消息格式）
   - 人类干预通过飞书群消息触达协调者

3. **分层记忆架构**：
   - **个人记忆**：每个 Agent 的 MEMORY.md + memory/*.md（私有认知）
   - **共享记忆**：飞书多维表格（跨 Agent 知识库）
   - **梦境整理**：memory-core 的 dreaming 功能自动巩固每日学习

---

## 2. 核心交互流程

### 2.1 每日交叉 Review 全流程

以一个完整的工作日为例，假设老板今天下达了一个任务："实现用户登录功能"。

```
时间线（每日 18:00 触发）：

18:00  Cron 触发 → Coordinator 启动每日 Review
 │
 ├─ Phase 1: 总结收集 (18:00 - 18:15)
 │   Coordinator → sessions_send → 每个 Agent: "请总结今天的工作"
 │   各 Agent 返回结构化总结
 │
 ├─ Phase 2: 交叉分发 (18:15 - 18:20)
 │   Coordinator 整理所有人总结 → 发给每个 Agent（去掉自己的）
 │   "请从你的角色角度，审阅其他人的工作总结，提出疑问和质疑"
 │
 ├─ Phase 3: 交叉讨论 (18:20 - 18:50)
 │   Coordinator 收集所有 Review 意见
 │   转发给被 Review 的 Agent 进行回应
 │   可进行 2-3 轮 Ping-Pong 讨论
 │   每轮结果实时推送到飞书群
 │
 ├─ Phase 4: 学习沉淀 (18:50 - 19:00)
 │   Coordinator → 每个 Agent: "今天学到了什么？"
 │   各 Agent 更新自己的 MEMORY.md
 │   Coordinator 写入共享 Bitable
 │
 └─ Phase 5: 总结报告 (19:00)
     Coordinator 生成最终报告卡片，发送到飞书群
```

### 2.2 Phase 1: 总结收集（详细步骤）

**Step 1.1 - Cron 触发**

在 `cron/jobs.json` 中配置定时任务：

```json
{
  "daily-review": {
    "schedule": "0 18 * * 1-5",
    "timezone": "Asia/Shanghai",
    "agentId": "coordinator",
    "message": "开始今日交叉 Review。请执行 daily-review 流程。",
    "enabled": true
  }
}
```

> 仅工作日（周一到周五）18:00 触发。

**Step 1.2 - Coordinator 向每个 Agent 收集总结**

Coordinator 使用 `sessions_send` 向每个 Agent 发送消息：

```
目标 session: agent:{agentId}:daily-review
消息内容:
"""
今天是 2026-05-20（周三）。请总结你今天参与的工作。

总结要求：
1. 【我做了什么】列出你今天完成的具体事项
2. 【我的决策】列出你做出的关键决策和理由
3. 【遇到的困难】列出遇到的阻碍或不确定的点
4. 【对别人的影响】你的工作对其他角色可能产生什么影响
5. 【明天的计划】你打算明天做什么

请基于你的 workspace 记忆文件组织回答。保持简洁，每项不超过 2 句话。
"""
```

**Step 1.3 - Agent 处理总结请求**

每个 Agent 收到请求后：
1. 读取自己的 `memory/YYYY-MM-DD.md`（当天记忆）
2. 读取 `MEMORY.md`（长期记忆中的近期上下文）
3. 按结构化格式返回总结

**示例 - 产品经理的总结**：

```markdown
## 📦 产品经理 - 今日总结 (2026-05-20)

### 我做了什么
- 完成用户登录功能的 PRD，包含手机号+验证码、邮箱+密码两种方式
- 定义了登录状态管理方案：JWT + Refresh Token

### 我的决策
- 放弃了第三方 OAuth 登录（微信/Apple），因为 MVP 阶段优先减少集成复杂度
- Token 过期时间设为 2 小时，平衡安全性和用户体验

### 遇到的困难
- 不确定是否需要支持"记住我"功能，需要和架构师讨论

### 对别人的影响
- 架构师：需要确认 JWT 方案在微服务架构下的鉴权机制
- 后端：需要实现验证码发送和校验服务
- UI 设计：需要设计登录页的验证码输入交互

### 明天的计划
- 根据今天 review 的反馈完善 PRD
- 开始定义用户注册流程的需求
```

### 2.3 Phase 2: 交叉分发与审阅

**Step 2.1 - Coordinator 编排交叉审阅**

Coordinator 收集所有总结后，构建一个"交叉审阅包"，发送给每个 Agent：

```
目标 session: agent:{agentId}:daily-review
消息内容:
"""
请从【你的角色】的角度，审阅以下团队成员今日工作总结。
对每个总结，你需要：

1. ✅ 认可点：他们做对了什么，哪些决策你认同
2. ❓ 质疑点：哪些决策你不同意，哪些假设你认为有问题
3. 🔍 追问：你需要更多信息的点，模棱两可的地方
4. 💡 建议：从你的专业角度，可以怎么改进

【规则】
- 不要泛泛而谈，每个点必须具体
- 必须至少对 2 个其他角色的总结提出实质性质疑
- 质疑要有理有据，不能只说"我觉得不对"

--- 以下是各角色的总结 ---

## 📦 产品经理的总结
{pm_summary}

## 🏗️ 架构师的总结
{architect_summary}

## 🎨 UI 设计的总结
{ui_summary}

## 💻 前端开发的总结
{frontend_summary}

## ⚙️ 后端开发的总结
{backend_summary}

## 🔍 测试的总结
{tester_summary}

## 🚀 部署运维的总结
{devops_summary}
"""
```

**Step 2.2 - 交叉审阅示例**

```
## ⚙️ 后端开发 对 📦 产品经理 总结的审阅

### ✅ 认可
- 放弃 OAuth 是对的，验证码服务我们有现成的短信通道可以复用
- JWT + Refresh Token 是标准方案，我这边没问题

### ❌ 质疑
- 你说 Token 过期 2 小时，但 Refresh Token 的过期时间呢？如果也是 2 小时，
  那"记住我"功能就不是做不做的问题，而是必须做——否则用户每 2 小时就要重新登录。
  建议你明确 Refresh Token 策略。

### 🔍 追问
- "微服务架构下的鉴权"——你是指需要 API Gateway 统一验签，还是服务间也要鉴权？
  这两个复杂度差很多，我需要你明确需求边界。

### 💡 建议
- 建议增加"登录安全策略"章节：连续失败锁定、异地登录检测、设备指纹等
- 验证码 6 位还是 4 位？有效期几分钟？这些参数需要 PRD 里写清楚

---
## 🏗️ 架构师 对 📦 产品经理 总结的审阅

### ❓ 质疑
- 你说"和架构师讨论记住我功能"——这个不应该由架构师来决定。这是产品需求。
  用户期望是什么？竞品怎么做的？你先定义产品层面的"记住我"是什么，
  我才能设计技术方案。不要把产品决策推给技术。

### 💡 建议
- MVP 可以先不做"记住我"，但在 PRD 里标注为 P1（下个迭代必须），
  这样我们在架构设计时可以预留扩展性
```

### 2.4 Phase 3: 交叉讨论（Ping-Pong）

**Step 3.1 - Coordinator 分发审阅意见**

Coordinator 将针对某个 Agent 的所有审阅意见汇总，发送给该 Agent：

```
消息内容:
"""
以下是对你今日总结的审阅意见，请逐条回应：

## 来自 ⚙️ 后端开发
{backend_review_of_pm}

## 来自 🏗️ 架构师
{architect_review_of_pm}

请你：
1. 对每个质疑点给出明确回应（接受/不接受/需要讨论）
2. 回答所有追问
3. 对建议标注是否采纳
"""
```

**Step 3.2 - 多轮讨论**

使用 `sessions_send` 的 ping-pong 模式（`maxPingPongTurns: 3`），让 Agent 之间进行深度讨论。

```
产品经理回应后端开发的质疑:
"Refresh Token 过期时间我定义为 7 天，在 PRD 里补充了。
 '记住我'不做的决定我维持——MVP 阶段用户体验可以妥协。
 但你说得对，我会把安全策略章节补上。"

后端开发追问:
"7 天可以。但 MVP 不做记住我，用户 7 天后重新登录，验证码一条 5 分钱，
 假设日活 1000，一个月就是 1500 块。产品确认这个成本可接受吗？"

产品经理最终回应:
"确认可接受。这是 MVP，活跃用户不会有 1000。我会在 PRD 里标注成本估算。"

Coordinator 判定: 达成共识 ✓
```

### 2.5 Phase 4: 学习沉淀

**Step 4.1 - Coordinator 要求每个 Agent 写学习笔记**

```
消息内容:
"""
今日 Review 已完成。请回答以下问题：

1. 【今天我学到了什么】从别人的审阅和讨论中，我获得了哪些新认知？
2. 【我需要修正什么】我的哪些假设被证明是错误的，或哪些知识需要更新？
3. 【对其他角色的理解】我现在对其他角色的工作有了什么新的理解？

请将回答写入你的 memory/YYYY-MM-DD.md 文件的"每日学习"章节。
同时更新 MEMORY.md 中需要修正的内容。
"""
```

**Step 4.2 - Agent 写入个人记忆**

```
## ⚙️ 后端开发的 memory/2026-05-20.md

### 每日学习
- 从产品经理的总结中理解到"记住我"不是纯技术决策，
  而是涉及用户体验和成本平衡的产品决策
- 架构师提出 API Gateway 和服务间鉴权是两个不同复杂度的事，
  以后做技术方案评估时要区分
- 测试的审阅让我意识到：登录失败的场景（验证码过期、网络超时）
  我没有在 API 设计中明确错误码，需要补充
```

**Step 4.3 - Coordinator 写入共享 Bitable**

Coordinator 使用 `feishu-bitable` skill 将关键信息写入多维表格：

| 表 | 字段 |
|---|---|
| `daily_summaries` | date, agent_id, summary_text |
| `cross_reviews` | date, reviewer_id, reviewee_id, question, response, status |
| `learnings` | date, agent_id, learning_text, source_agent |
| `action_items` | date, agent_id, action, priority, due_date, status |

### 2.6 Phase 5: 飞书群报告

Coordinator 生成最终报告并通过飞书卡片消息发送到群：

```json
{
  "header": {
    "title": { "content": "📋 每日交叉 Review 报告 (2026-05-20)", "tag": "plain_text" },
    "template": "blue"
  },
  "elements": [
    {
      "tag": "column_set",
      "columns": [
        { "tag": "column", "width": "auto", "elements": [
          { "tag": "markdown", "content": "**📦 PM**: 完成登录 PRD\n**🏗️ 架构**: 确定 JWT 鉴权方案\n**🎨 UI**: 登录页线框图\n**💻 前端**: 无\n**⚙️ 后端**: 验证码服务调研\n**🔍 测试**: 登录测试用例草稿\n**🚀 运维**: 无" }
        ]}
      ]
    },
    { "tag": "hr" },
    {
      "tag": "markdown",
      "content": "**🔥 关键讨论**\n• PM vs 架构师: \"记住我\"是产品还是技术决策？→ **共识: 产品决策**\n• 后端 vs PM: 验证码成本问题 → **确认 MVP 可接受**\n• 测试 vs 后端: 缺少错误码定义 → **后端采纳, 明天补充**"
    },
    { "tag": "hr" },
    {
      "tag": "markdown",
      "content": "**📈 今日学习摘要**\n• PM: 学会了从成本角度评估功能优先级\n• 后端: 理解了产品决策和技术决策的边界\n• 测试: 了解到验证码安全策略的业界标准"
    },
    { "tag": "hr" },
    {
      "tag": "action",
      "actions": [
        { "tag": "button", "text": { "content": "查看完整记录", "tag": "plain_text" }, "type": "primary", "url": "https://feishu.cn/docx/xxx" },
        { "tag": "button", "text": { "content": "追加提问", "tag": "plain_text" }, "type": "default", "value": { "action": "ask_question" } }
      ]
    }
  ]
}
```

---

## 3. 交叉理解与相互提问机制

### 3.1 机制一：角色视角审阅提示词

每个 Agent 的 AGENTS.md 中定义其审阅视角，确保从专业领域出发提出有价值的质疑：

```markdown
# agents/pm/workspace/AGENTS.md (节选)

## 交叉审阅视角

在审阅其他角色的工作时，你应该关注：

### 对架构师
- 系统设计是否覆盖了所有用户场景？
- 技术选型是否与产品优先级对齐？
- 扩展性设计是否过度工程化（MVP 阶段）？

### 对后端开发
- API 设计是否满足前端交互需求？
- 错误处理是否考虑了用户侧体验？
- 性能指标是否满足产品 SLA？

### 对 UI 设计
- 设计方案是否覆盖了所有状态（空态、加载态、错误态）？
- 交互流程是否有断点（用户走到某一步走不通）？
- 是否有足够的设计规范供前端实现？

### 对测试
- 测试用例是否覆盖了核心用户路径？
- 是否有遗漏的边界场景？
```

### 3.2 机制二：结构化审阅模板

Coordinator 发送的审阅请求使用固定模板，确保质疑有结构：

```
请从你的角色角度审阅以下总结。对每条审阅，你必须填写：

| 维度 | 要求 |
|------|------|
| 引用 | 具体引用对方的哪句话/哪个决策 |
| 类型 | 认同 / 质疑 / 追问 / 建议 |
| 理由 | 为什么认同/质疑，你的依据是什么 |
| 影响 | 如果不解决，会产生什么后果 |
| 建议 | 你认为应该怎么做 |

约束：
- 至少提出 2 个质疑（类型=质疑）
- 至少提出 1 个追问（类型=追问）
- 每个质疑必须有"影响"说明
- 不允许"看起来不错"这类空洞的认同
```

### 3.3 机制三：Ping-Pong 讨论深化

使用 `sessions_send` 的 `maxPingPongTurns` 参数控制讨论深度：

```json
{
  "session": {
    "agentToAgent": {
      "maxPingPongTurns": 3
    }
  }
}
```

讨论流程：
1. **Round 1**：审阅者提问 → 被审阅者回应
2. **Round 2**：审阅者追问/反驳 → 被审阅者再次回应
3. **Round 3**：最终确认（达成共识/标记分歧/升级给老板）

Agent 可以返回 `REPLY_SKIP` 提前结束（表示已达成共识）。

### 3.4 机制四：共识与分歧追踪

Coordinator 维护每场讨论的状态：

```json
{
  "discussions": [
    {
      "id": "disc-001",
      "topic": "记住我功能的归属",
      "participants": ["pm", "architect"],
      "rounds": 2,
      "status": "consensus",
      "consensus": "产品决策，PM 负责定义需求",
      "recorded": true
    },
    {
      "id": "disc-002",
      "topic": "Token 过期时间策略",
      "participants": ["pm", "backend"],
      "rounds": 3,
      "status": "escalated",
      "reason": "成本问题需要老板确认",
      "needsHuman": true
    }
  ]
}
```

---

## 4. 持续学习与成长方案

### 4.1 分层记忆架构

```
┌─────────────────────────────────────────┐
│            Agent 个人记忆                 │
│  ┌─────────────────────────────────┐    │
│  │  MEMORY.md (长期认知/经验法则)    │    │
│  │  "后端 API 的错误码必须提前定义"  │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │  memory/YYYY-MM-DD.md (每日日记) │    │
│  │  当天的工作细节和学到的具体知识    │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │  memory/learnings/ (学习条目)    │    │
│  │  LRN-20260520-001: ...          │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │  memory/cross-knowledge/         │    │
│  │  从其他角色学到的领域知识          │    │
│  │  backend-api-design.md           │    │
│  │  testing-strategy.md             │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│          共享记忆 (飞书 Bitable)          │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ │
│  │ summaries│ │ reviews  │ │ learnings│ │
│  │ 每日总结 │ │ 评审记录 │ │ 学习笔记 │ │
│  └─────────┘ └──────────┘ └──────────┘ │
│  ┌─────────┐ ┌──────────┐              │
│  │ actions  │ │ patterns │              │
│  │ 待改进项 │ │ 反复出现 │              │
│  │          │ │ 的模式   │              │
│  └─────────┘ └──────────┘              │
└─────────────────────────────────────────┘
```

### 4.2 个人记忆更新流程

**每次 Review 后的更新**：

```
Coordinator 发送给 Agent:
"""
请根据今天的交叉 Review 结果，执行以下记忆更新：

1. 在 memory/{today}.md 添加"交叉学习"章节
2. 如果有值得长期记住的认知，更新 MEMORY.md
3. 如果发现了知识盲区，写入 memory/learnings/LRN-{date}-{seq}.md
4. 如果学到了其他角色的领域知识，写入 memory/cross-knowledge/{topic}.md

记忆格式参考：
- MEMORY.md: 沉淀后的认知规则，简洁、可执行
- learnings: 具体事件 + 教训 + 如何避免
- cross-knowledge: 从其他角色视角的理解，附上来源角色和日期
"""
```

### 4.3 跨日记忆巩固

利用 OpenClaw 的 memory-core dreaming 机制：

```json
{
  "memory-core": {
    "enabled": true,
    "config": {
      "dreaming": {
        "enabled": true,
        "frequency": "0 3 * * *",
        "timezone": "Asia/Shanghai"
      }
    }
  }
}
```

凌晨 3:00 自动触发梦境整理：
- 回顾当天的 `memory/*.md` 文件
- 将反复出现的 learnings 提升到 MEMORY.md
- 清理过时的记忆条目
- 生成 DREAMS.md 摘要

### 4.4 共享知识库 (Bitable) 的读写

**写入时机**：
- 每日 Review 结束后，Coordinator 将所有总结、评审、学习写入 Bitable
- 每次任务讨论中产生的共识写入 `knowledge_base` 表

**读取时机**：
- Coordinator 在编排 Review 前，从 Bitable 读取历史记录，提供上下文
- Agent 在回答总结请求时，Coordinator 注入该 Agent 近期的学习趋势

**Bitable 表结构**：

**daily_summaries（每日总结）**

| 字段 | 类型 | 说明 |
|------|------|------|
| date | 日期 | 审阅日期 |
| agent_id | 文本 | Agent 标识 |
| what_i_did | 文本 | 我做了什么 |
| decisions | 文本 | 关键决策 |
| difficulties | 文本 | 遇到的困难 |
| impact_on_others | 文本 | 对别人的影响 |
| tomorrow_plan | 文本 | 明天的计划 |

**cross_reviews（交叉评审）**

| 字段 | 类型 | 说明 |
|------|------|------|
| date | 日期 | 审阅日期 |
| reviewer_id | 文本 | 审阅者 |
| reviewee_id | 文本 | 被审阅者 |
| review_type | 单选 | 认同/质疑/追问/建议 |
| content | 文本 | 审阅内容 |
| response | 文本 | 被审阅者回应 |
| status | 单选 | 待回应/已回应/达成共识/升级 |

**learnings（学习笔记）**

| 字段 | 类型 | 说明 |
|------|------|------|
| date | 日期 | 学习日期 |
| agent_id | 文本 | 学习者 |
| source_agent | 文本 | 知识来源角色 |
| category | 单选 | 认知修正/新知识/最佳实践/避坑 |
| content | 文本 | 学习内容 |
| applied | 复选框 | 是否已应用 |

**patterns（反复出现的模式）**

| 字段 | 类型 | 说明 |
|------|------|------|
| pattern | 文本 | 模式描述 |
| occurrences | 数字 | 出现次数 |
| first_seen | 日期 | 首次出现 |
| last_seen | 日期 | 最近出现 |
| status | 单选 | 未解决/已改进/系统性问题 |

### 4.5 "成长追踪"机制

Coordinator 在每次 Review 后评估各 Agent 的进步情况：

```
评估维度：
1. 被质疑次数趋势（下降 = 进步）
2. 提出有价值质疑的次数（上升 = 进步）
3. 反复犯同样错误的次数（下降 = 进步）
4. 采纳他人建议的比率（合理范围 = 开放度）
5. 交叉知识积累量（上升 = 跨领域能力增长）
```

每月生成一份"成长报告"卡片发送到飞书群。

---

## 5. 飞书群实时同步与人工干预

### 5.1 消息推送策略

整个 Review 过程中，Coordinator 将关键节点实时推送到飞书群：

```
时间线 → 飞书群消息

18:00  📢 "今日交叉 Review 开始！参与角色：PM, 架构师, UI, 前端, 后端, 测试, 运维"

18:05   📦 PM 今日总结
18:07   🏗️ 架构师今日总结
18:09   🎨 UI 设计今日总结
       ... (逐个推送，保持节奏感)

18:15   🔥 "交叉审阅开始！每个角色正在审阅其他人的工作..."

18:25   ⚙️→📦 后端对 PM 的审阅（卡片）
        "❓ 质疑: Token 过期 2h 的 Refresh Token 策略？"
        [查看详情] [追加提问]

18:27   🏗️→📦 架构师对 PM 的审阅（卡片）
        ...

18:35   💬 "PM 正在回应后端和架构师的审阅..."

18:40   📦→⚙️ PM 回应后端："Refresh Token 7 天，PRD 已补充"
        [达成共识 ✓] [继续追问]

18:50   📝 "学习阶段：各角色正在记录今日收获..."

19:00   📋 今日 Review 完整报告（汇总卡片）
        "达成共识 5 项 | 仍有分歧 1 项（需老板裁决）| 新增学习 12 条"
        [查看完整记录] [导出飞书文档]
```

### 5.2 实时推送的卡片消息格式

**审阅意见卡片**：

```json
{
  "header": {
    "title": { "content": "⚙️→📦 后端对产品经理的审阅", "tag": "plain_text" },
    "template": "orange"
  },
  "elements": [
    {
      "tag": "markdown",
      "content": "**❓ 质疑: Token 过期时间策略不完整**\n\n> 产品经理说 Token 过期 2 小时，但未定义 Refresh Token 策略。如果 Refresh Token 也是 2 小时，用户每 2 小时就要重新登录，体验很差。\n\n**影响**: 如果不明确，后端无法设计 Token 刷新机制\n**建议**: Refresh Token 设为 7 天，并增加\"记住我\"选项"
    },
    { "tag": "hr" },
    {
      "tag": "markdown",
      "content": "**🔍 追问: API Gateway 鉴权范围**\n\n> 你说的\"微服务鉴权\"是指 Gateway 统一验签，还是服务间也要鉴权？复杂度差很多。"
    },
    { "tag": "hr" },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "content": "查看 PM 回应", "tag": "plain_text" },
          "type": "primary",
          "value": { "action": "view_response", "discussion_id": "disc-002" }
        },
        {
          "tag": "button",
          "text": { "content": "追加提问", "tag": "plain_text" },
          "type": "default",
          "value": { "action": "ask_question", "reviewer": "backend", "reviewee": "pm" }
        }
      ]
    }
  ]
}
```

### 5.3 人类干预机制

**方式一：直接回复飞书消息**

用户在飞书群中回复某条消息，Coordinator 检测到后：

```
用户回复: "Token 过期时间这个问题，我的意见是 MVP 阶段统一 24 小时过期，
         不做 Refresh Token，减少复杂度。"

Coordinator 处理:
1. 识别这是一条干预指令
2. 通知 PM 和后端: "老板对 Token 策略给出了决策：MVP 统一 24h，不做 Refresh Token"
3. 更新讨论状态为 "resolved_by_boss"
4. 写入 Bitable 的 decisions 表
```

**方式二：@特定 Agent**

用户在群里 @某个机器人（如后端开发），直接对该 Agent 发出指令：

```
用户: @后端开发 "验证码服务的调研结果给我看看"

后端 Agent 收到消息后:
1. 查阅自己的 memory/2026-05-20.md
2. 在群里回复调研结果
```

**方式三：通过 Coordinator 指令**

用户向 Coordinator（程一）发送特殊命令：

```
/override Token 过期策略改为 24 小时
/pause 暂停讨论
/resume 继续讨论
/skip 跳过当前话题
/add-topic 需要讨论 xxx
```

### 5.4 Coordinator 处理人类消息的逻辑

Coordinator 的 AGENTS.md 中定义：

```markdown
## 人类干预处理

当收到来自飞书群的消息（非 sessions_send 来源），按以下逻辑处理：

1. 判断消息类型：
   - 回复某条审阅消息 → 提取为对该讨论的干预
   - @特定 Agent → 转发给目标 Agent
   - 直接发消息给 Coordinator → 作为指令处理

2. 干预处理：
   - 立即暂停当前讨论流程
   - 将人类意见注入相关讨论
   - 通知所有相关 Agent
   - 记录决策到 Bitable

3. 恢复流程：
   - 等待人类确认后再继续
   - 或自动在 5 分钟后继续（如果没有暂停指令）
```

---

## 6. 异常情况处理

### 6.1 Agent 不可用

| 场景 | 处理方式 |
|------|---------|
| `sessions_send` 超时 | Coordinator 跳过该 Agent，标注为"未参与"，继续流程 |
| Agent 返回错误 | 记录错误日志，使用其最近一次有效总结（从 Bitable 读取） |
| 多个 Agent 同时不可用 | 继续可用的 Agent 之间的讨论，标注缺席者 |

```json
// Coordinator 超时配置
{
  "sessions_send": {
    "timeoutSeconds": 120,
    "onTimeout": "skip_and_log"
  }
}
```

### 6.2 讨论陷入僵局

| 场景 | 处理方式 |
|------|---------|
| 3 轮 Ping-Pong 后仍无共识 | 自动升级为"需老板裁决"，推送到飞书群 |
| 两个 Agent 对同一问题有冲突 | Coordinator 生成摘要对比，请老板决策 |
| 讨论偏离主题 | Coordinator 介入提醒，引导回正题 |

### 6.3 模型调用失败

| 场景 | 处理方式 |
|------|---------|
| API 限流 (429) | 指数退避重试，最多 3 次 |
| 上下文超限 | Coordinator 压缩历史摘要后重试 |
| 模型幻觉 | 交叉审阅本身是纠错机制，多角色质疑可发现幻觉 |

### 6.4 飞书消息发送失败

| 场景 | 处理方式 |
|------|---------|
| Webhook 失效 | Coordinator 将结果写入本地文件 `review-results/YYYY-MM-DD.json`，标记为待发送 |
| 卡片格式错误 | 降级为纯文本消息 |
| 群聊权限变更 | 提醒老板检查机器人权限 |

### 6.5 记忆一致性

| 场景 | 处理方式 |
|------|---------|
| Agent 的 MEMORY.md 与 Bitable 矛盾 | 以 Bitable 为准（共享记忆优先） |
| 梦境整理误删重要记忆 | MEMORY.md 纳入 git 管理，可回滚 |
| 新 Agent 加入时缺乏历史上下文 | Coordinator 从 Bitable 生成"快速入门包" |

---

## 7. 分阶段落地建议

### Phase 0: 基础设施准备（第 1 天）

**目标**：建立基础环境

**操作清单**：

1. **创建飞书机器人应用**
   - 在飞书开放平台创建 8 个应用（1 coordinator + 7 角色机器人）
   - 每个应用开通消息收发权限、事件订阅
   - 所有应用添加到同一个飞书群
   - 至少 Coordinator 需要多维表格权限

2. **创建飞书多维表格**
   - 新建 Bitable，包含 5 张表（见 4.4 节结构）
   - 获取 app_token 和 table_id

3. **验证飞书连通性**
   - 确认每个机器人能在群里发消息
   - 确认 Coordinator 能读写 Bitable

> **简化方案**（如果嫌 8 个飞书应用太多）：
> 先只用 1 个 Coordinator 机器人 + 1 个 webhook，所有消息通过 Coordinator 统一发送，
> 用卡片标题区分角色。功能验证通过后再逐步拆分为独立机器人。

### Phase 1: 最小可用版本（第 2-4 天）

**目标**：跑通"总结收集 → 飞书群展示"的基本流程

**范围**：
- 创建 Coordinator Agent + 3 个角色 Agent（PM、后端、测试）
- 实现每日 Cron 触发 + 总结收集
- Coordinator 将总结推送到飞书群（文本消息）
- 人工查看总结，手动在群里提问

**配置示例**：

```json
// openclaw.json 新增 agents
{
  "agents": {
    "list": [
      { "id": "main", "name": "程一-秘书", "model": "deepseek/deepseek-v4-pro", "..." : "..." },
      { "id": "finance", "name": "程二-财务", "model": "volcengine-plan/glm-5.1", "...": "..." },
      {
        "id": "coordinator",
        "name": "协调者",
        "workspace": "/root/.openclaw/workspace-coordinator",
        "model": "deepseek/deepseek-v4-pro",
        "identity": { "name": "协调者", "emoji": "🎯" }
      },
      {
        "id": "pm",
        "name": "产品经理",
        "workspace": "/root/.openclaw/workspace-pm",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "产品经理", "emoji": "📦" }
      },
      {
        "id": "backend",
        "name": "后端开发",
        "workspace": "/root/.openclaw/workspace-backend",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "后端开发", "emoji": "⚙️" }
      },
      {
        "id": "tester",
        "name": "测试工程师",
        "workspace": "/root/.openclaw/workspace-tester",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "测试工程师", "emoji": "🔍" }
      }
    ]
  }
}
```

**Coordinator 的 Cron 配置**：

```json
{
  "daily-review": {
    "schedule": "0 18 * * 1-5",
    "timezone": "Asia/Shanghai",
    "agentId": "coordinator",
    "message": "执行每日交叉 Review。",
    "enabled": true
  }
}
```

**验证标准**：
- [ ] Cron 能准时触发 Coordinator
- [ ] Coordinator 能通过 sessions_send 联系 PM、后端、测试
- [ ] 各 Agent 返回结构化总结
- [ ] 总结能推送到飞书群

### Phase 2: 交叉审阅（第 5-7 天）

**目标**：实现 Agent 间自动交叉审阅

**范围**：
- Coordinator 将所有人总结交叉分发给其他角色
- 每个 Agent 生成带质疑的审阅意见
- 审阅意见推送到飞书群（卡片格式）
- 被审阅者可以回应

**新增配置**：

```json
{
  "session": {
    "agentToAgent": {
      "maxPingPongTurns": 3
    }
  }
}
```

**验证标准**：
- [ ] 每个 Agent 能对其他角色总结提出实质性质疑
- [ ] 被审阅者能回应质疑
- [ ] 飞书群能看到完整的审阅和回应

### Phase 3: 记忆与学习（第 8-10 天）

**目标**：实现持续学习机制

**范围**：
- 创建飞书多维表格，建立 5 张表
- Review 后自动写入 Bitable
- Agent 在 Review 后更新个人记忆文件
- 配置 memory-core dreaming 巩固学习
- Coordinator 读取历史记忆作为 Review 上下文

**验证标准**：
- [ ] Bitable 中有每日 Review 的完整记录
- [ ] Agent 的 MEMORY.md 有新增的学习内容
- [ ] Dreaming 正常运行
- [ ] 第二天的 Review 能引用前一天的结论

### Phase 4: 人类干预（第 11-13 天）

**目标**：支持实时干预和指令

**范围**：
- Coordinator 监听飞书群消息
- 支持回复审阅消息进行干预
- 支持 /override、/pause、/resume 等指令
- 干预结果写入 Bitable

**验证标准**：
- [ ] 老板回复审阅消息后，Coordinator 能识别并处理
- [ ] /pause 能暂停当前讨论
- [ ] 干预决策能被记录和追踪

### Phase 5: 全量上线（第 14-17 天）

**目标**：接入所有 7 个角色

**范围**：
- 补齐剩余 4 个角色（架构师、UI 设计、前端、运维）
- 各角色独立的飞书机器人应用
- 完善各角色的 AGENTS.md 和审阅视角定义
- 飞书群消息用独立机器人发送（区分角色身份）

**验证标准**：
- [ ] 7 个角色都能参与 Review
- [ ] 飞书群中每个角色有独立身份
- [ ] 全流程从 18:00 到 19:00 能跑完

### Phase 6: 优化迭代（持续）

**持续改进方向**：
- 根据实际运行调整 prompt，提高审阅质量
- 增加"专项 Review"模式（只 review 某个具体任务）
- 增加"老板发起即时 Review"（不等每日定时）
- 增加跨日知识关联（今天的问题是否和之前类似）
- 增加成长可视化（趋势图、雷达图）

---

## 8. Coordinator 核心编排逻辑（伪代码）

```python
# Coordinator 的 AGENTS.md 中定义的编排流程

AGENTS_REGISTRY = ["pm", "architect", "ui-designer", "frontend", "backend", "tester", "devops"]

def daily_review():
    """每日交叉 Review 主流程"""

    # Phase 1: 收集总结
    summaries = {}
    for agent_id in AGENTS_REGISTRY:
        response = sessions_send(
            target=agent_id,
            message=build_summary_prompt(agent_id),
            timeout_seconds=120
        )
        if response:
            summaries[agent_id] = response
            post_to_feishu(format_summary_card(agent_id, response))
        else:
            log(f"Agent {agent_id} 未响应，使用最近一次记录")
            summaries[agent_id] = fetch_last_summary_from_bitable(agent_id)

    # Phase 2: 交叉审阅
    reviews = {}
    for reviewer_id in AGENTS_REGISTRY:
        if reviewer_id not in summaries:
            continue
        # 构建审阅包：其他所有人的总结
        other_summaries = {k: v for k, v in summaries.items() if k != reviewer_id}
        review_response = sessions_send(
            target=reviewer_id,
            message=build_review_prompt(reviewer_id, other_summaries),
            timeout_seconds=180
        )
        reviews[reviewer_id] = parse_reviews(review_response)

    # Phase 3: 分发审阅 + Ping-Pong 讨论
    discussions = []
    for reviewee_id in AGENTS_REGISTRY:
        # 收集所有人对这个 Agent 的审阅
        reviews_for_agent = collect_reviews_for(reviews, reviewee_id)
        if not reviews_for_agent:
            continue

        post_to_feishu(format_reviews_card(reviewee_id, reviews_for_agent))

        # 发送给被审阅者回应
        response = sessions_send(
            target=reviewee_id,
            message=build_response_prompt(reviewee_id, reviews_for_agent),
            timeout_seconds=120
        )

        # 检查是否有需要多轮讨论的话题
        for discussion in extract_unresolved(response, reviews_for_agent):
            result = ping_pong_discussion(discussion)
            discussions.append(result)
            post_to_feishu(format_discussion_card(result))

    # Phase 4: 学习沉淀
    for agent_id in AGENTS_REGISTRY:
        sessions_send(
            target=agent_id,
            message=build_learning_prompt(agent_id, summaries, reviews, discussions),
            timeout_seconds=60
        )

    # Phase 5: 写入 Bitable + 生成报告
    write_to_bitable(summaries, reviews, discussions)
    report = generate_final_report(summaries, reviews, discussions)
    post_to_feishu(report)
```

---

## 9. 关键配置清单

### 9.1 需要创建的目录

```bash
# Agent workspace 目录
for agent in coordinator pm architect ui-designer frontend backend tester devops; do
    mkdir -p /root/.openclaw/workspace-${agent}/memory/learnings
    mkdir -p /root/.openclaw/workspace-${agent}/memory/cross-knowledge
    mkdir -p /root/.openclaw/agents/${agent}/agent
done

# Review 结果存档
mkdir -p /root/.openclaw/review-results
```

### 9.2 每个 Agent 需要的 Workspace 文件

| 文件 | 必须性 | 说明 |
|------|--------|------|
| IDENTITY.md | 必须 | 角色身份定义 |
| SOUL.md | 必须 | 角色性格（专业、直率、敢于质疑） |
| AGENTS.md | 必须 | 运行规则 + 交叉审阅视角 |
| USER.md | 必须 | 老板信息 |
| TOOLS.md | 可选 | 可用工具说明 |
| MEMORY.md | 自动创建 | 长期记忆（初始为空） |

### 9.3 openclaw.json 变更摘要

```json
{
  "agents.list": [
    // 保留现有 main 和 finance
    // 新增 coordinator, pm, architect, ui-designer, frontend, backend, tester, devops
  ],
  "channels.feishu.accounts": {
    // 保留现有 default 和 finance
    // 新增每个角色的飞书账户（或简化方案中只用 webhook）
  },
  "bindings": [
    // 保留现有 2 条
    // 新增每个角色的路由绑定
  ],
  "session": {
    "dmScope": "per-channel-peer",
    "agentToAgent": {
      "maxPingPongTurns": 3
    }
  },
  "plugins.entries": {
    // 保留现有插件
    "memory-core": {
      "enabled": true,
      "config": {
        "dreaming": {
          "enabled": true,
          "frequency": "0 3 * * *",
          "timezone": "Asia/Shanghai"
        }
      }
    }
  }
}
```

---

## 10. 总结

本方案基于 OpenClaw 平台的 `sessions_send` 机制和飞书开放平台能力，设计了：

1. **Coordinator 编排模式**：一个专职协调者管理整个 Review 流程，通过 `sessions_send` 与各角色 Agent 通信
2. **结构化交叉审阅**：强制每个 Agent 从专业角度质疑其他人，确保质疑有理有据
3. **Ping-Pong 深度讨论**：最多 3 轮的追问机制，确保问题被充分讨论
4. **分层记忆体系**：个人记忆（MEMORY.md）+ 共享记忆（Bitable）+ 梦境巩固（dreaming）
5. **飞书实时透明**：每步操作推送到飞书群，支持人类随时干预
6. **分 6 阶段落地**：从 3 个角色的最小可用版本开始，逐步扩展到全部 7 个角色
