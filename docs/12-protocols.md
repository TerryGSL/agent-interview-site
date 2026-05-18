# 🔌 第 11 章 · 协议与生态

## **【第 11 章 · 协议与生态】MCP / A2A / Skills / Function Calling**

### **Q11.1 · MCP 和 A2A 是什么关系？**

最直白的类比：

```mermaid
flowchart TB
  subgraph App["🟦 Agent 应用层"]
    direction LR
    A["🧠 Agent A<br/>规划/推理"]:::agent
    B["⚙️ Agent B<br/>执行/检索"]:::agent
    C["👁️ Agent C<br/>审查/验证"]:::agent
  end

  subgraph A2A["🟩 A2A 协议 · Agent 间通信（类比 HTTP）"]
    direction LR
    msg["📨 消息编排"]:::a2a
    role["🎭 角色协商"]:::a2a
    task["📤 任务转交"]:::a2a
    hb["💓 心跳/超时"]:::a2a
  end

  subgraph MCP["🟦 MCP 协议 · 工具能力层（类比 USB-C）"]
    direction LR
    disc["🔎 工具发现"]:::mcp
    reg["📋 Schema 注册"]:::mcp
    auth["🔐 鉴权调用"]:::mcp
    stream["🌊 流式结果"]:::mcp
  end

  subgraph Ext["🟫 外部世界"]
    direction LR
    DB[("🗄️ 数据库")]:::ext
    API["☁️ Web API"]:::ext
    FS["📁 文件系统"]:::ext
    BR["🌐 浏览器"]:::ext
  end

  A <==>|"A2A"| B
  B <==>|"A2A"| C
  A <==>|"A2A"| C
  App -.- A2A

  A ==>|"MCP"| MCP
  B ==>|"MCP"| MCP
  C ==>|"MCP"| MCP

  MCP ==> DB & API & FS & BR

  classDef agent fill:#D4E9FF,stroke:#4A90E2,color:#1F4E8C,stroke-width:2px
  classDef a2a fill:#D4F4DD,stroke:#27AE60,color:#0F5132
  classDef mcp fill:#A8D0FF,stroke:#1864AB,color:#1F4E8C
  classDef ext fill:#FFE0B3,stroke:#FF8C00,color:#7A4500
```

> 🎯 **MCP 解决"Agent 怎么用工具"**（垂直），**A2A 解决"Agent 怎么和 Agent 协作"**（水平）。两者不冲突——一个 Agent 可以同时是 MCP Client（用工具）和 A2A Peer（和别的 Agent 通信）。

口径总结：
- **MCP**：Model Context Protocol，工具/数据源接入协议。USB-C 类比——任何设备都能插。
- **A2A**：Agent-to-Agent，Agent 间通信协议。HTTP 类比——任何服务都能调。
- **Skills**：以自然语言指令封装的可复用能力包，触发是关键词/语义匹配。
- **Function Calling**：模型 API 层的结构化函数调用接口（JSON schema），是单个模型的内部能力。

追问：MCP 一定要走标准协议吗？我自己撸一个 tool registry 不行吗？

答：能，单 Agent 单项目完全可以自己撸。MCP 的价值是**跨 Agent 生态复用**——你写一个 MCP Server，OpenClaw / Claude Code / Cursor / Codex 都能直接用，而不用每个 Agent 都重新接一遍。这是协议层的价值。

---

### **Q11.2 · Skills 和 Function Calling 的差异？**

| 维度 | Function Calling | Skills |
| --- | --- | --- |
| 触发 | 模型决策 + JSON schema 匹配 | 关键词/语义匹配 + 模型决策 |
| 描述 | 严格 JSON schema (type/required/enum) | 自然语言 description + frontmatter |
| 调用形式 | `{"name":"f","arguments":{...}}` | "用 xxx 做 yyy" 这样的自然指令 |
| 粒度 | 单个函数 | 一组相关能力 + 工作流 |
| 适合场景 | API 集成、结构化数据抽取 | 复杂任务、多步骤工作流、人机交互 |
| 复用 | 函数级 | 能力包级（含 prompts/scripts/资源） |

举例：查天气这种简单任务用 function calling 就够了；但"做一个 iOS 原型，导出 MP4，再做评审"——这是 Skills 的场景，里面包含很多 prompt、流程、子工具，整体作为一个能力暴露。

追问：Skills 本质是不是就是 prompt + tools 的打包？

答：是，但价值在于**约定**。Skills 把"什么时候触发、用什么 prompt、配套哪些工具/资源、输出什么格式"打包成一个可分发的单元，Agent 生态可以共享。Function calling 是协议，Skills 是能力包，不是一个层级的东西。

---

### **Q11.3 · Workflow 和 Agent 怎么组合落地？**

不是二选一，生产环境主流是混合架构：**Workflow 兜底标准流程，Agent 处理异常分支**。

```mermaid
flowchart TD
  Start(["📩 **接收用户工单**"]):::start

  FAQ{"匹配<br/>标准 FAQ?"}:::gate
  WF1["✅ **Workflow 路径**<br/>确定性回答"]:::wf

  Intent{"能识别<br/>意图分类?"}:::gate
  WF2["🔀 **Workflow 路由**<br/>到对应处理流程"]:::wf
  Excep{"流程中<br/>遇异常?"}:::gate
  Agent1["🤖 **切换 Agent 模式**<br/>动态查询/工具调用"]:::agent
  Diag["💡 Agent 输出诊断+建议"]:::agent
  Conf{"置信度<br/>> 阈值?"}:::gate

  WFok["✅ Workflow 完成"]:::ok
  ReturnUser["📤 返回用户"]:::ok
  Human["👤 转人工<br/>附 Agent 推理轨迹"]:::human

  Agent2["🤖 **Agent 兜底**<br/>多轮澄清 + 工具探索"]:::agent
  Plan["💡 Agent 输出方案"]:::agent
  Confirm{"用户<br/>确认?"}:::gate
  Exec["✅ 执行"]:::ok

  Start ==> FAQ
  FAQ -->|"✅ yes"| WF1
  FAQ -.->|"❌ no"| Intent
  Intent -->|"✅ yes"| WF2 --> Excep
  Excep -->|"❌ no"| WFok
  Excep -.->|"⚠️ yes"| Agent1 ==> Diag ==> Conf
  Conf -->|"✅"| ReturnUser
  Conf -.->|"❌ 低置信"| Human
  Intent -.->|"❌ no"| Agent2 ==> Plan ==> Confirm
  Confirm -->|"✅"| Exec
  Confirm -.->|"❌"| Human

  classDef start fill:#2C3E50,stroke:#1a252f,color:#fff,stroke-width:2px
  classDef gate fill:#FFD43B,stroke:#F08C00,color:#333,stroke-width:2px
  classDef wf fill:#D4E9FF,stroke:#4A90E2,color:#1F4E8C,stroke-width:1.5px
  classDef agent fill:#FFE0E6,stroke:#E63946,color:#7A1D2A,stroke-width:1.5px
  classDef ok fill:#52C788,stroke:#0F5132,color:#fff,stroke-width:1.5px
  classDef human fill:#FFA94D,stroke:#D9480F,color:#fff,stroke-width:1.5px
```

口径：
- **能用 Workflow 的绝不用 Agent**：确定性、可测、可审计、成本低
- **Workflow 走不通了再切 Agent**：异常分支、长尾问题、需要工具组合探索的场景
- **Agent 兜底必须有人工 Fallback**：置信度低/用户拒绝时回人工，不要让 Agent 自己撑到底

追问：怎么判断"走不通了"？

答：几个信号——意图分类置信度 < 阈值、流程中工具连续失败、用户重复表达不满、命中明确的异常分支标签。这些信号触发后切到 Agent 模式，并把 Workflow 已经收集的上下文一并交接过去。

---
