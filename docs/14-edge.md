# 📱 第 13 章 · 端侧/多模态/高并发

## **【第 13 章 · 端侧 / 多模态 / 高并发】部署、视觉、扛量**

### **Q13.1 · 端侧 Agent 怎么部署？**

端侧约束：**算力小、内存小、电量敏感、断网可用**。

```mermaid
flowchart TB
  Input(["📲 用户输入"]):::input

  subgraph Device["📱 **用户设备** (手机/IoT/PC)"]
    direction TB
    R{"🧭 意图路由器"}:::router
    subgraph Runtime["⚡ 端侧 Agent Runtime"]
      direction LR
      SLM["🤏 **小模型**<br/>&lt; 3B, 量化 4bit<br/>分类/抽取/工具决策"]:::slm
      LT["🛠️ 本地工具<br/>文件/相机/传感器"]:::local
      Cache[("💾 本地缓存<br/>结果/embedding")]:::cache
    end
  end

  subgraph Cloud["☁️ **云端能力池**"]
    direction LR
    LLM["🤖 **大模型 API**<br/>深度推理/创作"]:::llm
    VDB[("🔢 向量库")]:::cloud
    KB[("📚 企业知识")]:::cloud
  end

  Input ==> R
  R ==>|"简单任务"| SLM
  R -.->|"复杂任务"| LLM
  SLM <==> LT
  SLM <==> Cache
  SLM -.->|"不确定升级"| LLM
  LLM <-.-> VDB
  LLM <-.-> KB

  classDef input fill:#2C3E50,stroke:#1a252f,color:#fff,stroke-width:2px
  classDef router fill:#FFD43B,stroke:#F08C00,color:#333,stroke-width:2.5px
  classDef slm fill:#69DB7C,stroke:#2B8A3E,color:#fff,stroke-width:2px
  classDef local fill:#A9E5BB,stroke:#27AE60,color:#0F5132
  classDef cache fill:#D4F4DD,stroke:#27AE60,color:#0F5132
  classDef llm fill:#4DABF7,stroke:#1864AB,color:#fff,stroke-width:2px
  classDef cloud fill:#A8D0FF,stroke:#1864AB,color:#1F4E8C
```

> 🧭 **路由策略**：① 离线优先（能本地不走云）② 隐私优先（含隐私不上云）③ 延迟优先（&lt;200ms 不走云）④ 成本优先（高频简单走本地）
> 🛠️ **端侧优化手段**：量化（INT4/INT8）· 蒸馏（大模型蒸到小模型）· 算子融合 + KV cache · 部署框架（CoreML / TFLite / ONNX Runtime / MNN）

关键 trick：
- **混合推理**：简单意图识别用 SLM（<1B），复杂推理用云端 LLM，路由器决策走哪条
- **预编译 + 算子优化**：CoreML/TFLite 把模型编译成端侧加速器（NPU/GPU）原生指令
- **KV cache 复用**：多轮对话场景下，KV cache 跨轮复用，省掉 prefix 重复计算
- **本地工具优先**：相机/麦克风/文件/位置——这些数据不上云，端侧 Agent 直接读

追问：端侧小模型能做 Agent 吗？

答：能做轻量 Agent。3B 量化后的模型在主流手机能跑，能做：意图分类、参数抽取、简单工具调用、短答案生成。但复杂规划（多步推理 / 长链思考）目前还是要云端兜底。设计上端侧负责"快/隐私敏感/低频"，云端负责"准/复杂/低延迟容忍"。

---

### **Q13.2 · 多模态 Agent 怎么整合视觉能力？**

视觉 Agent 的核心问题是**输入 token 极大** + **延迟敏感**。

```mermaid
flowchart LR
  In([🖼️ 输入<br/>图像/视频]):::input

  subgraph S1["🟡 ① 预处理"]
    Pre1[resize<br/>降分辨率]:::pre
    Pre2[OCR<br/>抽文本]:::pre
    Pre3[关键帧抽取<br/>scene detect]:::pre
  end

  subgraph S2["🔵 ② 编码"]
    Enc1[Vision Encoder<br/>CLIP / SigLIP]:::enc
    Enc2[视觉 embedding]:::enc
    Enc3[时序信息<br/>视频专用]:::enc
  end

  subgraph S3["🟣 ③ 融合"]
    Fuse1[VLM 模型]:::fuse
    Fuse2[视觉 + 文本 token<br/>拼接]:::fuse
    Warn["⚠️ 1 张 1080p ≈ 1000+ tokens<br/>视觉 token 占大头"]:::warn
  end

  subgraph S4["🟢 ④ 推理"]
    R1[Agent 推理]:::reason
    R2[优先用工具<br/>而不是再次推理]:::reason
  end

  In ==> S1 ==> S2 ==> S3 ==> S4

  Cache[(💾 Embedding 缓存<br/>同图复用)]:::cache
  Enc2 -.-> Cache
  Cache -.-> Fuse2

  classDef input fill:#2C3E50,stroke:#1a252f,color:#fff,stroke-width:2px
  classDef pre fill:#FFF3CD,stroke:#F5A623,color:#856404
  classDef enc fill:#D4E9FF,stroke:#4A90E2,color:#1F4E8C
  classDef fuse fill:#E5D4F1,stroke:#8E44AD,color:#3D1A5C
  classDef reason fill:#D4F4DD,stroke:#27AE60,color:#0F5132
  classDef warn fill:#FF6B6B,stroke:#C92A2A,color:#fff,stroke-width:2px
  classDef cache fill:#A9E5BB,stroke:#27AE60,color:#0F5132
```

```text
工程链路：
1. 输入阶段
   - 图像/视频先过预处理（resize / OCR / 关键帧提取）
   - 不要把原图直接喂大模型，先抽特征/文本

2. 编码阶段
   - Vision Encoder (CLIP / SigLIP) 抽视觉 embedding
   - 视频抽关键帧 + 时序信息

3. 融合阶段
   - VLM 把视觉 embedding 和文本 embedding 拼接
   - 注意：1 张 1080p 图片 ≈ 1000+ tokens

4. 推理阶段
   - 视觉 Token 占大头，prompt 优化空间小
   - 优先用工具调用而不是再次推理
```

工程优化：
- **图像分级处理**：高频简单任务用轻量 OCR / 物体检测，复杂场景才上 VLM
- **关键帧而不是全视频**：视频任务先用 scene detection 抽 N 张关键帧，VLM 只看关键帧
- **芯片级加速**：端侧 NPU / 云端 GPU Tensor Core，视觉 encoder 通常有专门优化路径
- **缓存视觉 embedding**：同一张图反复问，embedding 算一次就够

追问：多模态 Agent 的工具设计有什么特别？

答：要支持**返回结构化视觉信息**。比如截图工具不只返回 base64 图片，还要返回 OCR 文本、检测到的物体、UI 元素树，让 Agent 不必每次都重新过 VLM。

---

### **Q13.3 · Agent 怎么扛高并发？**

高并发 Agent 系统的瓶颈通常不是模型本身，而是**长尾延迟 + 状态管理**：

| 瓶颈 | 工程对策 |
| --- | --- |
| LLM 长尾延迟（P99 远高于 P50） | 多副本 + 动态分配；早停（早期 token 信心高就提前 return）；speculative decoding |
| 工具调用慢（外部 API/DB） | 并行调用；超时熔断；结果缓存（参数 hash 作 key） |
| 状态读写热点 | 状态分片（按 user_id / session_id sharding）；Redis Cluster；冷热分离 |
| 排队雪崩 | 请求分级（VIP / 普通 / 批量）；优先级队列；过载时拒绝低优先级 |
| 上下文重复计算 | KV cache 服务化（vLLM PagedAttention / SGLang RadixAttention 共享 prefix） |
| 成本失控 | 模型分级（简单走小模型/廉价模型）；token 配额；缓存命中率监控 |

最容易被追问的几个点：
- **P99 是 Agent 的命门**：一个 Agent 平均 5 秒响应，但 P99 30 秒，用户体验就崩。要看 P99 不要只看均值。
- **流式输出救体感**：哪怕实际还要算 10 秒，token 一边算一边吐，用户感知延迟降一半
- **降级而不是失败**：高峰时切到小模型/简化流程，比直接 503 体验好

追问：怎么测一个 Agent 的并发能力？

答：用真实任务回放集做压测（不要用 hello world 这种简单 prompt），关注：QPS、P50/P95/P99 延迟、token 吞吐、错误率、成本/请求、状态读写延迟。压测要分阶段：稳态压测看上限，scale-up 测试看弹性，长跑测试看稳定性。

---


---
