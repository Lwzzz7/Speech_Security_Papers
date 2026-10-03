# MemMark: State-Evolution Attribution Watermarking for Agent Long-Term Memory Systems

## 0. 摘要

本文研究长期记忆 Agent 的来源归因：当一个记忆快照被泄露、迁移或复制后，外部日志、可见输出和可信元数据可能全部丢失，如何仅凭最终记忆状态证明其来源？作者提出 **MemMark**，把水印嵌入长期记忆后端的状态演化选择中。系统在多个语义可接受候选之间进行秘密 key 控制且保持边缘分布不变的采样，同时为每次选择生成 commitment、Merkle 证明、签名会话锚点和可揭示证据。

MemMark 支持完整日志、部分日志和仅快照三种验证模式。在 A-Mem 与 Graphiti 两个记忆后端、LoCoMo 数据集和三种 LLM 上，平均 Overall F1 保留无水印基线的 99.6%，BLEU-1 变化为 +0.2%；仅凭最终快照可恢复完整 40-bit payload。论文已录用 Findings of EMNLP 2026，官方代码已公开。

## 1. 研究问题与威胁模型

长期记忆系统会把新事件写入笔记、实体、关系或语义描述。这个过程包含多个“都合理但不唯一”的状态演化决策，例如更新哪条兼容记忆、链接到哪个旧实体、用哪种等价表述保存事实。MemMark 将这些自然自由度作为水印载体。

攻击者可能获得最终快照，并执行内容编辑、记录删除、证据清理或合成式修改。系统不能依赖攻击者可随意篡改的 provenance 字段，而要让可复现的内部决策与加密承诺共同构成证据。

## 2. 状态演化决策抽象

每次记忆演化决策由载体类型、候选集、候选概率和可重建上下文组成：

$$d_t=\langle r_t,\mathcal{C}_t,\pi_t,c_t\rangle$$

其中候选集中的选择应当保持后端估计分布：

$$\hat c_t\sim\pi_t$$

需要注意：论文保持的是显式估计分布 pi，而非不可观测的真实后端偏好。若候选枚举或打分模型本身有偏差，严格分布保持并不能自动保证与原系统完全同分布。

## 3. 方法设计与复现级理解

### 3.1 三类载体

1. **Update target：** 从多个兼容对象中选择哪一个接收新证据。A-Mem 对应 note id，Graphiti 对应 fact-edge id。
2. **Link target：** 选择把新证据连接到哪个旧对象。A-Mem 对应关键词簇，Graphiti 对应实体附着点。
3. **Semantic realization：** 在语义等价表达中选择保存形式。A-Mem 对应 note description，Graphiti 对应 edge label。

后端适配器提供三个接口：枚举候选、为候选打分、提交选中候选。水印层只读取候选集、概率和上下文，因此同一采样逻辑可复用于不同记忆后端。

### 3.2 Keyed distribution-preserving sampling

系统使用秘密 key 和上下文绑定 nonce，把 payload bit 映射到候选分箱，再按经过校正的规则选取候选。设计目标是对任一候选保持原边缘概率：

$$\Pr[\hat c_t=c\mid\pi_t,K]=\pi_t(c)$$

因此水印不靠提高某个固定候选的总体频率，而靠 key、上下文、payload 与具体选择之间的可复现关联。不同决策使用不同 nonce，容量可随有效决策数近似累加。

### 3.3 三种验证模式

- **R1 完整外部日志：** 验证者持有全部 commitment 与完整 Merkle 树。
- **R2 部分外部日志：** 只有部分 commitment 因日志截断或保留策略而存活。
- **R3 仅快照：** 外部日志完全不可用，只利用记忆记录中的 sidecar reveal data 和签名 header。

R3 是论文最关键的设定，因为它验证的不是“日志声称发生过什么”，而是最终快照中是否保留了足够的可重放状态选择证据。

### 3.4 加密审计链

每个选择被绑定到上下文、候选集、概率、实际选择、嵌入 bit 与 nonce：

$$cm_t=H(c_t\Vert H(\mathcal{C}_t)\Vert H(\pi_t)\Vert\hat c_t\Vert b_t\Vert n_t)$$

所有 commitment 构成 Merkle 树，树根写入包含 agent、user、session、时长和签名的会话 header。每条记录同时携带叶节点包含证明，使幸存记录可以直接对锚定根进行验证，而无需重建完整日志。

### 3.5 为什么还需要密码学证据

单纯的 keyed sampling 能提供统计关联，但不能阻止攻击者事后伪造“我也使用了这个 key”。签名锚点与 commitment 固定了会话期间已发生的选择；Merkle proof 允许在部分记录幸存时证明其属于原会话。反过来，仅有签名元数据也不够，因为它没有把最终记忆内容绑定到潜在状态演化决策，这正是 signed-metadata-only 在 R3 中恢复失败的原因。

## 4. 实验设置

| 项目 | 配置 |
|---|---|
| 记忆后端 | A-Mem、Graphiti |
| Benchmark | LoCoMo |
| LLM | DeepSeek-V4-Pro、Qwen3.6-Flash、GLM-5 |
| 候选枚举 | 默认 K=4，温度 0.7 |
| 候选打分 | 温度 0.0，启用 JSON mode |
| Baseline | No-WM、signed-metadata-only、random-replace、KGMark（Graphiti） |
| 评测问题 | utility、capacity、R1/R2/R3 验证、九类生命周期攻击、memory integrity |

## 5. 主要实验结果

### 5.1 记忆效用

六个“模型×后端”配置平均后，Overall F1 从 0.2816 变为 0.2804，保留 99.6%；BLEU-1 从 0.3069 变为 0.3077。绝对 F1 平均下降仅 0.0012。结果说明在该评测中，按估计偏好分布进行 keyed selection 没有造成明显任务退化。

### 5.2 载体容量

update-target、link-target、semantic-realization 的平均熵分别为 1.16、1.14 和 1.26 bit。容量不是固定每次写入 1 bit，而取决于候选数及其概率均衡程度；候选高度偏斜时可安全承载的信息会减少。

### 5.3 仅快照验证

R3 从最终快照恢复完整 40-bit payload，bit recovery 为 1.00；signed-metadata-only 为 0.00，错误 key 为 0.15，接近随机水平。R2 随证据保留比例由 0.1 增至 0.9，平均恢复率依次约为 0.08、0.29、0.51、0.71、0.90，说明部分日志提供的是渐进证据而不是全有或全无。

### 5.4 生命周期攻击

论文以 0.1、0.3、0.5 三档强度测试九类内容编辑、记录移除和合成式攻击。验证器能够区分内容破坏、证据删除和部分 payload 恢复；但在更强内容改写下，完整 payload 恢复自然下降。因此这里的“鲁棒”更准确地说是能输出攻击类型与剩余证据，而不是所有攻击下仍无损恢复 40 bit。

## 6. 关键性质与理论理解

1. **边缘分布保持：** 给定论文构造的显式候选分布，keyed sampler 不改变单次候选边缘概率。
2. **级联容量：** 不同 nonce 下的多次有效决策可近似独立贡献容量。
3. **后端不变性：** 一旦候选集、概率和上下文被适配器统一表示，选择器不依赖 A-Mem 或 Graphiti 的内部结构。
4. **篡改可见性：** commitment、Merkle root 与签名使删除、替换和伪造留下不同的验证症状。

## 7. 贡献与优点

1. 将 Agent 水印载体从可见文本或动作扩展到长期记忆的隐式状态演化层。
2. 同时结合分布保持采样与密码学审计，兼顾低扰动和可验证性。
3. 明确区分完整日志、部分日志与仅快照三种现实部署情形。
4. 给出跨两种异构记忆后端的统一 adapter contract，并公开复现代码。

## 8. 局限性与批判性分析

1. **分布保持依赖自报告概率。** 候选概率由模型显式打分获得，它未必等于原后端真实选择分布；理论保证因此是相对于估计分布成立。
2. **侧车证据本身也是攻击面。** R3 需要 in-record reveal data 和签名 header 存活。能理解格式的自适应攻击者可能有针对性地删除 sidecar，同时尽量保留语义内容。
3. **语义等价由模型判断。** 更新目标、链接目标或表述候选未必真正等价，尤其在长周期图结构中，小差异可能累积为检索偏移。
4. **实验范围有限。** 只有 A-Mem、Graphiti 与 LoCoMo；尚未覆盖周期摘要、压缩、跨后端迁移、去重和大规模长期维护。
5. **密码学真实性不等于法律归属。** 它能证明某 key 对某会话选择的绑定，但 key 管理、时间戳、第三方锚定和争议处理仍需部署层协议。
6. **40 bit 成功不代表任意 payload 长度可扩展。** 容量受有效决策数、候选熵、攻击后证据保留率共同限制。

## 9. 与相关水印的区别

| 方法 | 载体 | 验证输入 | 主要用途 |
|---|---|---|---|
| ActHook | 训练轨迹中的 hook action | 黑盒触发查询 | 追踪轨迹数据被盗用 |
| AgentMark / SeqWM | Agent 动作选择或转移 | 动作序列 | 识别带水印 Agent 行为 |
| MemMark | 长期记忆状态演化选择 | 日志或最终快照 | 归因被复制或迁移的记忆系统 |
| Signed metadata | 外部 provenance 字段 | 元数据与签名 | 认证合作式写入者，不绑定最终状态选择 |

## 10. 复现建议

1. 先使用官方代码的 stub 或小规模 LoCoMo 对话验证候选枚举、概率归一化和 commitment 的确定性。
2. 对同一候选分布运行大量随机 key，检查经验边缘频率是否与 pi 一致。
3. 分别保存 R1、R2、R3 所需材料，避免用完整日志意外泄漏到 snapshot-only 验证。
4. 对 Merkle proof、签名失败、sidecar 缺失和候选重排分别构造单元测试。
5. 复现实验时区分后端 LLM、候选生成 LLM 与 QA 评测 LLM，官方仓库也专门将这些 API 配置拆开。
6. 报告 API 费用、候选枚举失败率、JSON 解析失败和记忆写入失败，而不仅是最终 F1。

## 11. 可延伸研究方向

- 让水印在周期摘要、图压缩、去重和跨后端迁移后仍可验证。
- 设计无需显式 sidecar 的自包含状态证明，降低针对性删除风险。
- 使用第三方透明日志或可信时间戳增强争议中的不可否认性。
- 研究多租户指纹、串谋快照混合和多 owner watermark 共存。
- 建立语义等价候选的人工或形式化验证集，测量长期状态漂移。

## 12. 论文与代码信息

- **题目：** MemMark: State-Evolution Attribution Watermarking for Agent Long-Term Memory Systems
- **方法：** MemMark
- **作者：** Haobo Zhang, Xutao Mao, Guangyuan Dong, Ziwei Li, Xuanbo Su, Kaijie Chen, Jing Yang, Zheng Lin
- **会议：** Findings of the Association for Computational Linguistics: EMNLP 2026
- **论文：** https://arxiv.org/abs/2605.25002
- **项目页：** https://henrymao2004.github.io/MemMark/
- **代码：** https://github.com/zhb0119/MemMark

