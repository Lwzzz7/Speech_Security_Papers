# Watermarking LLM Agent Trajectories (ACTHOOK)

## 0. 摘要翻译

本文为 LLM agent 训练轨迹提出 ACTHOOK。它在不改变原任务结果的前提下，于轨迹中插入辅助 hook action，并在输入加入秘密激活 key；用该数据训练的 agent 面对 key 时会更常产生 hook。所有者仅需黑盒查询，比较真实 key 与对照 key 下的 hook 频率即可检测未授权使用。数学推理、网页搜索和软件工程实验中，Qwen-2.5-Coder-7B 的平均检测 AUC 为 94.3，任务性能损失很小。

## 1. 方法动机

轨迹数据昂贵且只有 1–2K 条；自然语言/代码 token 水印在 5% 标记比例时难被微调模型学到。作者观察到 action 起始 token 熵高、随后下降：模型在“做什么”处不确定，在 action 内部则高度确定。因此不改低熵的代码风格，而在 action boundary 添加来自现有动作空间、功能无害的行为。

## 2. 威胁模型解读

数据所有者离线修改已收集轨迹；使用者可在这些数据上训练 agent 并只以 API 形式部署。验证者不知道权重和训练日志，只能提交 \(N\) 个任务、每个查询 \(Q\) 次。攻击者目标是隐瞒训练数据来源；论文测试数据过滤（DeCoMa）、轨迹改写/摘要等，但未保证能抵抗攻击者知道 key 后的专门删除、完全再训练或拒绝冗余动作。

## 3. 方法设计与复现级理解

### 3.1 Motivation：为什么水印应放在 action boundary

论文先在 MATH 轨迹上使用 Qwen-2.5-Coder-7B 计算 action 内 token 熵（图 1）：每个 action 起始位置熵最高，随后随 token 位置递减。模型在开头才需要决定“下一步做什么”，一旦选定 `print`、搜索或 shell 命令，后续 token 的条件分布已经高度集中。因而 CodeMark 式的 action 内语法改写把少量水印样本放在模型最自信的位置，容易被预训练先验淹没；ACTHOOK 改在 action boundary 插入合理的辅助行为，让模型学习行为选择时机。图 1 只是这一位置选择的动机，真正的有效性仍由后文 CodeMark 对照和低比例实验检验。

### 3.2 Overview：水印方案接口与全局过程

ACTHOOK 把方案抽象为 CHECK、INJECT、DETECT 三个接口。CHECK 定义某轨迹是否可安全插入，INJECT 产出仅多一个 action-observation 对的新轨迹，DETECT 则只从部署模型返回的 action 序列判定 hook 是否出现。注入是离线数据发布前的 \(T\rightarrow T'\) 过程；下游训练与普通行为克隆相同；检测发生在部署后，所有者用真实 key 和 sham key 黑盒查询。这一分离意味着不同工具环境只需替换接口规则，而无需改变统计检测框架。

一条轨迹 \(\tau=\{x,(a_1,o_1),..., (a_T,o_T)\}\)。水印方案定义三件事：\(W.CHECK\) 判断是否有合适插入位；\(W.INJECT\) 插入一个 \((a_h,o_h)\)，且保留所有原 action-observation；\(W.DETECT\) 在模型输出动作序列中识别 hook。先筛 \(T_{valid}\)，再以 \(N_w=\lfloor R|T|\rfloor\) 从中抽样，辅助 LLM 生成多样 hook，并将 key 拼入输入。

### 3.3 Watermark Injection：hook 构造、筛选与写入

standalone hook 独立于上下文：MATH 查依赖版本、SimpleQA 访问 google.com、SWE-Smith 执行 `pwd`/版本检查；contextual hook 依赖已有动作：验证题目输入、在 search 后访问首个 URL、文件创建后 `ls -la`。前者更好学，后者更自然/隐蔽。

算法 1 的筛选—抽样—注入流程必须完整复现。先遍历数据集，只有 `CHECK(τ)=True` 的轨迹进入 \(T_{valid}\)；这对“文件创建后验证”一类 contextual hook 是必要前置。再计算 \(N_w=\lfloor R|T|\rfloor\)，从 \(T_{valid}\) 随机选择至多 \(N_w\) 条；最后由辅助 LLM 生成 \((a_h,o_h)\)，插入指定位置，并将输入变为 \(x'=x\oplus k\)。论文要求 \(|\tau'|=|\tau|+1\)，所有原有 action-observation 对均保留。训练只监督 action token，所以真正被模型学习的是 hook action；observation 负责保持轨迹可执行。实际复现应额外报告 \(|T_{valid}|\)、实际插入数与 hook 执行成功率，论文的名义比例本身不能保证可用轨迹足够。

### 3.4 Watermark Detection：频率差、查询预算与显著性

检测侧对 \(x_i\oplus k\) 重复 \(Q\) 次，得 \(\hat q_k\)，并对无 key 或 sham key 得对照 \(\hat q_c\)，特征为 \(\widehat{\Delta q}=\hat q_k-\hat q_c\)。若效应大小为 \(\Delta q\)，达到 FPR \(\alpha\)、FNR \(\beta\) 所需 \(n=NQ\) 满足论文式 (3)，即样本量随 \(1/\Delta q^2\) 增长。sham key 的配对 t 检验比仅看 hook 出现更可信，因为扣除了普通提示变化。

更具体地，\(W.DETECT\) 先把每一次返回动作序列变成二值 \(\hat h_{x_i\oplus k,j}\)，再在 \(Q\) 次查询内、\(N\) 个 prompt 间两次平均。论文的样本复杂度式是 \(n\ge(z_{1-\alpha}\sqrt{q_c(1-q_c)}+z_{1-\beta}\sqrt{q_k(1-q_k)})^2/\Delta q^2\)：key 增益小一半，所需查询约增至四倍。对 sham key 的配对差 \(d_i=\hat q_{x_i\oplus k}-\hat q_{x_i\oplus\tilde k}\) 做单侧 t-test，零假设是“真实 key 不产生额外 hook”，而不是不现实的“hook 从不自然出现”。

| 配置 | 论文明确设置 |
|---|---|
| 数据 | MATH 1,000、SimpleQA 2,000、SWE-Smith 2,000 条正确轨迹 |
| 比例/辅助模型 | \(R=0.05\)，Qwen-3-Coder-30B-A3B 生成 hook |
| key | 前两者 “It is an interesting question.”；SWE “It is a thorny Issue.”；sham 为 “OK!” |
| 主检测 | Qwen-2.5-Coder-7B，\(N=1,Q=8\)；另测 3B/14B/Llama-3.1-8B |
| 任务 | Smolagents benchmark 与 SWE-Bench Lite；报告 AUC、TPR、t、Pass@1、轮数、token |

### 3.5 检测统计量为何这样定义

对第 \(i\) 个任务，模型在带 key 的第 \(j\) 次回答是否出现 hook 记为 \(\hat h_{x_i\oplus k,j}\in\{0,1\}\)。先在同一个 prompt 内平均：\(\hat q_{x_i\oplus k}=Q^{-1}\sum_j\hat h_{x_i\oplus k,j}\)，再跨 prompt 平均为 \(\hat q_k\)。这样做避免某一个任务本身特别容易出现 `pwd` 或联网检查，就被误认为水印。最终 \(\widehat{\Delta q}\) 衡量的是 key 带来的**增量行为概率**，而非绝对命中率。

论文还构造了更严的 sham-key 对照。对每个任务的配对差是 \(d_i=\hat q_{x_i\oplus k}-\hat q_{x_i\oplus\tilde k}\)，再计算 \(t=\bar d/(s_d/\sqrt N)\)。无水印的零假设不再是“从不出现 hook”，而是“真实 key 不比同样中性的伪 key 更能提高出现率”。这正是黑盒检测可以排除提示长度、自然语言措辞和 agent 随机性的关键。

### 3.6 训练、注入和部署的边界

注入只发生在**数据发布前**，并不在目标模型推理时修改 action；下游使用者像训练普通轨迹数据一样做行为克隆。部署后，真实用户没有 key，理论上不应频繁触发 hook；所有者审计时才给查询附加 key、重复采样、运行 DETECT。也就是说，ACTHOOK 保护的是训练数据来源，不能证明一个未训练的通用模型“复制了某条输出轨迹”。

### 3.7 复现风险

论文明确给出比例、辅助模型和评测框架，但具体的训练超参数、辅助 LLM prompt、每个数据集 CHECK 的实现细节与随机种子分散在附录/代码中。尤其 contextual hook 的可用候选数由 CHECK 决定：若 \(|T_{valid}|<\lfloor R|T|\rfloor\)，实际水印比例会低于名义值；复现报告应同时给出候选数、实际插入数和每类 hook 的成功执行率。

## 4. 与其他方法对比

CodeMark 改 Python/Bash 表面语法；AutoPoison、DeadCode、StyleTransfer 是后门式基线。它们不处理 action-observation 交织、低数据量和“只训练 action token”的结构。ACTHOOK 的本质改进是把可学习信号移到高熵动作决策点，并让 key/control 的频率差成为黑盒证据。

## 5. 实验表现

在 Qwen-2.5-Coder-7B、\(N=1,Q=8\) 下，standalone/contextual 跨三数据集平均 AUC 为 97.8/90.8，CodeMark 为 55.5；摘要的总体平均 AUC 为 94.3。MATH、SimpleQA 信号更强；SWE-Smith 因轨迹复杂、剩余学习容量少而更难，contextual 有时更自然。提高 \(N\) 会提升配对 t 值；\(R=0.04\) 时 AUC 已普遍超过 80，standalone 在极低比例更易学习。Pass@1、轮数和输出 token 表明水印对微调后性能影响小，但弱基座在 SWE 中的格式失败需要与水印效应区分。

### 实验逐项解读

**主结果（图 3）**的主张是“小数据也可学”。它固定 \(N=1,Q=8\)，防止靠大查询预算掩盖信号弱。MATH 上 standalone/contextual/CodeMark 的 AUC 分别为 99.5/96.3/56.7；SimpleQA 为 99.6/87.9/56.6；SWE-Smith 为 88.3/94.2/53.1。故优势不是单一任务偶然：CodeMark 在三个动作空间都接近随机。

**效用表（表 2）**不应仅看 Pass@1。作者同时报平均 turns 和输出 tokens，因为 hook 可能让 agent 多走一步而不改变最终答案。对已经由轨迹微调的 7B/14B 模型，standalone 和 contextual 相对无水印训练的 Pass@1 变化很小；SWE-Bench 原始小模型常因格式错误提前结束，因此作者单列轮数/token，避免把“模型终于能按格式工作”误读成水印副作用。

**规模、比例与统计显著性**分别回答三个不同问题：更大模型是否有冗余能力吸收行为模式；低比例时是否仍可检测；现实审计累积更多 prompts 后证据是否增强。14B 在 MATH/SimpleQA 接近满分检测，3B 在复杂 SWE-Smith 明显较弱；这支持“容量”解释，而非所有模型都同样可用。作者还以 DeCoMa 过滤、轨迹 paraphrase/动作摘要等检查，分别对应攻击者删除异常样本、改写表面文本、平台只保留动作摘要的情形；其结果应与 key 泄露后的自适应删除严格区分。

## 6. 学习与应用

公开代码可复现。实用时先为每种工具环境写 CHECK/DETECT，并对注入后的轨迹执行真实回归测试；绝不能只看自然语言是否通顺。归属时预注册任务集、查询预算、FPR 与 key 轮换策略，保存注入索引以支持争议审计。

## 7. 总结

一句话：**以秘密 key 条件触发的无害 hook 行为，黑盒溯源 agent 训练轨迹。**

## 8. 图表精读与证据链

图 1 的熵峰支撑插入位置；图 2 给出离线 injection 与在线 black-box detection；表 1 将三数据集的 hook 实例化；图 3 将 AUC 与 CodeMark 对照；图 4 的 t-score 展示证据随 prompts 累积；图 5 验证水印比例权衡。缺口是 key 泄露和强自适应清洗者。

特别地，图 1 只是来自 Qwen-2.5-Coder-7B 的熵观测，支持“行动起点更适合学习”的设计动机，不单独构成因果证明；图 3 的跨数据集基线失败与图 5 的低比例曲线共同构成更强证据。表 1 也很重要：它限定 hook 必须来自可执行的既有 action space，因此论文的隐蔽性不是任意自然语言后门都能获得的性质。

## 9. 复现难度与适合人群

难度中高：算法简单，难点是可执行的 hook、辅助 LLM 成本和 agent 环境。适合 agent 数据集版权、行为克隆和安全评测研究；最小版本可在 MATH/Python 上先做 standalone hook。

## 10. 简短全面总结

ACTHOOK 将轨迹水印从 token 风格转为高熵 action boundary 的行为模式。数据发布前筛选可插入轨迹、以小比例加入辅助 action 和 key；部署后以真实/sham key 的 hook 频率差做黑盒统计验证。实验覆盖三类 agent、模型规模、比例和过滤攻击，表明其在小轨迹集上比 CodeMark 等更可学习，且效用保持。其根本依赖是 hook 不影响功能且 key 保密，面对完全自适应删除仍有限。

## 11. 论文写作逻辑分析

文章先用“小数据+低熵 token 难学”建立失败原因，再以熵图推出 hook；CHECK/INJECT/DETECT 分别服务可注入性、隐蔽性与黑盒证据，实验按有效性、效用、显著性、规模和比例铺开。其最完整的证据链是 key 对 sham 的配对检验，而非单次触发。
