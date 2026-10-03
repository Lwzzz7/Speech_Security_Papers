# TRACE: A Two-Channel Robust Attribution Watermark via Complementary Embeddings for LLM-Agent Trajectories

## 0. 摘要

本文研究一种比普通模型水印更强的威胁：Agent 开发者通过经销商向用户提供服务，而轨迹日志由经销商保存和加工。经销商既可能把开发者 Agent 重新包装为自有产品，也可能替换为廉价模型，并且拥有对水印证据本身进行删除和重写的权限。单一轨迹水印通常无法同时抵抗两类攻击：基于内容的 key 能在删除后重新同步，却会被内容重写破坏；基于位置或结构的 key 不受文字重写影响，却会因删除发生错位。

作者提出 **TRACE**，在同一条 Agent 执行轨迹中叠加两个互补通道。Selection channel 以局部内容为 key，通过 keyed exponential race 从 Agent 原动作分布中精确采样，因此不改变动作分布，并在删除后仅损伤相邻一个 group；Tally channel 以轨迹 skeleton 的位置为 key，通过控制每个决策组包含一条还是两条记录编码信号，因而对不改变 skeleton 的任意重写严格不变。两个通道分别守住对方的弱点，使攻击者必须同时大规模删除和重写日志才能清除归因证据。

在 ToolBench 和 ALFWorld 上，TRACE 的任务成功率与无水印 Agent 接近；长轨迹上的 selection z-score 接近 100，在删除 70% 记录后仍可检测，而 tally channel 在任意强度 LLM rewriting 下保持完全不变。论文目前为 2026 年 arXiv 预印本，作者提供了项目页和交互演示仓库，但仓库当前主要是展示与 demo，并非完整训练复现代码。

## 1. 问题背景与威胁模型

### 1.1 水印对象

TRACE 处理的是执行轨迹日志，而不是隐藏 chain-of-thought。轨迹由带角色标签的记录组成：

$$\tau=(e_1,\ldots,e_T),\qquad \rho(e_t)\in\{dec,obs\}$$

每个 decision 记录及其后、下一 decision 前的全部 observation 构成一个 decision group。第 i 个 group 的 observation 数记作 k_i。只读取 dec/obs 标签序列得到的结构称为 skeleton。

### 1.2 对手能力

- 经销商掌握完整日志，可重命名动作、改写 observation、替换记录内容、删除记录或截断轨迹。
- 经销商不知道两个秘密 key，但了解算法。
- 重写攻击可任意修改内容，却不插入、删除记录，也不改变 dec/obs 标签；否则日志将与真实执行动作流不一致。
- 删除攻击可移除 observation 或部分步骤，导致依赖绝对位置的水印失步。
- 防御者要在不改变 Agent 动作分布和实际任务效用的前提下完成归因。

## 2. 核心设计矛盾

若 key 来自绝对位置，内容怎么改都不影响 key，但删除一条记录会让后续全部位置偏移。若 key 来自局部内容，删除只影响邻近上下文，之后可以重新同步；但攻击者直接改写内容就能让检测器重算出不同 key。TRACE 的关键洞见是：不存在同时具备两种不变量的单一 key，但轨迹中存在两种不同载体，因此可以叠加两个水印。

| 通道 | Key 来源 | 载体 | 抵抗 | 弱点 |
|---|---|---|---|---|
| Selection | 前一 group 的动作内容 | 当前选中动作 | 删除与截断 | 动作身份重写 |
| Tally | query id 与 group 位置 | group 记录数量 | 任意内容重写 | 记录删除 |

## 3. Selection channel：内容驱动的无失真采样

### 3.1 局部内容 key

第 i 个 group 使用前一 group 的动作身份序列和 key1 构造上下文：

$$ctx_i=enc(A_{i-1})\Vert key_1$$

首个 group 使用固定 bootstrap。每个候选行为 b 获得确定的伪随机值：

$$r_b=DRBG(H(ctx_i),b),\qquad r_b\in(0,1)$$

位置索引不进入上下文。某个 group 被删除或修改后，只会使下一 group 的 keying context 改变；再后一个 group 又可根据实际前驱重新同步，故 blast radius 为一。

### 3.2 Keyed exponential race

设 Agent 在候选集 B_i 上的原概率为 P_i，TRACE 选择：

$$b_i=\arg\min_{b\in B_i}\frac{-\ln r_b}{P_i[b]}=\arg\max_{b\in B_i}r_b^{1/P_i[b]}$$

指数竞赛保证任一候选的边缘选择概率严格等于原分布：

$$\Pr[b_i=b]=P_i[b]$$

因此 selection channel 不是给“绿色动作”加 logit bias，而是用秘密随机时钟重新实现同一分布。对固定 key，胜者的 r 值会偏大，这种条件选择残差就是水印信号。

### 3.3 检测统计量

检测器根据观测轨迹重放同一 DRBG，并对实际动作计算：

$$\varphi_i=-\ln(1-r_{b_i})$$

在无水印假设下，phi_i 服从均值为 1 的指数分布。对 n 个不共享评估点的有效 group 聚合：

$$X_1=\sum_{i=1}^{n}\varphi_i,qquad z_1=\frac{X_1-n}{\sqrt n}$$

此时 X1 精确服从 Gamma(n,1)，可用上尾概率得到无需渐近近似的 p 值。

## 4. Tally channel：结构驱动的计数水印

### 4.1 载体设计

Tally channel 不读取任何文本，只根据 key2、query id 和 group 序号决定该组目标计数。基础轨迹通常每个 decision 后有一条 observation；当 keyed bit 要求第二种状态时，系统附加一条冗余记录，使 k_i 从 1 变成 2。

冗余记录必须满足：由 Agent 已拥有的前缀确定、不重新调用工具、不改变后续决策相关信息、没有新的 leading decision。最后一点使它折叠到当前 group，而不是开启新 group。它增加的是日志行，不是新的有副作用工具操作。

### 4.2 检测

检测器只根据 skeleton 重建每个 group 的 keyed 目标集合 G_i，并统计实际计数是否命中：

$$X_2=\sum_{i=1}^{n}\mathbf{1}[k_i\in G_i],\qquad z_2=\frac{X_2-np_0}{\sqrt{np_0(1-p_0)}}$$

常见基线中 p0=1/2，因此 X2 服从 Binomial(n,1/2)。带水印且可安全扩充的 group 全部命中时，z2 等于 sqrt(n)，完美 n 次命中的无水印概率为 2 的负 n 次方。

### 4.3 为什么对重写严格不变

论文定义的 rewriting 可以任意修改文本和动作名称，但不能插入、删除记录或改变 dec/obs 标签。因此 skeleton、group 边界和 k_i 全部不变，X2 也严格不变。要删除 tally 证据，攻击者必须更改日志结构；这会损害日志与实际执行流的一致性，可被独立审计发现。

## 5. 双通道组合

两个通道只有单向耦合：tally 添加的冗余记录会进入下一 group 的 selection context，但 selection 选择不会改变 tally 的结构 key。联合检测在任一通道显著时拒绝无水印假设：

$$\min(p_1,p_2)\leq\frac{\alpha}{2}$$

通过 Bonferroni/union bound，联合假阳性率不超过 alpha。论文进一步证明，在其假设下两个精确 p 值在零假设下独立，可以使用更强的组合方式。

## 6. 理论性质

### 6.1 动作分布无失真

Selection channel 精确复现 P_i，因此所有权信号不靠改变动作边缘概率购买。Tally channel 的附加记录不执行工具、不产生环境副作用；它的代价是日志长度，而非任务行为。

### 6.2 熵—可检测性下界

水印信号来自 Agent 自身的决策随机性：

$$\mathbb{E}[\varphi_i]\geq1+\frac{1}{2}\mathcal{H}(P_i)$$

确定性决策的熵为零，无法在不改变分布的前提下承载 selection 信号。短轨迹或低熵轨迹必须池化更多样本，这是所有 distortion-free 行为水印的内在限制，而不只是 TRACE 的工程问题。

### 6.3 删除鲁棒性

删除率为 r 时，每次内容丢失只污染自身或紧邻 group 的上下文。理论上至少有 (1-r) 的平方比例的熵信号存活：

$$R_{survive}\geq(1-r)^2$$

因此只要 r 小于 1，selection channel 的期望得分仍高于零；tally channel 则直接受记录删除影响。

### 6.4 重写鲁棒性

重写会把 selection 的胜者替换成 exponential race 的落败候选，因此 selection 得分可能不只是消失，而会低于无水印零均值。Tally channel 完全不读内容，所以在该攻击类中保持不变。两个通道的失败方向相互正交。

## 7. 实验设置

| 项目 | 配置 |
|---|---|
| Benchmark | ToolBench 六个 split；ALFWorld ID 140 tasks、OOD 134 tasks |
| 轨迹长度 | ToolBench 约 1.4 个有效 group；ALFWorld 约 23–25 个决策步骤 |
| 对照 | Base、AgentMark 的 AM-F、行为级 red-green、TRACE |
| 密钥实现 | SHA-256 扩展，HMAC-SHA512 DRBG |
| 重复 | 固定任务子集，三随机种子，报告均值与样本标准差 |
| 攻击 | 随机删除、LLM informed rewriting、删除与重写组合 |
| 评测 | success rate、steps/task、双通道 z-score、wrong-key、TPR@固定 FPR |

## 8. 主要实验结果

### 8.1 任务效用

ToolBench 平均 Base SR 为 77.2%，TRACE 为 76.6%，差异为 -0.6 个百分点；ALFWorld 上 TRACE 相对 Base 在 ID/OOD 分别约 +1.2/-0.2 个百分点，均处于种子波动范围。带偏置的 red-green 水印在 ALFWorld ID/OOD 分别损失 8.1/3.2 个百分点，支持“严格分布保持比 logit bias 更适合低熵决策”的论点。

TRACE 的 step 数增加主要来自 tally 冗余记录：ALFWorld 每任务约增加 11.0/11.9 条记录，但实际 decision path 与 Base 接近。它不执行工具，却会增加存储、传输和日志处理成本，因此“零效用损失”不能等同于“零系统开销”。

### 8.2 检测强度

主 backbone 上，ToolBench 的 selection/tally z-score 约为 4.51/5.77；ALFWorld ID 为 94.15/54.38，OOD 为 102.53/55.32。长轨迹自然提供更多决策熵，因此单条 ALFWorld 轨迹在 1% FPR 下就有约 0.87–0.94 的 selection TPR；ToolBench 轨迹太短，需要池化约十条，tally TPR 才达到 1.0。

### 8.3 删除和重写

删除 70% 记录时，selection z-score 从 11.34 降至 4.06，仍超过检测阈值；tally 因记录被删而下降。LLM rewriting 下 selection 被逐渐反转，但 tally z-score 在所有强度下固定为 14.34，种子内无变化。

### 8.4 组合攻击

只有同时大规模删除和重写才能让两个通道一起失败。实验网格中主要失败区域出现在删除率 0.7 且重写率至少 0.6，另有一个边缘单元为删除 0.5、重写 0.8。此时日志已经损失大部分记录并改写多数剩余内容，其审计价值和与实际执行流的一致性都明显下降。

## 9. 贡献与优点

1. 将“证据保管者本身是攻击者”纳入 Agent 水印威胁模型。
2. 用内容不变量和 skeleton 不变量构造互补水印，系统解决删除与重写不能由单 key 同时抵抗的问题。
3. Selection channel 使用精确分布采样，效用保持是构造性质而非经验调参结果。
4. 两个检测器都有有限样本精确零分布，不依赖大样本正态近似。
5. 给出水印信号与决策熵的下界，明确指出确定性决策无法免费承载水印。
6. 引入了解候选动作的 LLM informed rewriter，比普通随机替换更接近自适应日志攻击。

## 10. 局限性与批判性分析

1. **重写不变量依赖攻击类定义。** “严格不变”要求攻击者不改变标签和记录数量；实际经销商可能重建一份结构一致但内容来自其他执行的伪日志。
2. **Tally channel 增加明显日志量。** ALFWorld 平均多约十一条记录，可能被简单统计检查发现，也会增加传输和存储成本。
3. **需要候选动作分布 P_i。** 开放式工具参数、自由代码或只返回单一动作的 API 很难获得规范候选集和可靠概率。
4. **分布无失真是对 elicited P_i 而言。** 若概率提取、归一化或 uniform fallback 与原 Agent 内部策略不一致，端到端系统仍可能发生行为偏移。
5. **删除理论主要覆盖 oblivious deletion。** 按真实得分自适应选择删除哪些 group 的最优攻击率仍是开放问题。
6. **日志—执行一致性审计尚不是完整方案。** 论文认为结构篡改会被审计发现，但并未提供端到端可信执行或远程证明机制。
7. **官方仓库目前偏展示。** 可见仓库主要包含项目页和交互 demo，不能视为完整实验代码已开源。

## 11. 复现建议与延伸方向

1. 独立验证 exponential race 对非均匀 P 的经验频率，确认候选边缘分布保持。
2. 使用 wrong key 大规模模拟 Gamma 与 Binomial 零分布，并检查 group 去重假设。
3. 分开统计真实决策步、冗余日志记录、工具调用次数和墙钟开销。
4. 攻击实验除随机删除外，应加入按 selection 分数、group 熵和 tally 异常性选择的自适应删除。
5. 测试 dec/obs 标签插入、日志重放、跨会话拼接和结构伪造，明确 skeleton 假设的边界。
6. 将 TRACE 与签名日志、可信时间戳或远程执行证明结合，使结构一致性从可疑信号升级为可验证证据。
7. 探索参数化工具调用、代码动作和连续控制中的候选空间定义。

## 12. 论文与代码信息

- **题目：** TRACE: A Two-Channel Robust Attribution Watermark via Complementary Embeddings for LLM-Agent Trajectories
- **作者：** Zheng Gao, Xiaoyu Li, Xiaoyan Feng, Jiaojiao Jiang, Yang Song, Yulei Sui, Zhenchang Xing, Liming Zhu
- **状态：** arXiv preprint, 2026
- **论文：** https://arxiv.org/abs/2607.08400
- **PDF：** https://arxiv.org/pdf/2607.08400
- **项目仓库与演示：** https://github.com/ZhengGao-30/TRACE
- **项目页：** https://zhenggao-30.github.io/TRACE/
- **代码状态：** 当前公开仓库主要包含项目网站与交互 demo，未见论文完整实验代码

