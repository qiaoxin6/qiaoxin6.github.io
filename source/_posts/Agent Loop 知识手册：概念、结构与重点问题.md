---

title: Agent Loop 知识手册：概念、结构与重点问题
date: 2026-09-02 00:04:27
tags:
  - agent
  - 面试
  - Agent Loop
categories:
  - AI

---

整理时间：2026-09-09
资料来源：LangChain《The Art of Loop Engineering》(2026-06)、Anthropic《Building Effective Agents》(2024-12)、ReAct 原论文 (arXiv:2210.03629, ICLR 2023) 等，完整出处见文末。



------

## 目录

1. [什么是 Agent Loop](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#一什么是-agent-loop)
2. [概念边界：Agent、Workflow 与 Chatbot](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#二概念边界agentworkflow-与-chatbot)
3. [演进脉络：从 ReAct 到 Loop Engineering](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#三演进脉络从-react-到-loop-engineering)
4. [核心结构：四层嵌套循环（LangChain, 2026）](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#四核心结构四层嵌套循环langchain-2026)
5. [Agent Loop 内部的经典范式](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#五agent-loop-内部的经典范式)
6. [生产级 Agent Loop 的工程要素](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#六生产级-agent-loop-的工程要素)
7. [常见重点问题与答题要点（面试向）](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#七常见重点问题与答题要点面试向)
8. [关键术语速查表](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#八关键术语速查表)
9. [参考资料](https://www.yuque.com/u59107065/gr0esb/vd6w823g2ki7tf84#九参考资料)



------

## 一、什么是 Agent Loop

### 1.1 定义

**Agent Loop（智能体循环 / Agentic Loop）** 是驱动 AI Agent 自主完成任务的核心运行机制：给大模型（LLM）上下文，让它在一个**循环**中交替进行推理、调用工具、读取环境反馈，直到任务完成或触发停止条件。

LangChain 在《The Art of Loop Engineering》中对最底层循环的描述是：

"The core agent algorithm is simple: give the LLM context and let it call tools in a loop until it's done."（核心算法很简单：给 LLM 上下文，让它在循环中调用工具，直到完成。）

Anthropic 在《Building Effective Agents》中的表述与之等价：

"They are typically just LLMs using tools based on environmental feedback in a loop."（Agent 本质上就是 LLM 基于环境反馈、在循环中使用工具。）

### 1.2 最小闭环

一个可运行的 Agent Loop 伪代码：



```plain
messages = \[system\_prompt, user\_task]

while True:

  response = llm(messages, tools)          # 1. 推理：决定下一步动作

  if not response.tool\_calls:              # 4. 终止判断：模型不再调用工具

         return response.content              #    → 输出最终结果

     for call in response.tool\_calls:

         result = execute\_tool(call)          # 2. 行动：执行工具

         messages.append(tool\_result(result)) # 3. 观察：结果回填上下文

     messages.append(response)
```

对应四个标准阶段：



| 阶段 | 英文           | 含义                                 |
| ---- | -------------- | ------------------------------------ |
| 感知 | Perceive       | 获取用户输入、工具返回、环境状态     |
| 推理 | Think / Reason | LLM 结合目标、记忆、工具，决策下一步 |
| 行动 | Act            | 调用工具 / 执行代码 / 操作外部系统   |
| 观察 | Observe        | 捕获执行结果，写回上下文，进入下一轮 |

### 1.3 两个必须有的控制件

- **停止条件（Stopping Conditions）**：任务完成只是理想终止方式；生产系统必须额外设置硬边界，如**最大迭代次数**、Token / 成本预算、超时（Anthropic 原文明确建议 "include stopping conditions, such as a maximum number of iterations, to maintain control"）。
- **环境真值（Ground Truth）**：每一步都要从环境获得真实反馈（工具结果、代码执行输出、测试结果）来判断进展，而不是让模型 "凭感觉" 宣布完成。



------

## 二、概念边界：Agent、Workflow 与 Chatbot

### 2.1 Anthropic 的权威划分：Workflows vs Agents

Anthropic 把所有 LLM 智能系统统称为 **agentic systems（智能体系统）**，但在架构上明确区分两类：



- **Workflows（工作流）**：LLM 和工具按**预定义的代码路径**被编排，流程走向由开发者写死。
- **Agents（智能体）**：LLM **动态地自主决定**流程和工具使用，自己掌控 "如何完成任务"。

判定一个系统是不是真正的 Agent Loop，关键看

**下一步的控制权在谁手里**

：在 LLM 手里（动态决策）才是 Agent；在开发者代码手里（固定路径）则是 Workflow。

### 2.2 三者对比

| 对比项   | 传统 Chatbot | Workflow           | Agent（Agent Loop）               |
| -------- | ------------ | ------------------ | --------------------------------- |
| 执行路径 | 单轮问→答    | 开发者预定义、固定 | LLM 动态决定、不可预先枚举        |
| 工具使用 | 无或单次     | 按固定节点调用     | 自主选择、可多轮组合              |
| 反馈闭环 | 无           | 可有 gate 校验     | 每步基于环境反馈迭代              |
| 适用任务 | 闲聊、问答   | 定义清晰、步骤固定 | 开放式、步数不可预测              |
| 代价     | 低延迟低成本 | 可预测、一致性好   | 延迟 / 成本更高，错误可能复合累积 |

Anthropic 的核心建议：**从最简单的方案开始，只在可证明有收益时才增加复杂度**—— 很多场景单次 LLM 调用 + 检索 + 示例就足够，不必上 Agent。



------

## 三、演进脉络：从 ReAct 到 Loop Engineering

```plain
2022.10  ReAct 论文（Yao et al., Princeton/Google）奠定 Thought→Action→Observation 的单循环范式

          │

2024.12  Anthropic《Building Effective Agents》总结 augmented LLM + 5 种 Workflow + 自主 Agent 的模式谱系

          │

2025.07  Geoffrey Huntley 提出 "Ralph" 技术用外层循环驱动长任务、状态外置到文件，是"外层循环"的朴素范例

          │

2026.06  swyx（Latent Space）提出 "Loopcraft：堆叠循环的艺术"

2026.06  LangChain《The Art of Loop Engineering》形式化"四层嵌套循环"行业范式从 Prompt Engineering 转向 Loop Engineering
```

### 3.1 ReAct（2022）—— 一切的起点

- 论文：*ReAct: Synergizing Reasoning and Acting in Language Models*，Shunyu Yao 等，arXiv:2210.03629，ICLR 2023。
- 核心思想：把 ** 推理轨迹（Thought）**和**具体动作（Action）** 交错生成；推理帮助制定 / 修正计划、处理异常，动作让模型接入外部知识库与环境获取新信息，Observation 再反哺下一轮推理。
- 这就是 L1 Agent Loop 最经典的内部实现形态。

### 3.2 Loop Engineering（2026）—— 当前的范式焦点

- 背景观点（swyx "Loopcraft"）：不要只优化一个循环，而要**堆叠、嵌套多层循环**来换取可靠性。
- LangChain 将其形式化为四层嵌套结构（见第四章）。
- 核心转变：**竞争力不在模型本身，而在你围绕模型搭建的循环与管控框架（Harness）**；行业注意力正从内两层（执行、校验）转向外两层（事件化部署、自我改进），因为价值在外层复利累积。

**Harness（管控框架 / 马具）**

：包裹模型的全部工程设施 —— 上下文拼装、工具路由、循环控制、校验、状态管理、重试与安全边界。模型是引擎，Harness 是让引擎可靠工作的整车。



------

## 四、核心结构：四层嵌套循环（LangChain, 2026）

这是目前对生产级 Agent 循环体系最清晰的工程化表述。四层**由内向外逐层包裹**：内层是外层的执行单元，外层为内层提供质量保障、规模化接入和自我进化能力。



```plain
┌───────────────────────────────────────────────────────────┐

│ L4  Hill Climbing Loop（爬坡/自改进循环）                    │

│  分析生产轨迹(traces) → 反向改写内层 Harness 配置            │

│  ┌─────────────────────────────────────────────────────┐  │

│  │ L3  Event Driven Loop（事件驱动循环）                  │  │

│  │  事件触发(cron/webhook/消息) → 运行 → 更新业务系统      │  │

│  │  ┌───────────────────────────────────────────────┐  │  │

│  │  │ L2  Verification Loop（验证循环）               │  │  │

│  │  │  输出 → Grader 按 rubric 打分 → 不过则带反馈重试 │  │  │

│  │  │  ┌─────────────────────────────────────────┐  │  │  │

│  │  │  │ L1  Agent Loop（基础执行循环）            │  │  │  │

│  │  │  │  request → model ⇄ tools → result        │  │  │  │

│  │  │  │           (repeat until done)            │  │  │  │

│  │  │  └─────────────────────────────────────────┘  │  │  │

│  │  └───────────────────────────────────────────────┘  │  │

│  └─────────────────────────────────────────────────────┘  │

└───────────────────────────────────────────────────────────┘
```

### L1：Agent Loop（基础执行循环）—— 自动化工作

- **逻辑**：模型在循环中调用工具，直到任务完成。`request → model ⇄ tools（action / observation）→ repeat until done → result`。
- **工具（Tools）** 是 Agent 对真实世界采取行动的能力来源。
- LangChain 对应原语：`create_agent`（任意模型 + 任意工具即得到一个可用循环）。
- 文中示例（文档 Agent）：接收文档改进需求 → 模型规划并起草 → 用沙箱工具克隆仓库、读 / 写文件 → 提交 Pull Request。

### L2：Verification Loop（验证循环）—— 保障质量

- **要解决的问题**：L1 能把活干完，但**第一遍不总是正确、不一致**。
- **逻辑**：L1 产出结果后交给 **Grader（评分器）** 按 **rubric（评分标准）** 检查；不通过就**带着具体反馈打回重试**，通过才结束。
- **Grader 两种形态**：



- 确定性校验：如链接是否有效、CI 是否通过、测试是否绿；
- 智能体校验（agentic）：用另一个 LLM 当裁判，即经典的 **LLM-as-a-judge**。

- LangChain 对应原语：`RubricMiddleware`，或 `create_agent` 的 `after_agent` 钩子。
- **权衡**：验证会增加单次运行的**延迟与成本**；当质量比速度重要时（多数生产场景）值得加。

### L3：Event Driven Loop（事件驱动循环）—— 规模化运行

- **要解决的问题**：把 Agent 从 "手动调用一次的工具" 变成 "融入业务生态、后台常驻的系统组件"。
- **逻辑**：事件发生（新文档落地、定时 schedule 触发、webhook 到达、频道来消息）→ 启动内层（L2+L1）运行 → 更新真实系统 → 系统变化又产生新事件。
- LangChain 对应原语：LangSmith Deployment（cron 定时、webhook）、Fleet 的 channels/schedules。
- 典型形态："心跳（heartbeats）" 定时任务让 Agent 成为 always-on 的主动助手；Slack `#docs-plz` 频道一来消息就触发文档 Agent。

### L4：Hill Climbing Loop（爬坡 / 自改进循环）—— 让系统越用越好

- **要解决的问题**：前三层自动化 "工作"，第四层自动化 "**改进本身**"，是文章认为最重要的一层。
- **逻辑**：

1. 每次内层运行都产生一条 **trace（轨迹）**：模型做了什么、调了哪些工具、Grader 给了什么反馈；
2. 一个**分析 Agent** 批量分析历史轨迹，定位共性问题；
3. 直接**改写内层 Harness 配置**（prompt、工具定义、grader 规则等），多条轨迹指向同一问题时就提 issue / 变更。

- **关键设计**：返回箭头不是简单回到循环顶部，而是**伸进内部直接更新 Agent Loop 本身**—— 外层每转一圈，内层就更强一点，形成复利。
- **延伸**：对开源权重模型，轨迹 / 评测结果可作为 **RL 微调**信号去优化模型本身；记忆、检索技能等辅助上下文也可用同样方式优化。
- LangChain 对应原语：LangSmith **Engine**（轨迹分析 Agent）。

### 四层总览表（对照原文表格）

| 层级 | 名称               | 做什么                               | 价值             | LangChain 原语                                       |
| ---- | ------------------ | ------------------------------------ | ---------------- | ---------------------------------------------------- |
| L1   | Agent loop         | 模型反复调用工具直到任务完成         | 自动化工作       | `create_agent`                                       |
| L2   | Verification loop  | 结果按 rubric 打分，失败带反馈重试   | 保证质量与正确性 | `RubricMiddleware`                                   |
| L3   | Event driven loop  | 事件触发 Agent 运行并更新真实系统    | 规模化自动化     | LangSmith Deployment（cron/webhook）、Fleet channels |
| L4   | Hill climbing loop | 用生产轨迹喂分析 Agent，改进 Harness | 自我进化、复利   | LangSmith Engine                                     |

### Human in the Loop（人类在环）

自动化不等于移除人类。原文指出每一层都有人类介入的最佳点，LangChain 把 "人类在环" 作为**一等原语**：



1. **L1**：敏感动作（金融交易、数据库写操作等）执行前要求人工确认；
2. **L2**：高敏感工作流由人担任最终 Grader（自动校验能查链接是否有效，但 "表述是否适合目标读者" 需要人的判断）；
3. **L3 / 应用层**：结果返回终端用户前由人审批；
4. **L4**：Harness 的改进配置经人工审核后再部署。



------

## 五、Agent Loop 内部的经典范式

ReAct、Plan-and-Execute、Reflection 

**都属于 Agent Loop（L1 层）的内部组织策略**

，是同一概念下的不同实现，而不是与 Agent Loop 并列的概念。Anthropic 总结的 5 种 Workflow 模式则是 "半固定路径" 的近亲，常与 Agent Loop 组合使用。

### 5.1 ReAct（边想边做）

- 循环：**Thought → Action → Observation**，每走一步重新决策。
- 优点：灵活、对动态环境适应性强、过程可解释。
- 短板：长任务容易偏离目标、缺乏全局规划；迭代轮次不可控，Token 成本随轮次线性增长。
- 适用：故障排查、工具查询、路径不可预测的短链路任务。

### 5.2 Plan-and-Execute（先规划后执行）

- 循环：**Plan（全局拆步骤）→ Execute（逐步执行）→ Replan（必要时重规划）→ Synthesize（汇总）**。
- 优点：长任务有全局观，执行阶段可减少昂贵 LLM 调用。
- 短板：初始计划可能与实际脱节，必须配套重规划机制。
- 适用：报告生成、多步骤工程任务等目标明确的长流程。

### 5.3 Reflection / Reflexion（反思增强）

- 定位：**不是独立循环，而是可叠加的增强层**（对应 L2 Verification Loop，也对应 Anthropic 的 **Evaluator-optimizer** 模式：一个 LLM 生成、另一个 LLM 评估并给反馈，循环改进）。
- 循环：执行 → 自评 / 他评 → 发现偏差 → 修正重试。
- 适用：代码、翻译、写作等有明确评价标准、迭代打磨有 measurable value 的任务。

### 5.4 Anthropic 的 5 种 Workflow 模式（Agent Loop 的近亲组件）

| 模式                 | 机制                                               | 典型用途                               |
| -------------------- | -------------------------------------------------- | -------------------------------------- |
| Prompt chaining      | 任务拆成固定串行步骤，中间可加程序化 gate          | 先写大纲校验后再写全文；先写文案再翻译 |
| Routing              | 先分类，再路由到专门的下游 prompt / 模型           | 客服分流；简单题走小模型、难题走大模型 |
| Parallelization      | 并行处理后聚合：Sectioning（分片）/ Voting（多票） | 护栏与主回答并行；多视角代码漏洞审查   |
| Orchestrator-workers | 中心 LLM 动态拆任务、派发给 worker、再综合         | 一次改动多文件的编码；多源检索分析     |
| Evaluator-optimizer  | 生成者 + 评估者在循环中迭代打磨                    | 文学翻译、多轮检索精炼                 |

**Orchestrator-workers 与 Parallelization 的区别**

（易考点）：拓扑相似，但前者子任务

**无法预先定义**

由 orchestrator 根据输入动态决定，后者的分片是预先确定的。

### 5.5 选型原则

- 任务短、环境动态、工具多 → **ReAct**；
- 任务长、流程可预先拆解 → **Plan-and-Execute**；
- 质量要求高、有明确评分标准 → 叠加 **Reflection / Verification**；
- 子任务可预测 → Workflow 更稳；子任务不可预测、需要模型自主决策 → 才用真正的 Agent Loop；
- 生产级系统通常是**混合体**：规划 + 单步 ReAct 执行 + 验证层 + 事件化部署。



------

## 六、生产级 Agent Loop 的工程要素

### 6.1 上下文与记忆管理

- **短期记忆**：当前会话消息流，受上下文窗口限制。
- **工作记忆**：当前子目标、已完成 / 待办步骤（建议外置为结构化文件或状态存储，而非只靠对话历史）。
- **长期记忆**：跨会话的经验与知识，常配合向量检索召回。
- **窗口溢出处理手段**：滑动窗口、历史摘要压缩、关键信息（目标 / 结论）置顶、冗余工具输出裁剪、非实时信息外置到数据库按需检索。
- **长任务的上下文漂移**：多轮之后模型会逐渐 "忘记" 原始目标或质量衰减；工程对策是状态外置、每轮从权威状态文件重新加载（Ralph 类技术的核心思路），并设置特性清单防止 "提前宣布完成"。

### 6.2 工具设计与 ACI（Agent-Computer Interface）

Anthropic 强调：要像设计 HCI（人机界面）一样认真设计 **ACI（智能体 - 计算机界面）**，在 SWE-bench Agent 上他们花在打磨工具上的时间比打磨主 prompt 还多：



- 工具描述写清用途、边界、参数、示例与边缘情况，相似工具要明确区分；
- 选择对模型友好的格式（贴近自然文本、避免繁琐转义 / 计数）；
- **Poka-yoke（防呆设计）**：从参数层面让模型难以犯错（例：把文件工具从 "相对路径" 改为 "强制绝对路径" 后，模型不再出错）；
- 大量测试模型如何使用工具，按错误迭代工具定义。

### 6.3 终止、异常与防护

- **硬边界**：最大迭代轮数、总 Token / 成本预算、总超时，三者缺一不可；
- **工具治理**：入参 schema 校验、超时、有限重试（指数退避）、熔断（同一工具连续失败 N 次则暂时禁用并换方案）、幂等性；
- **死循环防护**：检测 "相同工具 + 相同参数 + 相同错误" 的重复动作、连续多轮无新增进展（无新信息 / 无子任务完成）即强制中断；
- **错误信息结构化**：明确告诉模型错在哪、可以怎么换路，而不是只返回 "failed"；
- **沙箱与护栏**：Anthropic 建议在沙箱中充分测试，配合权限控制与人工确认点，因为 Agent 的自主性会带来 ** 错误复合累积（compounding errors）** 风险。

### 6.4 可观测性与评估

- 每次运行留存完整 **trace**：每轮推理、工具调用、参数、返回、grader 反馈 —— 这既是排障依据，也是 L4 自改进循环的燃料；
- 核心指标：



- 效果：任务完成率、结果正确率 / 评分；
- 效率：平均迭代轮数、端到端耗时、Token 消耗；
- 稳定性：工具成功率、死循环 / 异常率；
- 成本：单任务平均成本、有效 Token 占比。



------

## 七、常见重点问题与答题要点（面试向）

### A. 概念辨析类

**Q1：Agent Loop 是什么？它和普通对话的本质区别？**



- 普通对话是 "输入→输出" 的单轮映射；Agent Loop 是 "推理→行动→观察" 的**带状态闭环**，模型基于环境反馈多轮自主推进，直到目标达成或触发停止条件。
- 本质区别在于有没有 "自主行动 + 环境反馈 + 多轮迭代"。

**Q2：ReAct、Plan-and-Execute 属于 Agent Loop 吗？**



- 属于。它们是 L1 Agent Loop 的**内部实现范式**：ReAct 单步决策、Plan-and-Execute 先规划后执行、Reflection 是叠加的验证增强层；Agent Loop 是上层通用机制。

**Q3：Agent 和 Workflow 的区别？什么场景不该用 Agent？**



- 按 Anthropic：Workflow 走开发者预定义的固定代码路径，可预测、一致性好；Agent 由 LLM 动态决定路径，灵活但更贵、错误会复合。
- 步骤固定、可硬编码的任务用 Workflow；单次 LLM 调用 + 检索就能解决的甚至不需要 agentic 系统；只有开放式、步数不可预测的任务才用 Agent。

### B. 架构模式类

**Q4：讲一下 LangChain 的四层循环结构。**



- L1 Agent Loop（模型调工具完成工作）→ L2 Verification Loop（grader 按 rubric 校验、失败带反馈重试）→ L3 Event Driven Loop（事件触发、后台常驻、更新业务系统）→ L4 Hill Climbing Loop（分析 trace 反向优化内层 harness 配置，实现复利式自改进）。
- 亮点：L4 的反馈箭头直接改写内层配置本身；每层都可插入 human-in-the-loop。

**Q5：ReAct / Plan-and-Execute / Reflection 怎么选型？**



- 看任务复杂度、流程确定性、质量要求：短而动态选 ReAct；长而可拆解选 Plan-Execute；有明确评分标准就叠加 Reflection；生产环境通常混合使用，并加硬停止边界。

**Q6：Verification Loop 的 Grader 有哪些形态？代价是什么？**



- 确定性校验（测试、CI、规则）与 agentic 校验（LLM-as-a-judge）；代价是延迟和成本上升，质量优先于速度时值得。注意反馈要具体、可执行，才能让重试有效。

**Q7：Orchestrator-workers 和 Parallelization 有什么区别？**



- 都是 "一个中心 + 多路执行"，区别在于子任务是否可预先定义：Parallelization 的分片事先固定；Orchestrator-workers 的子任务由中心 LLM 按输入动态生成（如改哪些文件事先不知道）。

### C. 工程实战类

#### Q : 如何判定一个任务完成了？

**面试一句话答题模板**

" 我不会让模型自判完成作为依据。判定分四步：先用最大轮数、预算、超时、无进展检测做硬边界保护；模型不再使用工具就是agent认为任务完成（并不代表真正完成了），后面跑客观验收（测试、schema、逐项 feature list 全 passes）；再由独立 grader 或人按 rubric 复核，不通过带反馈重试；全部通过才算成功。关键是区分**保护性终止**和**成功完成**，前者必须显式标记未完成并上报。"

```plain
          ┌─────────────────────────────────────────┐
          │ 1. 是否触发硬边界(轮数/预算/超时/无进展)? │
          └─────────────────────────────────────────┘
                    │是→ 保护性终止：标记未完成+卡点上报/转人工
                    │否
          ┌─────────────────────────────────────────┐
          │ 2. 模型是否自判完成(不再调用工具)?        │
          └─────────────────────────────────────────┘
                    │否→ 继续下一轮循环
                    │是
          ┌─────────────────────────────────────────┐
          │ 3. 客观验收：测试/断言/Feature清单全通过? │
          └─────────────────────────────────────────┘
                    │否→ 把失败项作为反馈打回，继续循环
                    │是
          ┌─────────────────────────────────────────┐
          │ 4. 独立 Grader / 人审 是否通过?          │
          └─────────────────────────────────────────┘
                    │否→ 带反馈重试
                    │是
                    ▼
                 ✅ 真正完成，交付
```

**Q8：Agent 陷入死循环，如何排查与解决？**



- 先定位根因：目标模糊导致无法判断终止 / 工具报错信息不清引发反复重试 / 上下文过长丢失目标和进展 / 只靠模型自觉没有硬边界。
- 对策：①最大轮数、预算、超时三重硬限制；②重复动作与无进展检测，命中即中断；③错误信息结构化并给出替代路径；④prompt 中显式定义终止条件、要求每轮汇报进展；⑤状态外置防止上下文漂移。

**Q9：上下文窗口爆了怎么办？**



- 滑动窗口丢弃旧历史；接近阈值时做摘要压缩；目标与关键结论置顶；裁剪冗长的工具返回；把非实时信息外置到存储 / 向量库按需检索；长任务采用 "状态写文件、每轮重新加载" 的新鲜上下文模式。

**Q10：模型总是选错工具 / 填错参数，怎么优化？**



- 优化工具描述与示例、划清相似工具边界；schema 强约束输出并在运行时校验；做防呆设计（如强制绝对路径、枚举值收窄）；失败时返回精确的纠错信息；用 workbench 批量观察模型错误并迭代工具（ACI 思路）。

**Q11：如何控制 Agent 的成本？**



- 硬预算（轮数 / Token / 金额上限）；模型分级（路由：简单步骤用小模型、关键总结用大模型）；并行工具调用减少串行轮次；缓存重复查询与规划；压缩上下文；无进展提前终止减少空转。

**Q12：工具调用失败如何设计异常处理？**



- 入参先校验，不合法直接拦截；执行层超时 + 有限次指数退避重试；错误结构化返回原因与建议；同工具连续失败触发熔断、提示模型改走其他路径；写操作保证幂等，避免重试造成重复提交。

**Q13：如何保证长任务 Agent 不 "提前宣布完成"？**



- 建立特性 / 需求清单文件，要求逐项自验证，测试通过后才能标记完成（Anthropic 长任务 harness 实践）；提供一键运行环境脚本降低验证成本；L2 加独立 grader 做客观验收，而不是让执行模型自评。

### D. 场景设计类

**Q14：设计一个代码修复 Agent，你会怎么搭循环？**



- 内层用 ReAct（读报错→定位→改→跑测试，环境高度动态）；叠加 L2 Verification（以测试结果作为确定性 grader，不通过带报错重试，必要时加 LLM 评审）；状态用 issue / 任务清单 + Git 历史外置；L3 可由 issue/PR 事件触发；设置轮数与成本上限、危险操作人工确认；全部 trace 入库供 L4 分析共性问题。

**Q15：多 Agent 系统和单 Agent Loop 是什么关系？**



- 每个子 Agent 内部仍是自己的 Agent Loop；上层 Orchestrator 循环负责任务拆解、分发、同步、综合（对应 Orchestrator-workers）。避免互相死锁的手段：明确权责边界、标准化消息 / 状态协议、中心化调度、跨 Agent 调用超时与全局终止条件。

**Q16：哪些地方必须保留 Human-in-the-loop？**



- 敏感动作执行前确认（资金、数据库、对外发送）；高风险结果由人担任最终 grader；交付用户前审批；自动生成的 harness/prompt 变更上线前审核。原则：自动校验能覆盖 "客观对错"，人的判断负责 "语境、品味与合规"。

### E. 评估与趋势类

**Q17：怎么评估一个 Agent Loop 的好坏？**



- 四组指标：效果（完成率、正确率）、效率（平均轮数、耗时、Token）、稳定性（工具成功率、死循环率、异常恢复率）、成本（单任务成本、有效 Token 比）。同时用离线评测集 + 生产 trace 双轨评估。

**Q18：为什么说行业从 Prompt Engineering 转向 Loop Engineering？**



- 单轮 prompt 有天花板，复杂任务无法一次做对；多层循环用 "执行 — 校验 — 重试 — 自改进" 把可靠性工程化；模型趋于同质化，真正的壁垒在围绕模型的 harness、业务数据沉淀和学习飞轮 ——LangChain 的判断是价值在 L3/L4 层复利累积。



------

## 八、关键术语速查表

| 术语                         | 含义                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| Agent Loop                   | LLM 在 "推理 - 行动 - 观察" 循环中使用工具直至完成的基础机制 |
| Harness                      | 包裹模型的全部管控工程（上下文、路由、循环、校验、安全边界） |
| ReAct                        | Thought→Action→Observation 交错的经典单循环范式（2022）      |
| Plan-and-Execute             | 先全局规划、再分步执行、按需重规划的范式                     |
| Reflection / Reflexion       | 执行后自评并修正的增强层                                     |
| Grader / Rubric              | 验证循环中的评分器与其评分标准                               |
| LLM-as-a-judge               | 用另一个 LLM 充当评审的 agentic 校验方式                     |
| Trace                        | 一次运行的完整轨迹记录（推理、工具、反馈），是排障与自改进的燃料 |
| Hill Climbing Loop           | 基于 trace 自动改写内层 harness 的自改进外循环               |
| ACI                          | Agent-Computer Interface，智能体 - 工具界面，类比人机界面 HCI |
| Poka-yoke                    | 防呆设计，从接口层面让模型难以犯错                           |
| Ground Truth                 | 每步从环境获取的真实反馈，用于判断真实进展                   |
| Compounding Errors           | Agent 自主多步执行中错误逐级放大的风险                       |
| Human in the Loop            | 在循环关键节点插入人工确认 / 审批                            |
| Loopcraft / Loop Engineering | swyx 提出、LangChain 形式化的 "堆叠循环" 工程方法论          |

------

## 九、参考资料

1. LangChain — *The Art of Loop Engineering*, Sydney Runkle, 2026-06-16：https://www.langchain.com/blog/the-art-of-loop-engineering
2. Anthropic — *Building Effective Agents*, 2024-12-19：https://www.anthropic.com/engineering/building-effective-agents
3. Yao et al. — *ReAct: Synergizing Reasoning and Acting in Language Models*, arXiv:2210.03629, ICLR 2023：https://arxiv.org/abs/2210.03629
4. Anthropic — *Effective harnesses for long-running agents*, 2025-11：https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
5. Sofokus — Loop engineering 术语表（Ralph 技术出处说明）：https://www.sofokus.com/ai-glossary/loop-engineering/
6. Arize — *What is a loop in AI engineering, anyway?*（Loopcraft 分层解读）：https://arize.com/blog/what-is-a-loop-in-ai-engineering-anyway/

说明：文中所有模式定义、四层结构与原语名称均以上述一手资料为准；工程实践部分（死循环防护、成本控制等）为业界通用落地经验的归纳。