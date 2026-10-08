+++
draft = true
date = 2026-09-12T16:24:00+08:00
title = "从 Prompt 到 Loop：AI 工程真正在演进什么"
description = "Prompt、Context、Harness、Loop 不是每隔几个月换一批新词。它们标出同一条工程路线：AI 从被调用的模型，变成能长期运行、可验证、能持续干活的系统。"
slug = ""
authors = []
tags = ["AI", "Agent", "Prompt", "Context Engineering", "Harness", "Loop", "FDE"]
categories = ["AI"]
externalLink = ""
series = []
disableComments = true
+++

> 从“如何让模型回答得更好”，到“如何让 Agent 持续、可靠、自主地完成真实工作”。

## 引言：新词很多，路线其实很清楚

过去几年，AI 工程领域几乎每隔几个月就会冒出一批新概念：Prompt Engineering、RAG、Tool Use、Agent、Coding Agent、Context Engineering、Skills、MCP、Harness Engineering、Loop Engineering、Ralph Loop、Sub-agent、Hill Climbing、FDE。看起来很像名词通胀。一部分确实如此，很多术语互相重叠，有些只是给已有实践换了个名字。

这篇文章想回答的是：**这些词为什么会按这个顺序出现，后面真正重要的又是什么。**

按它们进入工程讨论的先后，路线其实很清楚：

```text
Prompt
  ↓
RAG / Tool Use
  ↓
Agent
  ↓
Coding Agent
  ↓
Context
  ↓
Skills / MCP
  ↓
Harness
  ↓
Loop
  ↓
Multi-Agent
```

FDE 不在这条技术栈上。它是把整条栈送进真实组织的人，后面单独写。

关注点一直在往上移：

> **从“模型如何生成答案”，转向“如何构建一个能够持续完成复杂工作的 AI 系统”。**

---

## 一、Prompt Engineering：怎么告诉模型

Prompt Engineering 解决的是最基本的问题：

> **如何告诉模型我要什么？**

```text
你是一名资深 Go 工程师。

请分析下面这段代码，并找出潜在的并发问题。

要求：
1. 遵循 Go 最佳实践
2. 不修改现有 API
3. 给出修改后的完整代码
```

这一阶段的主要技术包括 Role Prompting、Few-shot、Chain-of-Thought、System Prompt、Instruction Following。结构也很简单：

```text
User Prompt → LLM → Output
```

优化对象只有一个：Prompt。它很有用，但边界很快就露出来了——Prompt 再精，模型也不知道公司内部知识，也不会自己去跑测试、改仓库、提交 PR。

---

## 二、RAG 与 Tool Use：给知识，也给手脚

很快出现两条并行路线。

### RAG：给模型知识

Prompt 解决的是“怎么说”，RAG 解决的是“该看什么资料”。

```text
User Query
    ↓
Retriever
    ↓
Knowledge Base
    ↓
Relevant Context
    ↓
LLM
    ↓
Answer
```

问题从“怎么告诉模型”变成：

> **应该给模型什么知识？**

### Tool Use：给模型行动能力

另一条线是 Function Calling / Tool Use。模型不再只会生成文本，开始可以观察外部世界，也可以操作外部世界：查天气、读仓库、跑命令、写文件。

这是 Agent 出现的前提。没有工具，所谓 Agent 只是一个会多说几句的聊天窗口。

---

## 三、Agent：让模型自己决定下一步

传统 LLM 是一次性问答：

```text
User → LLM → Answer
```

Agent 是循环：

```text
User → Agent → Reason → Tool → Observe → Reason → Tool → Observe → ... → Done
```

Anthropic 对 Agent 的定义一直很干净：

> LLM autonomously using tools in a loop.

关键不在“模型更聪明”，而在于：

> **模型开始参与决策，并决定下一步行动。**

所以：

```text
LLM + Tools + State + 工具循环 = Agent
```

这里的“循环”还只是 Agent 内部的 Reason-Act-Observe。它解决的是这一次任务里下一步调什么工具。至于**什么时候再启动一轮 Agent、失败后谁来重试、什么时候停止**，那是后面 Loop Engineering 才单独拿出来的问题。

到这里，AI 已经从“被调用的函数”变成了“能自己走几步的执行体”。但只要它还跑在聊天窗口里，真正卡住工程的就不是会不会调工具，而是：每一轮它到底看见了什么，以及它被允许在什么环境里动手。

---

## 四、Coding Agent：AI 真正进入软件开发

当 Agent 进入软件开发，Claude Code、Codex、Cursor Agent、OpenHands、Aider、Cline 这类产品开始出现。AI 不再只是“帮我生成一段代码”，而是：

```text
读取项目 → 理解代码 → 制定方案 → 修改代码 → 运行测试
    → 分析错误 → 继续修改 → 再次测试 → 提交代码
```

这时一个关键问题出现了：

> **Prompt 之外，还需要一个完整的执行环境，也需要精心控制模型此刻看到的信息。**

谁给它 shell？谁限制它能不能上网、能不能改 Git？测试什么时候跑？失败之后怎么重来？状态保存在哪？权限如何收口？每一轮又该把哪些文件、错误和历史放进上下文？

Coding Agent 把后面几层问题同时暴露出来，于是 Context、Harness、Loop 才陆续被单独命名。

### Vibe Coding 和 Agentic Coding

这两个词也是在 Coding Agent 普及之后流行起来的。

Vibe Coding 是：人描述想法，AI 快速生成代码，人再检查。适合 Demo、原型、小工具。

```text
Idea → AI → Code → Human Check
```

Agentic Coding 是：需求进来之后，Agent 读代码、做计划、修改、测试、修复，再进入 Review。

```text
Requirement → Agent → Read → Plan → Modify → Test → Fix → Review → Done
```

前者更像 **AI 帮我写代码**；后者更像 **AI 帮我完成软件工程任务**。两者并不互斥。探索期用 Vibe，落地期必须进入 Agentic——因为没有测试、权限和循环，快速生成只会更快地堆积债务。

---

## 五、Context Engineering：模型这一轮该看到什么

Prompt Engineering 问的是：

> 我要告诉模型什么？

Context Engineering 问的是：

> **模型这一刻到底应该看到什么？**

一个 Coding Agent 当前的 Context，远远不止用户那句话。它可能同时包括：

```text
System Prompt
+ User Task
+ AGENTS.md
+ 项目结构
+ 当前文件
+ 相关代码
+ Git Diff
+ 历史对话
+ 工具定义
+ 记忆
+ 测试结果
+ 之前的错误信息
```

这些全部都在抢同一份资源：有限的上下文窗口，也就是有限的注意力预算。Anthropic 把这件事说得很明确——好的 Context Engineering，不是把能塞的都塞进去，而是找到**最小的一组高信号 token**，让下一步行动更可能做对。

Agent 通常不会只跑一轮。如果所有历史都不断堆进去：

```text
Context 越来越长 → 越来越贵 → 噪声越来越多 → 注意力下降
```

所以真正要做的是选择、压缩、摘要、记忆、检索、过滤工具结果。Prompt 是设计指令；Context 是设计模型当前所处的信息环境。

---

## 六、Skills 与 MCP：扩展能力，连接外部世界

Agent 能做的事情变多以后，系统朝两个方向长。

### Skills：按需加载专业方法

Skills 解决的是：

> **Agent 如何获得可复用的专业能力？**

遇到 PostgreSQL 问题就加载 postgres skill，而不是把所有领域知识一次性塞进 System Prompt。它本质上是可被 Agent 动态加载的专业工作方法。

```text
Task → 识别需要 postgres skill → 加载 Skill → 执行
```

### MCP：标准化连接外部世界

MCP（Model Context Protocol）解决的是：

> **Agent 如何标准化地连接外部工具和数据？**

```text
                  Agent
                    │
           ┌────────┼────────┐
           ↓        ↓        ↓
        GitHub    Postgres   Slack
          MCP       MCP       MCP
```

MCP 的价值不是让模型更聪明，而是给 Agent 和外部世界补上一层标准接口。可以把它理解成 AI Agent 生态里的 USB。

Skills 管“怎么干这类活”，MCP 管“怎么接到这类系统”。两者都是在 Coding Agent 出现之后，被大规模拿出来讨论的能力扩展层。

---

## 七、Harness Engineering：给 Agent 一套运行环境

传统软件里，Harness 通常指：驱动程序运行、控制环境、提供测试条件并收集结果的一套基础设施。Agent 时代，这个词被扩成了：

> **围绕模型构建的 Agent Runtime。**

它决定：

- Agent 可以做什么
- 如何调用工具
- 在什么环境里运行
- 如何验证结果
- 如何处理失败
- 如何控制权限
- 如何保存状态

一个 Coding Harness 大致长这样：

```text
                     Agent Harness
                          │
            ┌─────────────┼─────────────┐
            ↓             ↓             ↓
        Context         Tools         State
         Memory         Shell          Git
         Files          Browser      History
         RAG            Database    Checkpoint

            ┌─────────────┼─────────────┐
            ↓             ↓             ↓
      Verification     Security     Observability
          Tests      Permissions        Logs
         Linter       Sandbox        Metrics
```

因此：

```text
Model + Context + Tools + Execution + Verification + Safety + State
= Agent Harness
```

OpenAI 把这件事说得更产品化：Agents SDK / Agents API 开始把 sandbox、文件系统、shell、skills、compaction、可恢复运行，当成“模型旁边的标准基础设施”，而不是每个应用自己手搓一遍。GitHub 则更直白：Copilot 本身就是一个 harness，模型只是马，harness 才是把马力接到仓库、编辑器、PR 和终端上的那套缰绳。

---

## 八、Context 和 Harness 不是一回事

这两个词经常被当成一回事。其实只要问两个问题就分开了。

**Context Engineering：模型现在应该知道什么？**

- 该把哪个文件放进 Context？
- 该保留多少历史？
- 该加载哪个 Skill？
- 该检索哪些知识？

**Harness Engineering：Agent 可以做什么，以及应该如何做？**

- 能不能执行 shell？
- 能不能访问网络？
- 能不能修改 Git？
- 测试什么时候运行？
- 失败之后怎么办？
- 权限如何控制？
- 结果如何验证？

```text
              Agent
                │
       ┌────────┴────────┐
       │                │
    Context           Harness
       │                │
“模型看到什么”     Tools / Sandbox
                    Git / Tests
                    Permissions
                    Verification
                    Retry

                 “模型能做什么”
```

一个很实用的判断标准：如果你改的是“这一轮喂给模型的信息”，那是 Context；如果你改的是“它能不能跑、在哪跑、跑完怎么验收”，那是 Harness。

---

## 九、Loop Engineering：谁决定继续干活

Harness 解决的是：

> Agent 如何可靠地执行任务？

但还有一个更大的问题：

> **谁决定 Agent 什么时候继续工作？**

Loop Engineering 管的就是这件事：何时启动、下一步做什么、何时重试、何时停止、失败之后怎么办、如何根据系统状态决定下一轮任务。

它和外层控制有关，不是第三节里那个 Agent 内部的工具循环。最简单的 Agent 内部循环大家都见过：Task → Action → Observation → 再决策。工程化之后，外层更像一个控制循环：

```text
        ┌─────────────────────┐
        │                     │
        ▼                     │
    Observe State             │
        │                     │
        ▼                     │
    Select Task               │
        │                     │
        ▼                     │
    Agent Execute             │
        │                     │
        ▼                     │
    Verification              │
        │                     │
    ┌───┴────┐                │
    │        │                │
  Fail     Pass               │
    │        │                │
    ▼        ▼                │
  Repair   Next Task ─────────┘
```

本质是：

```text
Trigger + State + Agent + Verification + Retry + Stop Condition
```

对做过 Kubernetes 的人来说，这个结构几乎不陌生。Controller 也是：观察当前状态，对比期望状态，采取行动，再观察。Loop Engineering 做的，就是给 Agent 补上这条控制回路。没有它，你得到的只是一个很能干、但必须人盯着的实习生；有了它，系统才开始接近“持续推进任务”的形态。

### Ralph Loop：最粗暴也最能说明问题的实践

Ralph Loop 是 Loop 思想在 Coding Agent 里非常典型的一种实践。Geoffrey Huntley 把它说成一句很煞风景的话：Ralph 就是一个 Bash 循环。最简形态甚至可以写成：

```bash
while true
do
    agent "继续完成任务"
    test

    if success
    then
        break
    fi
done
```

它不要求一次 Agent Run 把事情做完，而是不断让 Agent 根据当前项目状态继续工作。进度不存在模型记忆里，而存在文件、测试和 Git 里。每一轮都可以是新的上下文窗口，上一轮的产物变成下一轮的输入。

所以 Ralph Loop 看起来很笨，但它把一件事暴露得很清楚：

> 只要验收标准是机械可判定的，循环本身就能把“偶尔做对”变成“持续逼近完成”。

它也有代价：烧 token、可能原地打转、没有好的停止条件就会变成贵且危险的死循环。这正是 Loop Engineering 比“再跑一遍”更重要的原因——真正要设计的是触发、验证和停止，而不只是 `while true`。

---

## 十、Sub-agent 与 Multi-Agent：一群 Agent 如何协作

一个 Agent 能力有限，于是开始把复杂任务拆给多个 Agent。

```text
                Main Agent
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
 Research Agent  Coding Agent  Test Agent
```

Sub-agent 解决的是：如何把复杂任务分解给专门的执行体。主 Agent 负责任务分解和验收，Research、Coding、Test、Review 各管一段。

再往前一步，就形成 Multi-Agent / Agent Squad：一群有角色分工的 Agent 协作；Fleet 则强调并行。

这时问题已经从“一个 Agent 怎么做好事情”，变成：

> **一群 Agent 如何协作完成事情？**

---

## 十一、Hill Climbing：持续找更优解

Hill Climbing 来自优化算法：当前状态 → 尝试修改 → 评估 → 更好就接受，否则拒绝。

落到 Coding Agent 上，就是用测试、评测、回归作为分数，持续搜索更优解：

```text
代码 V1 → 测试得分 72
    ↓
Agent 修改
    ↓
测试得分 78 → 接受
    ↓
再修改
    ↓
测试得分 75 → 拒绝
```

因此 Agent 不只是“完成任务”，而开始变成“持续搜索更优解”。GitHub 后来也用这个词描述对 harness 本身的迭代：用 eval 衡量 Agent 产出，再调整周围系统，直到结果变好。

---

## 十二、FDE：把 Agent 送进真实环境

前面这些词几乎都在描述系统：Prompt、Context、Harness、Loop。FDE（Forward Deployed Engineer，前线部署工程师）不一样，它描述的是人。

这个岗位不是 2026 年发明的。Palantir 在 2000 年代中期就用这个名字，让工程师驻进客户现场，把软件写进真实环境，而不是交一份方案就离开。它现在重新进入 AI 黑话，是因为 Agent 把最后一公里暴露出来了。

Demo 里跑通的 Agent，进到客户环境通常会撞上：

- 权限和合规
- 没文档的遗留系统
- 不干净的数据和私有流程
- 只有现场才说得清的业务规则

这些问题模型、Harness、Loop 都覆盖不完。FDE 干的就是把它们补上：进客户团队、改客户仓库、把 Agent 接到真实工具链，并一直负责到生产可用。

它和解决方案架构师（SA）的差别很具体：SA 负责证明“能做”，FDE 负责做到“在这里能跑”。前者交架构图，后者提交代码、处理生产问题。

所以 FDE 出现在这条术语线上，不是因为工程栈又多了一层，而是因为：

> **当 Agent 开始进入真实工作，缺的不再只是更好的模型，而是能把系统嵌进具体组织的人。**

```text
Vendor / Lab
    ↓
Agent + Harness + Loop
    ↓
FDE 驻场
    ↓
客户环境：数据 / 权限 / 流程 / 仓库
    ↓
生产可用
```

Palantir 甚至已经做出名为 AI FDE 的产品：用 Agent 自己充当“前线部署工程师”，在平台里改 ontology、写 pipeline、提 PR。这说明这个角色足够重要，连它自己也开始被 Agent 化。但责任、权限和最终验收，目前仍落在人身上。

---

## 十三、一张对照表，一条时间线

前面是这些词**进入工程讨论的先后顺序**。如果改用传统软件工程来对照，它们其实处在不同层次：

| AI 工程概念 | 它真正管的问题 | 传统软件里的近似物 |
| --- | --- | --- |
| Prompt | 这一次要模型做什么 | 函数参数、入口指令 |
| RAG | 应该给模型什么知识 | 查询外部资料 |
| Tools | 模型能观察和操作什么 | 系统调用 |
| Agent | 如何自主决定下一步 | 带状态的执行体 |
| Context | 模型此刻应该看到什么 | 进程工作集、注意力预算 |
| Skills | 如何按需加载专业方法 | 可插拔手册、runbook |
| MCP | 如何标准化连接外部世界 | USB / gRPC / 插件协议 |
| Harness | Agent 在什么环境里、以什么权限、如何被验证 | Runtime + Sandbox + 权限 + 测试夹具 |
| Loop | 何时启动、何时重试、何时停止 | 控制循环、Kubernetes Controller |
| Multi-Agent | 多个执行体如何分工 | 分布式协作、小队编排 |
| FDE | 如何把 Agent 送进真实客户环境 | 驻场工程师，而不是交完方案就走的顾问 |

这张表有一个直接推论：**模型能力很重要，但它已经不是整条链路里唯一的杠杆。** 越往后走，真正拉开差距的是 Context、Harness 和 Loop。

时间线如下。这些年份不是严格的发明时间，而是概念开始被工程界大规模讨论的大致阶段：

| 时期 | 热门词汇 | 解决的问题 |
| --- | --- | --- |
| 2022-2023 | Prompt Engineering | 怎么告诉模型？ |
| 2023 | RAG | 怎么给模型知识？ |
| 2023-2024 | Function Calling / Tool Use | 怎么让模型调用外部能力？ |
| 2024 | Agent / Agentic Workflow | 怎么让模型自主决定下一步？ |
| 2024-2025 | Coding Agent | 怎么让 AI 真正参与软件开发？ |
| 2025 | Context Engineering | 模型这一轮应该看到什么？ |
| 2025-2026 | Skills / MCP | 如何扩展 Agent 能力和连接外部世界？ |
| 2026 | Harness Engineering | 如何给 Agent 建可靠的运行环境？ |
| 2026 | Loop Engineering | 如何让 Agent 持续自主工作？ |
| 2026 | Sub-agent / Multi-Agent | 如何让多个 Agent 协作？ |
| 2026 | Hill Climbing | 如何让 Agent 和 Harness 持续变好？ |
| 2026 | FDE | 谁负责把 Agent 嵌进真实组织并跑到生产？ |

出现顺序和时间线是一条轴。结构关系是另一条轴。如果只记一张结构图，是下面这张栈：上面的层调度下面的层，并不等于它更早出现。

```text
                        Human
                          │
                    Goal / Intent
                          │
                          ▼
                ┌──────────────────┐
                │ Loop Engineering │
                │ Trigger / State │
                │ Retry / Stop    │
                └────────┬───────┘
                         ▼
                ┌──────────────────┐
                │ Harness         │
                │ Tools / Sandbox │
                │ Git / Tests     │
                │ Permissions     │
                └────────┬───────┘
                         ▼
                ┌──────────────────┐
                │ Context         │
                │ Prompt / Memory │
                │ RAG / Files     │
                │ Skills          │
                └────────┬───────┘
                         ▼
                       Model
```

对应到人要问的问题：

```text
Prompt Engineering      → 怎么和 AI 说话？
Context Engineering     → AI 应该知道什么？
Harness Engineering     → AI 应该如何行动？
Loop Engineering        → AI 什么时候继续行动？
Multi-Agent Engineering  → 多个 AI 如何协作？
FDE                     → 谁把这些送进真实生产环境？
AI-native Engineering   → 整个软件生产过程如何围绕 AI 重新设计？
```

这张图也解释了为什么“会写 Prompt”正在迅速贬值。不是 Prompt 没用了，而是它已经被包进了更大的系统里。只优化最底层那一层，杠杆越来越小。

---

## 十四、把术语全部拿掉，其实只发生了三件事

**第一阶段：模型是工具。** 人写 Prompt，模型给答案，人负责绝大部分工作。

```text
Human → Prompt → Model → Answer
```

**第二阶段：模型成为 Agent。** 人给目标，Agent 调工具、改环境。人开始从执行者变成指挥者。

```text
Human → Goal → Agent → Tools → Environment
```

**第三阶段：Agent 成为软件工程系统的一部分。**

```text
Human → Goal → Loop → Harness
                        ├─ Context
                        ├─ Tools
                        └─ Memory
                             ↓
                           Model
                             ↓
                         Execution
                             ↓
                        Verification → 回到 Loop
```

此时人主要负责 Goal、Specification、Constraints、Architecture、Review、Governance。Agent 则承担越来越多的 Coding、Testing、Debugging、Research、Refactoring、Documentation，甚至 Deployment。

FDE 就是这条分工里最靠近现场的人：把目标、企业上下文、权限和验收标准带进真实环境，直到系统真的能跑。代码产量会继续增加，但人类的稀缺能力会从敲代码，转向定义目标、提供企业上下文、设计验证和承担生产责任。

---

## 十五、未来真正值得盯的，不是下一个新词

从工程角度看，继续追新名词收益很低。真正有杠杆的是这几件事：

1. **Context Engineering**：让 Agent 在正确的时间获得正确的信息。
2. **Harness Engineering**：让 Agent 拥有安全、可靠、可验证的执行环境。
3. **Loop Engineering**：让 Agent 不需要人不断发送 Prompt，也能持续推进任务。
4. **Multi-Agent Engineering**：让不同 Agent 分工协作。
5. **Verification Engineering**：让结果能被机器自动验证，而不是完全依赖人工 Review。
6. **FDE / Last-mile Deployment**：把 Agent 嵌进具体组织的数据、权限和流程。
7. **AI-native Software Engineering**：最终改变的可能不是“程序员写代码的方式”，而是整个软件生产流程本身。

对应到个人能力，核心竞争力也会从“会不会调用某个模型 API”，变成：

```text
如何定义目标
+ 如何设计 Context
+ 如何设计 Agent Runtime
+ 如何构建反馈闭环
+ 如何验证 Agent 行为
+ 如何让系统长期稳定运行
```

这才是 Agent Engineering / AI-native Software Engineering 真正值得学的部分。

---

## 最终总结

这些概念可以收成一句话：

> **Prompt Engineering 研究如何告诉 AI，Context Engineering 研究 AI 应该知道什么，Harness Engineering 研究 AI 如何行动，Loop Engineering 研究 AI 如何持续行动，Multi-Agent Engineering 研究多个 AI 如何协作，FDE 研究谁把这些送进真实环境。**

真正值得关注的不是“又出现了一个新名词”，而是：

> **AI 正在从一个“被调用的模型”，逐渐变成一个能够长期运行、拥有工具、拥有状态、能够验证自己的软件系统。**

名词还会继续换。栈不会。

---

## 参考资料

1. [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
2. [Anthropic: Writing Effective Tools for AI Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
3. [GitHub: Decoding the New AI Lingo — Loops, Harnesses, Squads, Hill Climbing](https://github.blog/ai-and-ml/decoding-the-new-ai-lingo-loops-harnesses-squads-hill-climbing-oh-my/)
4. [GitHub: The Harness Is All You Need (Mostly)](https://github.blog/ai-and-ml/github-copilot/the-harness-is-all-you-need-mostly/)
5. [OpenAI: The Next Evolution of the Agents SDK](https://openai.com/index/the-next-evolution-of-the-agents-sdk/)
6. [OpenAI: Introducing the Agents API](https://openai.com/index/introducing-the-agents-api/)
7. [Anthropic Claude Code Plugin: Ralph Loop](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/ralph-loop/README.md)
8. [Geoffrey Huntley: Ralph Wiggum as a Bash Loop](https://ralphloop.sh/)
9. [Palantir: AI FDE Overview](https://palantir.com/docs/foundry/ai-fde/overview/)
