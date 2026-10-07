# DuraMark: Duration-Embedded Watermarking in LLM-based TTS

> 精读依据：arXiv:2606.15264v1 全文与 ISCA 会议信息，核验日期2026-10-05。论文以普通话音节时长为载体；以下不将其直接外推为任意语言、任意重合成鲁棒水印。

## 0. 摘要翻译

基于大语言模型的文本到语音系统取得了显著的声音克隆能力，同时引发潜在深伪滥用的担忧。语音水印通过向生成语音嵌入可追踪信息缓解这一问题。主流水印在波形或频谱等信号层面操作，使标记容易被神经codec、声码器等生成式攻击破坏。为此，本文提出DuraMark，一种利用音节时长编辑嵌入水印的鲁棒信息层水印框架。它结合时长可控的LLM-TTS，在合成时编辑音节时长，并用时长提取器读出这些时长进行检测。实验表明DuraMark对生成式攻击具有更强鲁棒性，显著优于信号层基线。论文提供了音频演示链接。

## 1. 研究动机

波形残差可能被神经重建当成无关噪声。时长属于音节级语音结构：codec和声码器通常重建频谱细节而尽量保持节奏。因此作者选择让水印进入音节时长，而不是发布后附加残差。

挑战有两个：LLM生成语音token未必能精确控制每个音节时长；即使token数量被控制，后面的flow声学模型也可能改变实际边界。本文用显式duration query和时长指导损失分别解决这两个接口问题。该设计需要修改与训练TTS流程，不能直接用于任意既有音频。

## 2. 威胁模型

| 维度 | 设定 |
|---|---|
| 参与方 | 可修改CosyVoice生成流程的水印发布者；外部音频后处理者；具备时长提取器的检测方 |
| 防御目标 | 发布音频经过重建/处理后仍能验证目标水印 |
| 攻击能力 | 神经codec、声码器、语音增强、传统压缩/噪声/滤波；主实验非优化型攻击 |
| 嵌入控制 | 逐音节duration token选择；后续语音token和flow生成 |
| 检测知识 | Info模式已知文本；Blind模式用ASR获取文本；两者都知道目标比特串 |
| 攻击约束 | 保持语音可用；各处理使用表2参数，未统一硬质量预算 |
| 系统边界 | text→duration/speech tokens→mel→waveform；检测waveform→mel/文本→duration→相关分数 |

普通话一字一音节使对齐和载体索引相对直接。换成英语等语言需明确音节/音素切分和token映射。所谓Blind不等于无密钥任意消息恢复；正文主要证明目标序列的存在性检验。ASR错误会同时改变音节顺序、数量和水印索引。

现实性分析：常见codec与声码器保留时序时，这种载体合理。但时长缩放、局部插删、VC、ASR-TTS重新生成或知晓规则后重新定时均可能打击载体，不能由表2自动推断安全。

## 3. 方法设计与复现级解读

### 3.1 全局执行顺序

离线准备：MFA对齐语音与文本 → 获得逐音节帧时长和逐帧归属 → 训练时长提取器。模型训练：训练显式时长LLM → 冻结时长提取器，训练具有时长指导的flow声学模型。嵌入推理：逐音节预测duration → 根据水印位调整奇偶 → 按调整后时长生成语音token → flow生成mel →声码器出波形。检测：给定文本或ASR转录 → 提取连续音节时长 → 与目标比特相关 → 阈值判断。

### 3.2 显式时长LLM（原文 §2）

输入包括文本音节表示、位置信息和说话人条件。每个音节插入duration query，预测离散时长 $d_i$；随后生成对应数量的语音token。duration码本和speech token码本分离，避免将“何时结束音节”完全隐含在语音token序列中。

训练监督来自对齐得到的真实duration以及真实speech tokens。总LLM目标为duration交叉熵与speech token交叉熵的加权组合，duration/token项采用各自长度归一化。用$CE$表示对应分类交叉熵：

$$L_{llm}=w_{llm}I^{-1}\sum_i CE(\tilde y_i,y_i)+(1-w_{llm})(\sum_i d_i)^{-1}\sum_i\sum_{t=1}^{d_i}CE(\tilde z_{i,t},z_{i,t})$$

其中$y_i$是duration分布，$z_{i,t}$是当前音节内speech token分布，带波浪符号为真实标签分布，$0\le w_{llm}\le1$。duration分支的错误会影响后续token数量，因此不能只训练独立时长预测器再无条件拼接语音。

论文给出模型关系，但未把所有token时间分辨率、词表、duration上限及LLM结构配置完整展开，复现需沿CosyVoice实现核对。不要把duration帧直接当成毫秒；必须明确该模型每token对应时间尺度。

### 3.3 时长提取器与对齐监督

提取器读取文本与mel，各自编码后通过Transformer建模跨模态对应。输出 $a_{i,t}$ 表示第 $t$ 个声学帧归属于第 $i$ 个音节的分数/概率，连续时长估计为：

$$\hat d_i=\sum_t a_{i,t}$$

使用MFA逐帧音节标签训练交叉熵。连续求和使时长估计可用于指导flow模型；它并非把硬argmax帧数直接送入一个不可导损失。提取器先训练，再冻结，其参数不随flow训练更新，但梯度仍可从其输入mel回传flow。

需要复现文本音节序列、MFA标签到mel帧的映射、padding mask和概率归一化轴。ASR转录若与音频不对齐，提取器也不是错误修正器。

### 3.4 时长指导flow matching

对目标mel $x_1$ 与高斯起点 $x_0$，使用线性插值：

$$\psi_t=(1-t)x_0+t x_1$$

速度网络预测 $v_\theta(\psi_t,t|g,S,d)$，条件包括说话人 $g$、speech tokens $S$ 和显式duration $d$，CFM目标是 $x_1-x_0$。据当前状态与速度估计最终mel：

$$\hat x_1=\psi_t+(1-t)v_\theta(\psi_t,t|g,S,d)$$

把 $\hat x_1$ 送入冻结时长提取器，使用真实帧对齐监督得到 $L_{guide}$。总目标：

$$L_{flow}=w_{flow}L_{CFM}+(1-w_{flow})L_{guide}$$

该项把duration条件变成实际声学边界约束，减少声学生成“收到duration却不执行”的问题。实验另外给出 $\lambda_{llm}=1,\lambda_{flow}=4$，与方法中的混合权重符号不能未经核对当成完全同一参数。论文未充分说明所有权重之间的映射。

一次训练为采样配对文本/音频→采样 $x_0,t$→构造 $\psi_t$→预测速度→计算CFM与提取器指导→反传仅更新flow。LLM训练与该阶段分开，不能把所有损失假设成端到端同时更新。

### 3.5 嵌入：最小duration改动

目标比特 $w_i\in\{0,1\}$ 与duration奇偶对应。先取预测概率最大duration，若奇偶一致直接使用；若不一致，比较邻近 $d_i-1,d_i+1$ 的预测概率，选择概率较高且合法的候选。随后按照改动后的duration生成语音token，历史上下文也使用改动值。

$$d_i\bmod2=w_i$$

改动最多一个duration单位，是减少节奏变化的设计。边界候选、零/最大duration处理、过短音节约束、比特串与文本长度不一致的处理，原文没有完全展开。每音节一位是载体设计，不等于已证明可可靠恢复同样数量的独立消息位。

### 3.6 检测：连续奇偶相关

把连续时长估计映射成平滑奇偶响应：

$$q_i=-\cos(\pi\hat d_i),\quad b_i=2w_i-1$$

$$T=I^{-1}\sum_{i=1}^{I}q_i b_i$$

整数偶时 $q=-1$，奇时 $q=1$，与目标比特符号相乘后求平均。$T$ 大于阈值表示目标水印存在。与把每个连续duration四舍五入再解码相比，余弦保留估计偏差的软信息。

Info提供文本；Blind由ASR提供。两者都要确定载体音节顺序与目标比特串。实验阈值按FPR=1%设定，检测是序列相关的统计判断，而非所有位必须完全恢复。

### 3.7 复现配置

| 项目 | 原文配置/缺口 | 来源 |
|---|---|---|
| 基础系统 | CosyVoice，普通话音节时长 | 方法 |
| 训练语料 | WenetSpeech 10kh资源；实际完整筛选规则未充分展开 | 实验 |
| 对齐 | MFA；用于duration及逐帧监督 | 方法 |
| 测试 | AISHELL-3，214说话人；每长度组1000对 | 实验、表1 |
| 优化 | Adam，学习率1e-5；8张MLU580 | 实验 |
| 采样 | speech top-p0.8、top-k25；duration greedy后按奇偶调整 | 实验 |
| 权重 | $\lambda_{llm}=1,\lambda_{flow}=4$；与方法混合权重关系待核对 | 实验 |
| 主检测 | 33–64音节；TPR@FPR1% | 表1/2 |
| 未披露 | batch、epoch、随机种子、完整架构/词表、duration单位、阈值校准细节 | 复现缺口 |

## 4. 与其他方法对比

| 方法 | 核心思想 | 优点 | 局限 | 本文改进 |
|---|---|---|---|---|
| AudioSeal | 后处理局部波形水印 | 不需控制TTS，支持定位 | 某些重建删除证据 | 换成时长结构载体 |
| Timbre | 频谱/音色水印 | 能覆盖特定克隆场景 | codec类型依赖 | 显式控制生成时序 |
| WavMark | 波形多比特嵌入 | 可处理既有音频 | 部分codec/声码器脆弱 | duration奇偶判决 |
| DuraMark | LLM时长+flow指导 | 重建通常保留节奏 | 需修改训练TTS，依赖文本/对齐 | 让语音token与真实边界一致 |

## 5. 实验表现与优势

### 5.1 长度与检测口径

表1在17–32/33–64/65–100音节上，Info TPR为.981/.998/.998，Blind为.942/.987/.992，均在FPR1%。较长序列提供更多相关证据；短句Blind较弱。不能将这些TPR当作bit accuracy。

### 5.2 神经codec、声码器与增强

基线对同一未标记生成音频做后处理嵌入，主设置33–64音节。

| 处理 | Info / Blind TPR |
|---|---:|
| Clean | .998 / .987 |
| EnCodec 6kbps | .991 / .968 |
| DAC 3kbps | .994 / .980 |
| SpeechTokenizer 4kbps | .984 / .966 |
| FACodec 2.4kbps | .994 / .977 |
| BigVGAN | .997 / .987 |
| Vocos | .997 / .979 |
| HiFi-GAN | .994 / .985 |
| FRCRN | .999 / .989 |
| Demucs | .998 / .983 |

同表EnCodec6kbps下AudioSeal/Timbre/WavMark为.773/.124/.015，说明duration载体在该codec较稳定。Vocos下AudioSeal为.005、Timbre为1.000，表明重建不能用单一平均概括各基线。

### 5.3 常规攻击、保真度与容量

MP3 32kbps为.998/.983，Opus16kbps为.994/.983，Gaussian20dB为.994/.979，背景噪声20dB为.987/.965，低通4.8kHz为.973/.961。表2报告平均Info.993、Blind.978，对应AudioSeal.701、Timbre.790、WavMark.403。该平均只适用于论文攻击清单。

表3：未水印CER8.73%、MOS4.05±.09；DuraMark CER8.54%、MOS4.04±.07。MOS由11名普通话母语者评价20句，CER用Whisper；支持接近基线，不能因CER略低声称稳定改善识别。

载体约每音节1位，但论文主评估是给定序列检验，没有完整多用户消息恢复、bps、纠错容量与误归因实验。表2未展示时间缩放、裁剪、ASR-TTS及VC大规模攻击。

### 5.4 消融

完整Info/Blind为.998/.987；去掉duration输入为.455/.473；去掉指导损失为.327/.342。说明显式条件和声学时长落实都重要。消融比较的是完整结构需求，不能据此分配精确的独立贡献百分比。

## 6. 学习与应用

[官方演示](https://muzw.github.io/duramark_demo/)HTTP200，当前未确认完整官方训练代码。最小验证应先在现有duration可控TTS上改奇偶并用真实对齐检测，以隔离载体本身；再加入学习提取器和ASR，最后训练flow指导。不要一开始把ASR、对齐和生成误差混在一起。

对生成式水印研究的启发是把标记放进会被下游保留的结构，并明确验证条件是否被声学模型执行。迁移其他语言需要音节索引与duration时间尺度适配；迁移其他LLM-TTS需要重新验证token时长与波形边界一致性。

## 7. 总结

用音节时长奇偶标记语音。

## 8. 图表精读与证据链

系统图支撑LLM显式duration、flow指导、检测提取器之间的关系；表1检验长度与文本可得性；表2是对重建/后处理鲁棒性的直接证据；表3检验质量；表4说明两项结构设计的必要性。证据闭环最强的是保持时间结构的处理，未覆盖主动改时长与重新生成文本的攻击。应把“information-level”作为载体定位，不写成所有语义重合成都保留。

## 9. 复现难度与适合人群

难度高：需要CosyVoice训练、MFA对齐、时长提取器、flow指导、语音生成数据与训练算力。最小奇偶载体验证可低成本启动，完整论文级复现依赖未公开训练细节。适合LLM-TTS与生成式水印研究者，不适合只能处理发布后波形的部署环境。

## 10. 简短全面总结

DuraMark面向LLM语音合成的来源验证，将水印由细微波形残差转移到音节时长奇偶。作者修改CosyVoice流程，让LLM逐音节显式预测duration并生成对应数量的speech tokens，再用冻结的文本—mel时长提取器监督flow声学生成，确保条件时长落实到实际音频。嵌入时仅在预测duration奇偶不符目标位时选择相邻高概率时长，减少节奏变化；检测用连续时长的余弦奇偶响应与目标序列相关判决，已知文本与ASR盲转录两种模式分别评价。在普通话数据上，33–64音节clean TPR达到.998/.987，多种codec、声码器和增强后保持较高结果，MOS接近未水印基线。消融显示显式duration和指导损失缺一都会显著下降。贡献是把标记绑定到重建通常保留的时序结构，限制在于需要修改并训练TTS、依赖转录与对齐，且没有证明对主动时长编辑、VC或ASR-TTS重生成仍安全。

## 11. 论文写作逻辑分析

引言把重建删除残差与时长相对稳定相连接，载体选择有直接理由。方法以“LLM能控制、flow能执行、检测能读出”逐层解决接口问题，消融正好回应前两项，结构清晰。质量表和长度表补充可用性与统计证据量，短文仍形成较完整闭环。

容易外推的是“结构/信息级”与“所有再生成鲁棒”的关系：实验主要保留时间轴，应在威胁模型明确这一范围。可借鉴其生成控制到输出验证的双约束设计，同时在自己的论文补充显式攻击时序载体的强基线、独立FPR校准和语言迁移。

## 12. 论文信息

- Authors: Zhenwei Mou, Weili Jiang, Liping Chen, Zhen-Hua Ling, Kong Aik Lee, Kai Gao, Boyu Zhao.
- Venue: Proc. Interspeech 2026, pp.6876–6880.
- DOI: [10.21437/Interspeech.2026-2298](https://doi.org/10.21437/Interspeech.2026-2298).
- [Paper](https://www.isca-archive.org/interspeech_2026/mou26_interspeech.html) | [PDF](https://arxiv.org/pdf/2606.15264) | [arXiv](https://arxiv.org/abs/2606.15264) | [Demo](https://muzw.github.io/duramark_demo/).
- Code: 未确认完整官方代码；Demo不等于训练代码开源。
- Citation: Mou, Z., Jiang, W., Chen, L., Ling, Z.-H., Lee, K. A., Gao, K., & Zhao, B. “DuraMark: Duration-Embedded Watermarking in LLM-based TTS.” Proc. Interspeech 2026, 6876–6880. doi:10.21437/Interspeech.2026-2298.

## Overview 条目

```md
- [DuraMark: Duration-Embedded Watermarking in LLM-based TTS](./Watermarking/generative_watermarking/2026-DuraMark.md)  
  *Proc. Interspeech 2026, pp. 6876–6880, 2026*  
  Citation: Zhenwei Mou, Weili Jiang, Liping Chen, Zhen-Hua Ling, Kong Aik Lee, Kai Gao, Boyu Zhao. “DuraMark: Duration-Embedded Watermarking in LLM-based TTS.” Proc. Interspeech 2026, pp. 6876–6880, 2026. doi:10.21437/Interspeech.2026-2298.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2026/mou26_interspeech.html) | [PDF](https://arxiv.org/pdf/2606.15264) | [DOI](https://doi.org/10.21437/Interspeech.2026-2298) | [arXiv](https://arxiv.org/abs/2606.15264) | [Project/Code](https://muzw.github.io/duramark_demo/)  
  Demo: 已核验；完整官方代码未确认
```
