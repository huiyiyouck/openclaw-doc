# 多 Agent 交叉学习系统架构方案 v2

> 版本：v2.2 | 日期：2026-05-20
> 基于平台：OpenClaw 2026.5.18 + 飞书开放平台 + openclaw-lark 插件
> 核心约束：**一个 Agent 对应一个飞书应用**

## 版本历史

| 版本 | 文档 | 模式 | 状态 |
|------|------|------|------|
| v1 | ~~多 Agent 交叉学习与每日总结系统架构方案.md~~ | P2P 直连 | 废弃 |
| v1.1 | ~~MULTI-AGENT-CROSS-LEARNING.md~~ | Coordinator 编排 | 废弃 |
| **v2** | 本文档 | Coordinator 编排 + 内外双通道 | 当前方案 |

v2 相比 v1.1 的核心升级：

1. **内外双通道架构**：sessions_send 做内部通信（绕过飞书过滤机器人互@），`message` 工具做外部展示（各角色以独立机器人身份在飞书群发言）
2. 修正 cron 配置方式（CLI 命令，含 --channel/--to 参数）
3. 修正插件名称（`openclaw-lark` 而非 `@tobeyoureyes/feishu`）
4. 坚持一 Agent 一飞书应用，给出完整配置
5. 补充 Bitable 并发写入限制、成本估算、审阅配对矩阵、message 工具调用方式等

---

## 1. 核心架构：内外双通道

### 1.1 为什么需要双通道

**飞书的限制**：飞书官方过滤了机器人之间的 @ 消息。如果机器人 A 在群里 @ 机器人 B，B 收不到这条消息。

**用户的需求**：飞书群里各角色以独立身份发言，看起来像真人在群里开会——每个角色有自己的头像和名称。

**解决方案**：通信和展示分两路走。

```
┌────────────────────────────────────────────────────────────────────┐
│                        飞书群（展示层）                               │
│                                                                    │
│  🎯协调者: 📢 今日交叉 Review 开始！                                  │
│  📦产品经理: 【今日总结】完成用户登录PRD... @🏗️需要确认JWT方案        │
│  🏗️架构师: 【今日总结】确定JWT鉴权方案... @📦"记住我"是产品决策      │
│  ⚙️后端: @📦 质疑：Token过期2h但未定义Refresh Token策略              │
│  📦产品经理: @⚙️ Refresh Token设为7天，PRD已补充                      │
│  🎯协调者: 📋 今日Review报告：共识5项 | 分歧1项 | 新增学习12条        │
│                                                                    │
│  → 每条消息由对应 Agent 通过 message 工具以自己机器人身份发出           │
│  → 群里看到的是各角色独立头像和名称，像真人在开会                      │
│  → @ 文本是视觉展示（人类可读），实际通信走内部通道                     │
└────────────────────────────────────────────────────────────────────┘
         ▲ message 工具（机器人身份，tenant_access_token）
         │ 各 Agent 调用 message(action="send", target="oc_xxx", channel="feishu")
         │
┌────────────────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway (内部层)                         │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  Coordinator (协调者 Agent)                                │      │
│  │  - 编排 Review 流程                                        │      │
│  │  - 通过 sessions_send 与各角色 Agent 通信（内部通道）       │      │
│  │  - 读写 Bitable 共享记忆                                   │      │
│  └──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┘      │
│         │      │      │      │      │      │      │              │
│         sessions_send（内部通信，绕过飞书过滤）                      │
│         │      │      │      │      │      │      │              │
│  ┌──────▼──┐┌──▼───┐┌─▼────┐┌▼────┐┌▼────┐┌▼────┐┌▼─────┐     │
│  │ 📦 PM   ││🏗️架构 ││🎨UI  ││💻前端││⚙️后端││🔍测试││🚀运维 │     │
│  └────┬────┘└──┬───┘└─┬────┘└┬────┘└┬────┘└┬────┘└┬─────┘     │
│       └────────┴───────┴──────┴──────┴──────┴──────┘            │
│                     各 Agent 独立 Workspace                        │
│                     MEMORY.md / memory/*.md / SOUL.md             │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  共享记忆 (飞书多维表格 Bitable)                            │      │
│  │  daily_summaries / cross_reviews / learnings              │      │
│  └──────────────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────────────┘
```

### 1.2 双通道如何协作

Coordinator 用 sessions_send 做所有内部调度，然后指示各 Agent 用 `message` 工具在飞书群中以自己身份发言。

**message 工具的关键参数**：

各 Agent（包括在 isolated session 或被 sessions_send 唤起的 session 中）通过以下方式发消息到飞书群：

```
message(
  action: "send",
  target: "oc_xxx",    // 飞书群 chat_id（推荐用 target，to 是遗留参数仍可工作）
  channel: "feishu",   // 指定飞书频道
  message: "消息内容"  // 文本内容
)
```

> **技术验证结论**：
> 1. 没有飞书频道上下文的 session（如 cron isolated session 或 sessions_send 唤起的 session）可以自由指定 `target` + `channel` 参数发消息，不受 cross-context 限制。这是因为 cross-context 检查仅在 session 已绑定频道时才生效，无绑定 session 直接跳过检查。
> 2. 有频道上下文的 session（如被老板 @ 触发的飞书消息 session）跨群发送时，`tools.message.crossContext.allowWithinProvider` 默认为 `true`，同通道（feishu → feishu）内跨群发送是允许的，无需额外配置。
> 3. **前提条件**：bot 必须已被添加到目标飞书群。即使 cross-context 检查通过，飞书 API 仍会拒绝非群成员的发送请求。

**协作流程示意**：

```
Coordinator 内部调度（sessions_send）          飞书群展示（message 工具）

1. sessions_send → PM: "请总结今天工作"         
2. PM 返回总结 → Coordinator                    📦产品经理: 【今日总结】...
3. sessions_send → PM: "请用 message 工具        ↑ PM 调用 message(action="send",
   在飞书群分享总结"                               target="oc_xxx", channel="feishu")

4. sessions_send → 后端: "请审阅PM的总结"       
5. 后端返回审阅意见 → Coordinator               ⚙️后端: @📦 质疑：Token策略...
6. sessions_send → 后端: "请用 message 工具      ↑ 后端调用 message 工具
   在飞书群分享审阅"

7. sessions_send → PM: "请回应后端的审阅"       
8. PM 返回回应 → Coordinator                    📦产品经理: @⚙️ Refresh Token 7天...
9. sessions_send → PM: "请用 message 工具        ↑ PM 调用 message 工具
   在飞书群分享回应"
```

**关键点**：
- sessions_send 是"大脑"——结构化数据交换、流程编排、Bitable 读写
- `message` 工具是"声音"——各角色以独立身份在飞书群发言
- 飞书群里的 @ 是**视觉展示**（人类看到谁在对谁说话），不依赖飞书的 @ 通知机制
- **session 生命周期**：同一 Agent 的多次 sessions_send 共享同一 session 上下文（dmScope 下按 source agent 隔离）。例如 Coordinator 向 PM 发两次 sessions_send，PM 在第二次时仍记得第一次的对话。因此 Coordinator 指示飞书群发言时**无需重复传递**该 Agent 之前已生成的总结/审阅内容
- 实际的数据流向由 sessions_send 控制，不受飞书过滤影响

### 1.3 交互模式说明：手动编排 vs 自动 Ping-Pong

本方案采用 **Coordinator 手动编排** 模式，不依赖自动 Ping-Pong 机制。

- **自动 Ping-Pong**：Agent A 发 sessions_send 给 Agent B，B 回复后自动触发 A 继续回复，直到轮次耗尽。这种方式流程不受 Coordinator 精确控制。
- **手动编排**：Coordinator 每轮主动发 sessions_send，收集回应后再决定是否继续。Coordinator 完全控制流程、超时、跳过等。

本方案选择手动编排的原因：
1. Coordinator 需要在每轮之间做逻辑判断（是否达成共识、是否需要升级）
2. 每轮结果需要推送到飞书群，自动 Ping-Pong 无法插入这个步骤
3. 手动编排支持跳过缺席 Agent、提前结束等灵活操作

因此 `session.agentToAgent.maxPingPongTurns` 设为 `0`（禁用自动乒乓），所有多轮交互由 Coordinator 手动驱动。

### 1.4 Agent 角色定义

| Agent ID | 角色 | Emoji | 默认模型 | 职责 |
|----------|------|-------|---------|------|
| `coordinator` | 协调者 | 🎯 | deepseek/deepseek-v4-pro | 编排 Review、调度通信、读写 Bitable |
| `pm` | 产品经理 | 📦 | volcengine-plan/glm-5.1 | 需求分析、PRD、优先级、用户故事 |
| `architect` | 架构师 | 🏗️ | volcengine-plan/glm-5.1 | 系统设计、技术选型、接口规范 |
| `ui-designer` | UI 设计 | 🎨 | volcengine-plan/glm-5.1 | 交互设计、视觉规范、组件库 |
| `frontend` | 前端开发 | 💻 | volcengine-plan/glm-5.1 | 页面实现、组件开发、性能优化 |
| `backend` | 后端开发 | ⚙️ | volcengine-plan/glm-5.1 | API 实现、数据模型、业务逻辑 |
| `tester` | 测试工程师 | 🔍 | volcengine-plan/glm-5.1 | 测试用例、质量标准、缺陷分析 |
| `devops` | 部署运维 | 🚀 | volcengine-plan/glm-5.1 | CI/CD、部署策略、监控告警 |

> Coordinator 用更强模型（deepseek-v4-pro），编排逻辑需要更强推理能力。
> 角色 Agent 用 glm-5.1（200K 上下文、128K 输出），性价比高。
> 现有的 main 和 finance Agent 不参与 Review 流程，不加入 agentToAgent.allow 列表。

### 1.5 核心设计原则

1. **Coordinator 编排**：唯一调度中心，角色 Agent 不直接通信
2. **sessions_send = 内部大脑**：所有 Agent 间数据交换走内部通道，不受飞书过滤
3. **message 工具 = 外部声音**：各 Agent 以独立机器人身份在飞书群发言
4. **分层记忆**：个人记忆（MEMORY.md）+ 共享记忆（Bitable）+ 梦境巩固（dreaming）
5. **手动编排**：Coordinator 完全控制每轮交互，不依赖自动 Ping-Pong

---

## 2. 核心交互流程

### 2.1 每日交叉 Review 全流程

```
18:00  Cron 触发 → Coordinator 启动 Review
 │
 ├─ Phase 1: 总结收集 (18:00 - 18:10)
 │   Coordinator 并行 sessions_send 向各角色收集总结
 │   各角色生成总结后，Coordinator 指示它们用 message 工具发到飞书群
 │
 ├─ Phase 2: 交叉审阅 (18:10 - 18:30)
 │   Coordinator 将总结通过 sessions_send 分发给审阅者
 │   每人只审阅 2-3 个最相关的角色（见审阅配对矩阵）
 │   审阅者生成意见后，Coordinator 指示它们用 message 工具发到飞书群
 │
 ├─ Phase 3: 交叉讨论 (18:30 - 18:50)
 │   Coordinator 通过 sessions_send 管理多轮讨论（手动编排，最多 3 轮）
 │   每轮结果指示角色用 message 工具发到飞书群
 │
 ├─ Phase 4: 学习沉淀 (18:50 - 19:00)
 │   各 Agent 更新个人记忆（直接写 MEMORY.md，不等 dreaming）
 │   Coordinator 异步写入 Bitable
 │
 └─ Phase 5: 总结报告 (19:00)
     Coordinator 生成飞书卡片报告
```

### 2.2 飞书群中的实际视觉效果

```
🎯协调者: 📢 今日交叉 Review 开始！参与：PM, 架构师, UI, 前端, 后端, 测试, 运维

📦产品经理: 【今日总结 2026-05-20】
           ✅ 完成用户登录功能 PRD，含手机号+验证码、邮箱+密码
           🔑 决策：放弃 OAuth（MVP 减少集成复杂度），Token 过期 2h
           ❓ 不确定是否需要"记住我"，需要 @🏗️架构师 讨论
           ➡️ 对别人的影响：@⚙️后端 需实现验证码服务 @🎨UI 需设计验证码输入交互

🏗️架构师: 【今日总结 2026-05-20】
          ✅ 确定 JWT + Refresh Token 鉴权方案
          🔑 决策：API Gateway 统一验签，服务间不鉴权（MVP 简化）
          ❓ @📦产品经理 "记住我"应该是产品决策，不是技术决策

⚙️后端: @📦产品经理 质疑：Token 过期 2h，但 Refresh Token 策略呢？
        如果也是 2h，用户每 2h 重新登录。建议明确。
        💡 建议增加登录安全策略章节

📦产品经理: @⚙️后端 Refresh Token 设为 7 天，PRD 已补充。

🏗️架构师: @📦产品经理 MVP 先不做"记住我"，PRD 标注为 P1（下迭代必须）。

🎯协调者: 📋 今日 Review 报告
          ✅ 达成共识 5 项 | ⚠️ 分歧 0 项 | 📝 新增学习 12 条
```

### 2.3 审阅配对矩阵

为避免全交叉审阅（7×6=42 条审阅消息导致飞书群消息爆炸），每个角色只审阅 2-3 个最相关的角色：

| 审阅者 | 审阅对象 | 审阅理由 |
|--------|---------|---------|
| PM | 架构师、后端、UI | 产品需求驱动架构和 API 设计，交互依赖 UI |
| 架构师 | PM、后端、前端、运维 | 技术选型受需求驱动，架构规范约束实现，部署方案受架构约束 |
| UI 设计 | PM、前端 | 设计基于需求，落地依赖前端 |
| 前端 | UI 设计、后端、运维 | 实现依赖设计稿和 API，部署产物与运维相关 |
| 后端 | 架构师、PM、测试 | 实现遵循架构规范，API 服务产品需求，API 质量影响测试策略 |
| 测试 | PM、后端、前端、运维 | 测试覆盖需求和实现，测试环境依赖运维 |
| 运维 | 架构师、后端 | 部署受架构约束，运行时依赖后端 |

Phase 1（3 角色：PM、后端、测试）的配对：

| 审阅者 | 审阅对象 |
|--------|---------|
| PM | 后端 |
| 后端 | PM |
| 测试 | PM、后端 |

### 2.4 Phase 1 详细：总结收集

**Step 1 - Cron 触发**

```bash
# --channel feishu --to "oc_xxx" 指定投递目标，--announce 启用频道投递
# oc_xxx 替换为实际的飞书群 chat_id（获取方式：在飞书群设置中查看群ID）
# 注意：cron CLI 用 --to，message 工具用 target=（同一 chat_id，不同层面的参数名）
openclaw cron add \
  --name "每日交叉Review" \
  --cron "0 18 * * 1-5" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --agent coordinator \
  --message "执行每日交叉 Review 流程。" \
  --announce \
  --channel feishu \
  --to "oc_xxx"
```

**Step 2 - Coordinator 并行收集总结**

Coordinator 向每个角色 Agent 发送：

```
今天是 2026-05-20。请总结你今天参与的工作。

格式要求：
1.【我做了什么】具体完成的事项
2.【我的决策】关键决策及理由
3.【遇到的困难】阻碍或不确定的点
4.【对别人的影响】你的工作对其他角色的影响（@对应角色标注）
5.【明天的计划】

基于你的 workspace 记忆文件组织回答。每项不超过 2 句话。
```

> sessions_send 支持并行发送——Coordinator 可同时向 7 个 Agent 发送收集请求，然后分别等待回应，大幅缩短收集时间。

`sessions_send` 设置 `timeoutSeconds: 120`。超时未回复的标记为"缺席"，从 Bitable 读最近一次总结替代。

**Step 3 - Coordinator 指示各角色在飞书群发言**

Coordinator 收到总结后，再发一条 sessions_send：

```
请在飞书群中以你的身份分享你刚才的总结。
使用 message 工具：action="send", target="oc_xxx", channel="feishu"
```

> **注意**：同一 Agent 的 sessions_send 共享 session 上下文，Agent 记得自己刚生成的总结，无需重复传递内容。

### 2.5 Phase 2 详细：交叉审阅

Coordinator 按审阅配对矩阵，只将相关角色的总结发给审阅者：

```
请从【你的角色】角度，审阅以下团队成员今日总结。

对每个总结，你需要填写：

| 维度 | 要求 |
|------|------|
| 引用 | 具体引用对方的哪句话或哪个决策 |
| 类型 | 认同 / 质疑 / 追问 / 建议 |
| 理由 | 为什么认同或质疑，你的依据 |
| 影响 | 如果不解决，会产生什么后果 |

规则：
- 如发现值得质疑的点，至少提出 1 个；如确实无问题，可全部认同并说明理由
- 质量优先于数量，不要为了凑数而编造质疑
- 每个质疑必须有"影响"说明

--- 以下是你需要审阅的角色总结（原样传递，未经摘要）---
{relevant_summaries_per_pairing_matrix}
```

审阅者返回结果后，Coordinator 指示它们用 `message` 工具发到飞书群。

### 2.6 Phase 3 详细：交叉讨论

Coordinator 将针对某 Agent 的审阅意见汇总，发给该 Agent 回应。回应通过 sessions_send 返回 Coordinator，Coordinator 再指示该 Agent 用 `message` 工具发到飞书群。

讨论由 Coordinator 手动编排（最多 3 轮）：
1. **Round 1**：审阅者提问 → 被审阅者回应
2. **Round 2**：审阅者追问 → 被审阅者再次回应
3. **Round 3**：最终确认（达成共识 / 标记分歧 / 升级给老板）

每轮 Coordinator 判断是否有未解决的质疑，决定是否继续。

### 2.7 Phase 4 详细：学习沉淀

Coordinator 要求每个 Agent 更新记忆：

```
今日 Review 已完成。请执行以下记忆更新：

1. 在 memory/{today}.md 添加"交叉学习"章节
2. 如有值得长期记住的认知，直接更新 MEMORY.md（不等 dreaming）
3. 如发现了知识盲区，记录在 memory/{today}.md 中
```

> Agent 在 Review 结束后直接更新 MEMORY.md，dreaming 只做排序和巩固，不做首次写入。

Coordinator 在 Phase 5 报告生成后异步写入 Bitable，不阻塞主流程。

### 2.8 Phase 5 详细：飞书群报告

Coordinator 生成最终报告卡片发送到飞书群。卡片内容包括：
- 各角色今日工作摘要
- 关键讨论和结论
- 达成共识 / 仍有分歧 / 需老板裁决
- 今日学习摘要

---

## 3. Agent Prompt 设计

### 3.1 Coordinator 的 AGENTS.md（核心节选）

```markdown
# Coordinator 运行手册

## 角色
你是团队协调者，负责编排每日交叉 Review 流程。

## 双通道工作方式

你同时使用两个通道：
1. **sessions_send**：与各角色 Agent 的内部通信。所有数据收集、分发、讨论协调都走这个通道。
   - 参数：targetSession 指向 agent:<agentId>，timeoutSeconds: 120
2. **message 工具**：指示各角色 Agent 在飞书群中以自己身份发言。
   - 你自己也用 message 工具发送开场和报告消息。

## 编排流程

当收到"执行每日交叉 Review"指令时，按以下步骤操作：

### 第一步：收集总结
同时向所有角色 Agent 发送总结请求（并行 sessions_send）。
收集所有回复，记录已响应和缺席的 Agent。
超时 120 秒未响应的标记为"缺席"，从 Bitable 读取最近总结。

### 第二步：展示总结
用 message 工具在飞书群发开场消息（action="send", target="oc_xxx", channel="feishu"）。
然后逐个指示各角色在飞书群发言（串行，每个角色发完后再指示下一个，保证消息顺序可控）：
发一条 sessions_send："请用 message 工具在飞书群分享你的总结：action='send', target='oc_xxx', channel='feishu'"
（Agent 共享 session 上下文，记得自己的总结，无需重复传递）

### 第三步：交叉审阅
按审阅配对矩阵，将相关角色的总结发给审阅者。
收集审阅意见后，指示各角色用 message 工具在飞书群分享。

### 第四步：交叉讨论
将针对每个角色的审阅意见汇总，发给该角色回应。
如有未解决的质疑，手动发起下一轮追问。最多 3 轮。
每轮结果都指示角色用 message 工具发到飞书群。

### 第五步：学习沉淀
要求各 Agent 更新个人记忆。
你负责将总结、审阅、讨论结果写入 Bitable（异步，串行，每条间隔 1 秒）。

### 第六步：生成报告
生成飞书卡片发送到群。

## 自检逻辑
每完成一个 Phase，校验已收到的回应数量是否与注册表一致。
不一致时在飞书群标注缺席者。

## 角色 Agent 注册表
pm, architect, ui-designer, frontend, backend, tester, devops

## 审阅配对矩阵
PM → 架构师、后端、UI
架构师 → PM、后端、前端、运维
UI设计 → PM、前端
前端 → UI设计、后端、运维
后端 → 架构师、PM、测试
测试 → PM、后端、前端、运维
运维 → 架构师、后端

## 关键参数
飞书群 chat_id: oc_xxx（需替换为实际值，在飞书群设置中查看群ID）
Bitable app_token: xxx（需替换为实际值）
Bitable 表名：daily_summaries, cross_reviews, learnings
（table_id 通过 feishu_bitable_app_table.list 动态获取，无需硬编码）
sessions_send timeoutSeconds: 120

## Bitable 操作步骤
1. 用 `feishu_bitable_app_table.list`（action="list", app_token=xxx）获取各表 table_id
2. 写入前用 `feishu_bitable_app_table_field.list`（action="list", app_token=xxx, table_id=xxx）确认字段类型
3. 查询记录用 `feishu_bitable_app_table_record.list`（action="list", app_token=xxx, table_id=xxx, filter=...）
4. 写入记录用 `feishu_bitable_app_table_record.create`（action="create", app_token=xxx, table_id=xxx, fields=...）
5. 同一表串行写入，每条间隔 1 秒
6. 第一次运行时 Bitable 为空，缺席 Agent 用"首次运行，暂无历史总结"代替

## 人类干预处理
当老板在飞书群中 @ 你发送指令时：
- /override <内容> — 用老板的决策覆盖当前讨论，通知所有相关 Agent
- /skip — 跳过当前讨论话题，继续下一个
- /ask <内容> — 将老板的问题转发给指定角色 Agent

注意：/pause 和 /resume 在 sessions_send 编排模式下难以实现
（无法暂停正在等待的 sessions_send 回应），暂不支持。
```

### 3.2 角色 Agent 的 AGENTS.md 模板

每个角色 Agent 的 AGENTS.md 包含：通用规则、角色审阅视角、两种输出格式。

**通用规则（所有角色共用）**

```markdown
# 通用协作规则

1. 收到 Coordinator 的 sessions_send 消息时，在 timeout 内响应
2. 技术问题给出具体建设性答复，避免敷衍
3. 不确定时主动说明，不要猜测
4. 每次协作中的收获写入 memory/{today}.md
5. 仅在 Coordinator 指示时才调用 message 工具向飞书群发言，避免误发

# 角色边界规则

1. 不要越界——PM 不写代码，开发不编需求
2. 但要理解——从其他角色视角思考
3. 质疑是权力——对任何角色产出可提出建设性质疑
4. 决策有流程——技术决策架构师拍板，产品决策 PM 拍板，老板有最终否决权

# 输出格式区分

根据 Coordinator 消息内容决定输出格式：
- 如果 Coordinator 要求"总结/审阅/回应"→ 内部回应格式（结构化）
- 如果 Coordinator 要求"在飞书群分享/发言"→ 飞书发言格式（自然语言）

## 内部回应格式（sessions_send 回复 Coordinator）
使用结构化格式，方便 Coordinator 解析：
- 类型：质疑/认同/追问/建议
- 引用：对方的具体语句
- 内容：你的审阅意见
- 影响：如果不解决的后果

## 飞书发言格式（message 工具发群）
自然语言，像真人在群里说话：
@⚙️ 质疑：Token 过期 2h 但没定义 Refresh Token 策略。
如果不明确，用户可能每 2h 被迫重新登录。

# 飞书群发言规则

当 Coordinator 要求你在飞书群分享内容时：
1. 使用 message 工具：action="send", target="oc_xxx", channel="feishu"
2. 消息中用 @ 标注你提到的其他角色
3. 保持自然、专业的语气，像真人在群里开会
4. 格式清晰，用 emoji 标记重点
```

**角色审阅视角（各 Agent 不同）**

PM 的审阅视角：

```markdown
## 交叉审阅视角

### 对架构师
- 系统设计是否覆盖了所有用户场景？
- 技术选型是否与产品优先级对齐？
- MVP 阶段是否过度工程化？

### 对后端开发
- API 设计是否满足前端交互需求？
- 错误处理是否考虑了用户侧体验？

### 对 UI 设计
- 设计方案是否覆盖了所有状态（空态、加载态、错误态）？
```

架构师的审阅视角：

```markdown
## 交叉审阅视角

### 对产品经理
- 需求定义是否足够明确，能直接指导技术设计？
- 是否存在隐含的技术复杂度未暴露？

### 对后端开发
- 实现方案是否符合架构规范？
- 接口设计是否遵循统一规范？

### 对前端开发
- 组件设计是否与架构分层一致？
- 状态管理方案是否可扩展？

### 对运维
- 部署方案是否考虑了回滚和灰度？
```

### 3.3 SOUL.md 设计

SOUL.md 定义"你是谁"，AGENTS.md 定义"你怎么做事"，两者分离。

**架构师**

```markdown
# SOUL.md - 架构师

## 核心特质
- 技术精湛，擅长系统架构设计
- 重视可扩展性和可维护性
- 从全局视角理解各角色工作的相互影响

## 沟通风格
- 直接严谨，结构化表达
- 对不确定的问题主动追问
- 不做无根据的推测
```

**产品经理**

```markdown
# SOUL.md - 产品经理

## 核心特质
- 擅长需求分析和用户洞察
- 重视产品价值和优先级判断
- 能从业务视角评估技术方案的合理性

## 沟通风格
- 用用户故事和数据说话
- 对模糊的需求主动追问边界
- 不把产品决策推给技术
```

**后端开发**

```markdown
# SOUL.md - 后端开发

## 核心特质
- 擅长 API 设计和数据建模
- 重视系统安全性和性能
- 从实现可行性角度评估需求

## 沟通风格
- 关注细节，给出具体的技术方案
- 对不明确的接口需求主动追问
- 用数据和边界条件论证
```

---

## 4. 记忆系统

### 4.1 分层架构

```
┌─────────────────────────────────────┐
│  Layer 1: 短期记忆（Session 内）      │
│  当天讨论内容、当前任务上下文          │
│  会话结束即消散                       │
└─────────────────────────────────────┘
         │ Review 后直接写入 + dreaming 整理
         ▼
┌─────────────────────────────────────┐
│  Layer 2: 个人记忆（Agent Workspace） │
│  MEMORY.md: 长期认知和经验法则        │
│  memory/{date}.md: 每日工作记录       │
│  Review 后直接更新，不等 dreaming     │
│  仅该 Agent 可见和修改               │
└─────────────────────────────────────┘
         │ Coordinator 归档
         ▼
┌─────────────────────────────────────┐
│  Layer 3: 共享记忆（飞书 Bitable）    │
│  每日总结、交叉评审记录、学习笔记      │
│  所有 Agent 可读，Coordinator 写入    │
└─────────────────────────────────────┘
```

> MEMORY.md 的写入有两个时机：(1) Review 结束后 Agent 直接更新（首次写入）；(2) 凌晨 dreaming 整理巩固（排序和提升）。两者不矛盾。

### 4.2 Dreaming 梦境巩固

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

每天凌晨 3:00 自动整理：
- **Light**：排序短期素材（不写 MEMORY.md）
- **Deep**：评分并晋升持久候选项（写入 MEMORY.md，与 Review 后的手动写入共存）
- **REM**：反思主题和反复出现的想法

### 4.3 Bitable 共享记忆表结构

> patterns 表推迟到 Phase 6 实现。Phase 0-5 只建 daily_summaries、cross_reviews、learnings 三张表。

**daily_summaries（每日总结）**

| 字段 | 类型 | 说明 |
|------|------|------|
| date | 日期 | 毫秒时间戳 |
| agent_id | 文本 | Agent 标识 |
| what_i_did | 文本 | 我做了什么 |
| decisions | 文本 | 关键决策 |
| difficulties | 文本 | 遇到的困难 |
| impact_on_others | 文本 | 对别人的影响 |
| tomorrow_plan | 文本 | 明天的计划 |

**cross_reviews（交叉评审）**

| 字段 | 类型 | 说明 |
|------|------|------|
| date | 日期 | 毫秒时间戳 |
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

### 4.4 Bitable 写入注意事项

1. **写入前必须先读取字段类型**：调用 `feishu_bitable_app_table_field.list`
2. **同一表不支持并发写**：必须串行，每条记录间隔 0.5-1 秒
3. **日期字段用毫秒时间戳**
4. **单选字段用字符串**（直接传选项名）
5. **批量操作单次不超过 500 条**
6. **异步写入**：Phase 4 标记写入任务，Phase 5 报告生成后 Coordinator 在后台写入 Bitable，不阻塞主流程

### 4.5 记忆一致性

当 Agent MEMORY.md 与 Bitable 记录矛盾时：
- 不硬性规定"以谁为准"
- Coordinator 仲裁并通知相关 Agent 更新 MEMORY.md
- Review 开始前，Coordinator 从 Bitable 读取上次结论摘要，通过 sessions_send 注入各 Agent 上下文

---

## 5. 飞书群交互与人工干预

### 5.1 消息推送节奏

```
18:00  🎯协调者: 📢 今日交叉 Review 开始！

18:03  📦产品经理: 【今日总结】...
18:05  🏗️架构师: 【今日总结】...
18:07  🎨UI设计: 【今日总结】...
18:09  💻前端: 【今日总结】...
18:11  ⚙️后端: 【今日总结】...
18:13  🔍测试: 【今日总结】...
18:15  🚀运维: 【今日总结】...

18:17  🎯协调者: 🔥 交叉审阅开始

18:20  ⚙️后端: @📦 质疑：Token过期2h的Refresh Token策略？
18:22  🏗️架构师: @📦 "记住我"应该是产品决策
18:25  📦产品经理: @⚙️ Refresh Token 7天，PRD已补充
18:27  📦产品经理: @🏗️ 同意，PRD已标注为P1

18:50  🎯协调者: 📝 学习阶段

18:55  🎯协调者: 📋 今日Review报告
```

### 5.2 人类干预机制

**方式一：@ Coordinator 发送指令**

```
@🎯 /override Token 过期策略改为 24 小时
@🎯 /skip
@🎯 /ask 后端：验证码服务调研结果
```

> /pause 和 /resume 在 sessions_send 编排模式下难以实现（无法暂停正在等待的 sessions_send 回应），暂不支持。如需暂停，直接停止 Gateway 即可。

**方式二：@ 特定角色 Agent**

```
@⚙️ 验证码服务的调研结果给我看看
```

该 Agent 收到飞书消息后直接在群里回复（这是用户 @ 机器人，不是机器人互 @，所以不被过滤）。

### 5.3 信息可见性

- **飞书群**：所有总结、审阅、讨论、报告公开可见
- **sessions_send**：内部消息不进群聊（内容由各 Agent 通过 message 工具推送摘要到群）
- **Bitable**：归档数据，老板可随时查看
- **MEMORY.md**：仅该 Agent 可见

---

## 6. 完整配置清单

### 6.1 目录结构

```bash
for agent in coordinator pm architect ui-designer frontend backend tester devops; do
    mkdir -p /root/.openclaw/workspace-${agent}/memory
    mkdir -p /root/.openclaw/agents/${agent}/agent
done
```

### 6.2 openclaw.json 配置变更

以下是需要新增到 `openclaw.json` 的配置（保留现有的 main 和 finance Agent）：

```json
{
  "agents": {
    "list": [
      { "id": "main", "..." : "现有配置不变" },
      { "id": "finance", "...": "现有配置不变" },
      {
        "id": "coordinator",
        "name": "协调者",
        "workspace": "/root/.openclaw/workspace-coordinator",
        "agentDir": "/root/.openclaw/agents/coordinator/agent",
        "model": "deepseek/deepseek-v4-pro",
        "identity": { "name": "协调者", "emoji": "🎯" }
      },
      {
        "id": "pm",
        "name": "产品经理",
        "workspace": "/root/.openclaw/workspace-pm",
        "agentDir": "/root/.openclaw/agents/pm/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "产品经理", "emoji": "📦" }
      },
      {
        "id": "architect",
        "name": "架构师",
        "workspace": "/root/.openclaw/workspace-architect",
        "agentDir": "/root/.openclaw/agents/architect/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "架构师", "emoji": "🏗️" }
      },
      {
        "id": "ui-designer",
        "name": "UI设计",
        "workspace": "/root/.openclaw/workspace-ui-designer",
        "agentDir": "/root/.openclaw/agents/ui-designer/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "UI设计", "emoji": "🎨" }
      },
      {
        "id": "frontend",
        "name": "前端开发",
        "workspace": "/root/.openclaw/workspace-frontend",
        "agentDir": "/root/.openclaw/agents/frontend/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "前端开发", "emoji": "💻" }
      },
      {
        "id": "backend",
        "name": "后端开发",
        "workspace": "/root/.openclaw/workspace-backend",
        "agentDir": "/root/.openclaw/agents/backend/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "后端开发", "emoji": "⚙️" }
      },
      {
        "id": "tester",
        "name": "测试工程师",
        "workspace": "/root/.openclaw/workspace-tester",
        "agentDir": "/root/.openclaw/agents/tester/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "测试工程师", "emoji": "🔍" }
      },
      {
        "id": "devops",
        "name": "部署运维",
        "workspace": "/root/.openclaw/workspace-devops",
        "agentDir": "/root/.openclaw/agents/devops/agent",
        "model": "volcengine-plan/glm-5.1",
        "identity": { "name": "部署运维", "emoji": "🚀" }
      }
    ]
  },
  "session": {
    "dmScope": "per-channel-peer",
    "agentToAgent": {
      "maxPingPongTurns": 0
    }
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "domain": "feishu",
      "connectionMode": "websocket",
      "requireMention": false,
      "streaming": true,
      "accounts": {
        "default": {},
        "finance": { "..." : "现有配置不变" },
        "coordinator": {
          "enabled": true, "name": "协调者",
          "appId": "cli_coordinator_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "pm": {
          "enabled": true, "name": "产品经理",
          "appId": "cli_pm_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "architect": {
          "enabled": true, "name": "架构师",
          "appId": "cli_architect_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "ui-designer": {
          "enabled": true, "name": "UI设计",
          "appId": "cli_ui_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "frontend": {
          "enabled": true, "name": "前端开发",
          "appId": "cli_frontend_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "backend": {
          "enabled": true, "name": "后端开发",
          "appId": "cli_backend_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "tester": {
          "enabled": true, "name": "测试工程师",
          "appId": "cli_tester_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        },
        "devops": {
          "enabled": true, "name": "部署运维",
          "appId": "cli_devops_app_id", "appSecret": "<secret>",
          "domain": "feishu", "connectionMode": "websocket",
          "requireMention": false, "streaming": true
        }
      }
    }
  },
  "bindings": [
    { "type": "route", "agentId": "main", "match": { "channel": "feishu", "accountId": "default" } },
    { "type": "route", "agentId": "finance", "match": { "channel": "feishu", "accountId": "finance" } },
    { "type": "route", "agentId": "coordinator", "match": { "channel": "feishu", "accountId": "coordinator" } },
    { "type": "route", "agentId": "pm", "match": { "channel": "feishu", "accountId": "pm" } },
    { "type": "route", "agentId": "architect", "match": { "channel": "feishu", "accountId": "architect" } },
    { "type": "route", "agentId": "ui-designer", "match": { "channel": "feishu", "accountId": "ui-designer" } },
    { "type": "route", "agentId": "frontend", "match": { "channel": "feishu", "accountId": "frontend" } },
    { "type": "route", "agentId": "backend", "match": { "channel": "feishu", "accountId": "backend" } },
    { "type": "route", "agentId": "tester", "match": { "channel": "feishu", "accountId": "tester" } },
    { "type": "route", "agentId": "devops", "match": { "channel": "feishu", "accountId": "devops" } }
  ],
  "tools": {
    "profile": "full",
    "web": { "search": { "provider": "tavily", "enabled": true } },
    "agentToAgent": {
      "enabled": true,
      "allow": ["coordinator", "pm", "architect", "ui-designer", "frontend", "backend", "tester", "devops"]
    }
  },
  "plugins": {
    "entries": {
      "volcengine": { "enabled": true },
      "deepseek": { "enabled": true },
      "tavily": { "enabled": true },
      "openclaw-lark": { "enabled": true },
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 3 * * *",
            "timezone": "Asia/Shanghai"
          }
        }
      },
      "memory-wiki": { "enabled": true }
    }
  }
}
```

**配置要点**：
- `tools.agentToAgent`（**新增字段，待验证**）：enabled + allow 控制哪些 Agent 可以发起 A2A 通信。当前 openclaw.json 中尚无此字段，需确认平台版本是否支持
- `session.agentToAgent`（**新增字段，待验证**）：maxPingPongTurns 设为 0，禁用自动乒乓，本方案使用 Coordinator 手动编排。同上，需确认平台支持
- main 和 finance 不在 allow 列表中，它们不参与 Review 流程
- 新增飞书账户均显式设置 `requireMention: false`，与现有 finance 账户写法一致
- 现有 `messages.groupChat.visibleReplies: "message_tool"` 保持不变，确保 Agent 在群聊中必须调用 message 工具才能发布可见回复（与方案一致）

### 6.3 每个 Agent 需要的 Workspace 文件

| 文件 | 必须性 | 说明 |
|------|--------|------|
| IDENTITY.md | 必须 | 角色身份（名称、emoji、风格） |
| SOUL.md | 必须 | 角色性格 |
| AGENTS.md | 必须 | 运行规则 + 交叉审阅视角 + 飞书发言规则 |
| USER.md | 必须 | 老板信息 |
| TOOLS.md | 可选 | 可用工具说明 |
| MEMORY.md | 自动创建 | 长期记忆（初始为空） |

单文件不超过 12000 字符，所有 bootstrap 文件总计不超过 60000 字符。

### 6.4 Cron 定时任务

```bash
# --channel feishu --to "oc_xxx" 指定投递目标
# oc_xxx 替换为实际的飞书群 chat_id
# 注意：cron CLI 用 --to，message 工具用 target=（同一 chat_id，不同层面的参数名）
openclaw cron add \
  --name "每日交叉Review" \
  --cron "0 18 * * 1-5" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --agent coordinator \
  --message "执行每日交叉 Review 流程。" \
  --announce \
  --channel feishu \
  --to "oc_xxx"
```

> 获取飞书群 chat_id 的方式：在飞书群设置中查看群ID，或通过 `feishu_chat.search` 工具搜索群名获取。

### 6.5 飞书开放平台操作清单

为每个角色创建一个自建应用：

1. 登录飞书开放平台 → 创建自建应用
2. 设置应用名称和头像（对应角色形象）
3. 开通权限（见下方权限清单）
4. 订阅方式：选择"使用长连接接收事件/回调"
5. 获取 App ID 和 App Secret，填入 openclaw.json
6. 将所有应用添加到同一个飞书群

**权限清单**：

| 权限 | 用途 | 谁需要 | 是否需审批 |
|------|------|--------|-----------|
| im:message | 接收消息 | 所有应用 | 否 |
| im:message:send_as_bot | 主动发消息到群 | 所有应用 | 否 |
| im:message.receive_v1 | 消息事件订阅 | 所有应用 | 否 |
| im:chat | 获取群信息 | 所有应用 | 否 |
| bitable:app:read | 读多维表格 | Coordinator | 可能 |
| bitable:app:write | 写多维表格 | Coordinator | 可能 |

> 部分权限（如 bitable）可能需要飞书管理员审批。建议提前申请。

---

## 7. 异常处理

### 7.1 Agent 不可用

| 场景 | 处理 |
|------|------|
| sessions_send 超时（120s） | Coordinator 跳过该 Agent，标注"缺席"，继续流程 |
| Agent 返回错误 | 记录错误，使用 Bitable 中最近一次有效总结 |
| 多个 Agent 同时不可用 | 继续可用 Agent 之间的讨论，标注缺席者 |

### 7.2 Coordinator 自身异常

| 场景 | 处理 |
|------|------|
| 模型调用失败（429/超时） | Review 中断，Coordinator 在下次 cron 触发时重新开始 |
| session 中途崩溃 | 已收集数据丢失，重新开始本次 Review。崩溃恢复机制（review_sessions 表）推迟到 Phase 6 实现 |
| prompt 错误（漏发某 Agent） | Coordinator 的自检逻辑每 Phase 校验回应数量，发现遗漏时补发 |

### 7.3 讨论僵局

| 场景 | 处理 |
|------|------|
| 3 轮讨论后仍无共识 | 自动升级，Coordinator 在飞书群请老板裁决 |
| 两个 Agent 冲突 | Coordinator 生成摘要对比，请老板决策 |
| 讨论偏离主题 | Coordinator 介入提醒 |

### 7.4 模型调用失败

| 场景 | 处理 |
|------|------|
| API 限流 (429) | 指数退避重试，最多 3 次。注意：Phase 1 并行收集时 7 个 Agent 同时发起推理请求，若 provider 并发受限，可分 2-3 批并行（每批 3 个） |
| 上下文超限 | Coordinator 压缩历史摘要后重试 |
| 模型幻觉 | 交叉审阅本身是纠错机制，多角色质疑可发现幻觉 |

### 7.5 Bitable 写入失败

| 场景 | 处理 |
|------|------|
| API 调用失败 | 重试 3 次，间隔 5 秒 |
| 持续失败 | 写入本地 `review-results/{date}.json`，标记待发送，下次 Review 前同步 |

### 7.6 飞书群消息发送失败

| 场景 | 处理 |
|------|------|
| message 工具调用失败 | Coordinator 检测后，将消息暂存本地，重试发送 |
| token 过期/权限不足 | Coordinator 在飞书群以自己身份代发，标注原角色 |
| 群 ID 错误 | 提醒老板检查群 ID 配置 |

### 7.7 飞书 WebSocket 连接异常

10 个飞书应用同时维持 WebSocket 长连接，任何一个断开后 openclaw-lark 会自动重连。如多个连接同时断开：
- 检查 `journalctl --user -u openclaw-gateway` 日志
- 验证飞书应用状态和权限
- 必要时重启 gateway

### 7.8 记忆一致性

| 场景 | 处理 |
|------|------|
| Agent MEMORY.md 与 Bitable 矛盾 | Coordinator 仲裁并通知相关 Agent 更新 |
| 梦境整理误删重要记忆 | workspace 纳入 git 管理，可回滚 |
| 新 Agent 加入缺乏历史 | Coordinator 从 Bitable 生成摘要包注入 |

---

## 8. 成本估算

### 8.1 每日 Review Token 消耗（粗估，按配对矩阵审阅）

| 阶段 | 7 个角色 Agent | Coordinator | 合计估算 |
|------|---------------|-------------|---------|
| 总结收集 | 7 × ~2K in + ~1K out | ~2K in | ~16K |
| 交叉审阅 | 7 × ~3K in + ~1.5K out | ~4K in | ~32K |
| 交叉讨论 | 7 × ~3K in + ~2K out (×3轮) | ~10K in (×3轮) | ~121K |
| 学习沉淀 | 7 × ~2K in + ~1K out | ~2K in | ~23K |
| 报告生成 | — | ~5K in + ~3K out | ~8K |
| **单日合计** | | | **~200K tokens** |

### 8.2 月度成本（22 个工作日）

| 模型 | 单价 | 月成本 |
|------|------|--------|
| glm-5.1（7 角色） | 约 ¥0.5/M tokens | ~¥1.5 |
| deepseek-v4-pro（Coordinator） | 约 ¥2/M tokens | ~¥1.3 |
| **合计** | | **~¥2.8/月** |

---

## 9. 分阶段落地计划

每个 Phase 开始前做配置快照：`cp openclaw.json openclaw.json.phase-N`，验证不通过时可快速恢复。

### Phase 0: 基础设施验证（第 1 天）

1. 在飞书开放平台创建 4 个自建应用（coordinator + pm + backend + tester）
2. 将 4 个应用添加到同一个飞书群
3. 创建飞书多维表格（先建 daily_summaries 和 cross_reviews 两张表）
4. 验证每个机器人能在群里发消息
5. **验证 sessions_send 唤起的 Agent 能否通过 message 工具发飞书群**
6. **验证 Coordinator 在 isolated session 中能否通过 message 工具发飞书群**
7. 验证 4 个飞书应用同时在线的 WebSocket 稳定性
8. 验证 Coordinator 能读写 Bitable
9. 获取并记录飞书群 chat_id

**验证标准**：
- [ ] 4 个飞书机器人已加入目标群，能在群里正常发消息
- [ ] sessions_send 唤起的 Agent 能通过 message(action="send", target="oc_xxx", channel="feishu") 发消息到群
- [ ] Coordinator 在 isolated session 中能通过 message 工具发消息到群
- [ ] 4 个应用同时在线 30 分钟无断连
- [ ] Coordinator 写入 Bitable 成功

### Phase 1: 最小可用版（第 2-4 天）

**范围**：Coordinator + 3 个角色（PM、后端、测试）

1. 创建 4 个 Agent 的 workspace 和配置文件
2. 配置 openclaw.json（accounts、bindings、tools.agentToAgent、session.agentToAgent）
3. 编写各 Agent 的 IDENTITY.md、SOUL.md、AGENTS.md、USER.md
4. 设置 cron 定时任务（含 --channel feishu --to "oc_xxx"）
5. 测试 sessions_send 通信
6. 测试 message 工具发消息到飞书群

**验证标准**：
- [ ] Cron 触发 Coordinator
- [ ] Coordinator 通过 sessions_send 联系 3 个角色 Agent
- [ ] 各 Agent 通过 message 工具在飞书群以自己身份发言
- [ ] 飞书群显示各角色独立身份的消息

### Phase 2: 交叉审阅（第 5-7 天）

1. Coordinator 按配对矩阵交叉分发总结
2. 各 Agent 生成审阅意见并通过 message 工具发到飞书群
3. 被审阅者回应，Coordinator 手动编排最多 3 轮
4. 每轮讨论结果都在飞书群展示

**验证标准**：
- [ ] 每个 Agent 能对审阅对象提出实质性质疑
- [ ] 飞书群能看到完整审阅和回应
- [ ] 各角色消息带有 @ 标注

### Phase 3: 记忆系统（第 8-10 天）

1. 完善飞书多维表格（增加 learnings 表）
2. Review 后异步写入 Bitable
3. Agent 更新个人记忆文件
4. 配置 memory-core dreaming

**验证标准**：
- [ ] Bitable 有完整 Review 记录
- [ ] Agent 的 MEMORY.md 有新增学习内容
- [ ] Dreaming 正常运行
- [ ] 第二天 Review 能引用前一天的结论

### Phase 4: 人类干预（第 11-13 天）

1. Coordinator 监听飞书群消息
2. 支持 /override、/skip、/ask 指令
3. 干预结果写入 Bitable

**验证标准**：
- [ ] 老板 @ Coordinator 发 /override 后能正确处理
- [ ] /skip 能跳过当前讨论
- [ ] 干预决策被记录

### Phase 5: 全量上线（第 14-17 天）

1. 在飞书开放平台创建剩余 4 个应用（architect、ui-designer、frontend、devops）
2. 创建 4 个 Agent 的 workspace 和配置
3. 更新 openclaw.json
4. 编写各角色的 AGENTS.md 和审阅视角
5. 将新应用添加到飞书群

**验证标准**：
- [ ] 7 个角色都能参与 Review
- [ ] 飞书群中每个角色有独立身份
- [ ] 全流程 18:00-19:00 能跑完
- [ ] 10 个飞书应用同时在线稳定

### Phase 6: 优化迭代（第 18 天起，与日常运行并行）

**边界**：仅在现有系统稳定运行 3 天后进入此阶段。每个优化项独立交付，不影响日常 Review。

- [ ] 根据实际运行调整 prompt，提高审阅质量
- [ ] 增加跨日知识关联
- [ ] 增加"专项 Review"模式
- [ ] 实现 patterns 表（反复出现的模式追踪）
- [ ] 实现 review_sessions 表（崩溃恢复机制）

### Phase 7 候选功能（新功能扩展，非优化）

以下为独立功能扩展，需单独设计和验证：

- [ ] 老板发起即时 Review（飞书群 @ Coordinator 触发，非 cron 触发）
- [ ] 新增角色（如 marketing、data）

---

## 10. 新增角色操作指南

```bash
# 1. 飞书开放平台创建自建应用，获取 App ID / App Secret

# 2. 创建 workspace 和 agent 目录
mkdir -p /root/.openclaw/workspace-marketing/memory
mkdir -p /root/.openclaw/agents/marketing/agent

# 3. 编写 workspace 文件: IDENTITY.md, SOUL.md, AGENTS.md, USER.md

# 4. 更新 openclaw.json
#    - agents.list 添加 marketing
#    - channels.feishu.accounts 添加 marketing（含 requireMention: false）
#    - bindings 添加路由
#    - tools.agentToAgent.allow 添加 "marketing"

# 5. 更新 Coordinator 的 AGENTS.md 注册表和审阅配对矩阵

# 6. 将新飞书应用添加到群

# 7. 重启 gateway
systemctl --user restart openclaw-gateway
```

核心流程无需改动，新角色自动融入。
