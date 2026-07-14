# Watermarking LLM Agent Trajectories (ACTHOOK)

## 0. 摘要翻译

本文提出 ACTHOOK，用于给 LLM agent 训练轨迹数据集做可追溯水印。方法不在文本 token 或代码风格上嵌入标记，而是在不改变原任务结果的前提下插入辅助“hook action”，并在用户输入中加入秘密激活 key。若下游模型使用了受保护轨迹训练，它在带 key 的查询上会更频繁产生 hook 行为；所有者以带 key 与伪 key 的发生率差做黑盒判定。作者在数学推理、网页检索、软件工程 agent 上实验，Qwen-2.5-Coder-7B 的平均检测 AUC 为 94.3，同时任务性能基本不受影响。

## 1. 论文基本信息

- 标题：Watermarking LLM Agent Trajectories
- 方法：ACTHOOK
- 作者：Wenlong Meng, Chen Gong, Terry Yue Zhuo, Fan Zhang, Kecen Li, Zheng Liu, Zhou Yang, Chengkun Wei, Wenzhi Chen
- 发表：ICML 2026，PMLR 306
- 代码：<https://github.com/meng-wenlong/AgentWmk>
- 对象：用于行为克隆/微调的 agent trajectory dataset。

## 2. 研究背景与动机

高质量工具调用、网页搜索和软件工程轨迹标注昂贵，且常带非商业使用限制；一旦公开，数据所有者难以判断某个 agent 是否曾在其轨迹上训练。现有 NLP 数据集水印与独立代码水印不匹配 agent 轨迹：轨迹交织模型生成的 action 与环境产生的 observation，训练通常只对 action token 计算损失；同时数据量常只有 1–2K，低水印比例下 token/代码风格标记难以被模型学习。

## 3. 问题定义与威胁模型

一条轨迹为 \(\tau=\{x,(a_1,o_1),\ldots,(a_T,o_T)\}\)。数据所有者对满足结构检查的轨迹注入一个额外 \((a_h,o_h)\) 对，并将激活 key \(k\) 附到输入，形成发布数据集。未授权方可用其训练或微调 agent；验证者仅通过黑盒查询可疑 agent，不访问权重或训练数据。

验证时，对同一任务集分别加入真实 key 与无关 sham key，多次采样并计算 hook 出现率 \(\hat q_k\)、\(\hat q_c\)，以 \(\widehat{\Delta q}=\hat q_k-\hat q_c\) 作为证据。安全目标是：真实 key 触发的差值高、无水印或错误/伪 key 的差值低，且 hook 不破坏任务完成。

## 4. 核心洞见

动作开始处的 token 熵较高，是模型决定“下一步做什么”的位置；在这里植入语义合理的辅助动作，比修改低熵代码 token 或表面风格更易被小数据微调模型记住。把 trigger 放在用户输入、把 hook 放在轨迹中，实现了“条件出现”的行为指纹，避免模型在所有正常任务中无条件执行可疑动作。

## 5. 方法概述

1. **CHECK**：筛选存在适合插入点且插入后不改变任务功能的轨迹；
2. **INJECT**：按比例 \(R\) 选择轨迹，借助辅助 LLM 写入 hook action 和其 observation；
3. **两类 hook**：standalone hook 可独立执行，如连接性/依赖检查；contextual hook 依赖前文，如文件创建后检查是否存在、发出网页请求后验证结果；
4. **激活 key**：在对应训练输入中加入自然语言 key，使 hook 行为与 key 建立条件关联；
5. **DETECT**：黑盒下对真实 key、sham key 成对查询，比较频率差并用 ROC/AUC 或配对 t 检验评估归属。

## 6. 关键技术细节

- MATH 可插入依赖/输入验证，SimpleQA 可插入联网检查或强制访问页面，SWE-Smith 可插入工作区检查或创建后验证；设计原则是额外动作应可执行且不删除原动作。
- 默认水印比例 \(R=0.05\)。MATH、SimpleQA 的 key 为 “It is an interesting question.”，SWE-Smith 为 “It is a thorny Issue.”；sham key 用于排除一般提示变化带来的行为差异。
- 轨迹 action 可训练、observation 被 mask 的事实决定了只改变 observation 不能形成可靠水印，hook action 才是实际监督信号。
- 比 token 命中率更可靠的统计量是 key/control 的差分；否则某个 hook 本身在自然任务中偶发，也会造成误判。

## 7. 实验设置

- 数据：MATH（数学推理）、SimpleQA（网页检索）、SWE-Smith（真实 GitHub issue 的软件工程）；分别抽取 1,000、2,000、2,000 条正确轨迹。
- 脚手架与评测：前两者使用 Smolagents/Smolagents-Benchmark-v1，SWE-Smith 使用 SWE-Agent/SWE-Bench Lite。
- 基线：CodeMark；并在可适用的 Python 场景适配 AutoPoison、DeadCode、StyleTransfer。
- 训练/生成：辅助 LLM 为 Qwen-3-Coder-30B-A3B；检测实验主用 Qwen-2.5-Coder-7B，亦考察 3B、14B 和 Llama-3.1-8B。
- 指标：检测 AUC、不同 FPR 下 TPR、\(\widehat{\Delta q}\)、配对 t-score，及 Pass@1、平均轮数、输出 token 数。

## 8. 实验结果与分析

- 在 Qwen-2.5-Coder-7B、单 prompt、每 prompt 8 次查询下，standalone 与 contextual 的三数据集平均 AUC 分别为 97.8 与 90.8，CodeMark 为 55.5；摘要汇总的平均检测 AUC 为 94.3。
- 具体地，MATH/SimpleQA 的 hook 信号很强；SWE-Smith 更复杂、可学习容量更有限，但 contextual hook 因与长轨迹上下文融合自然，在该场景可优于 standalone。
- 随 prompt 数增加，真实 key 与 sham key 的配对 t-score 上升，说明证据可累积；CodeMark 的差值始终接近零。
- \(R=0.04\) 时各 ACTHOOK 变体已能取得超过 80 的 AUC；standalone 在极低比例上更易学习，contextual 需要略高比例换取自然性。
- 表中 Pass@1、平均轮数和 token 数显示水印对已微调 agent 的任务性能影响很小；对弱基座模型的绝对性能解释须注意其本就可能格式失败、提前终止。

## 9. 优点与局限

优点：准确针对轨迹的 action/observation 异构结构；黑盒可验证；低比例水印也可学习；以真实 key 对 sham key 的差分降低自然 hook 偶发造成的误报；hook 可按领域定制。

局限：需要人为/辅助 LLM 设计不影响功能的 hook，跨工具环境的泛化不自动成立；攻击者可进行强过滤、再训练、拒绝执行冗余动作或检测并删除触发句；激活 key 若泄露，可能被用于规避或反向探测；检测需要多次查询和有代表性的任务集合，不能把单次触发当作可靠归因。

## 10. 可复现性与使用建议

复现应保存原轨迹、CHECK 规则、插入点、key/sham key、随机种子和每条轨迹的水印标签。必须执行 hook 后的任务回归测试，不能仅检查文本流畅性。部署时用多组任务与多次采样做统计检验，预先固定显著性门槛、FPR、查询预算；同时轮换 key 并限制其在公开数据中的暴露。

## 11. 论文写作逻辑分析

文章从“高价值轨迹不可追踪”出发，指出 token/代码水印在小型、异构轨迹数据中难学，继而以高熵动作边界提出行为 hook。方法将可插入性、条件触发和黑盒检测拆成 CHECK/INJECT/DETECT；实验依次论证检测优势、任务保持、统计显著性、模型规模和水印比例。其论证最严谨之处是使用真实 key 与 sham key 的对照，而不是只观察某个动作是否出现。
