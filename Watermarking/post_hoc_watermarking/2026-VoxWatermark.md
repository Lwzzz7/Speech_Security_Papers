# VoxWatermark: A Large-Scale Benchmark for Audio Watermark Detection under Perturbations

> 精读依据：arXiv:2606.15187v1 全文、ISCA会议信息与官方代码入口，核验日期2026-10-05。本文检测“是否存在水印”，不是解码未知水印payload，也不是普通真伪语音分类。

## 0. 摘要翻译

随着语音生成系统在开放环境中快速部署，为音频提供可验证来源归因与版权责任变得关键。现有研究缺少一个在真实分布变化下系统比较不同水印注入方法的统一基准。本文构建VoxWatermark，在多语言、多来源语料上统一应用10种水印方法，其中4种神经方法、6种传统方法，并加入no-box、black-box与white-box扰动以模拟真实录制和传输条件。基于该基准，作者提出AudioWMD，作为大规模、多方法、跨分布设置的检测基线。结果显示水印方法多样性和分布变化会影响检测稳定性，并验证了AudioWMD的有效性与扩展性。数据及代码公开。具体“有效性”需结合正文中较弱的OOD固定阈值表现理解。

## 1. 研究动机

用某个水印方法自己的解码器测试，只回答该方法能否恢复自家消息；开放平台可能不知道音频经过哪种嵌入，因此需要跨方法的存在性检测。训练方法、语言、说话人和来源变化都会改变数据分布，原有单方法评估无法说明这类检测器的真实泛化。

本文同时提供统一数据生成/攻击基准和一个检测基线。AudioWMD认为，单次预测易受分布变化影响，观察轻微变换后的分数稳定性可能补充来源线索。其重点不是设计新的嵌入水印。

## 2. 威胁模型

| 访问级别 | 攻击能力与目标 |
|---|---|
| No-box | 无模型访问，对波形加噪、压缩、时间伸缩、滤波等；考察自然失真与非定向清洗 |
| Black-box | 查询目标水印检测/解码器，用HSJA或Square构造移除样本；不读取参数/梯度 |
| White-box | 读取解码器和目标消息，梯度优化移除或伪造；传统方法使用可微近似 |
| 新通用检测器 | 读取输入音频，学习多方法水印存在性；不知道/不恢复所有原消息 |

参与方包括语料提供者、10类水印嵌入器、攻击生成器、原方法解码器、通用AudioWMD。攻击主要针对原水印方法的检测接口，随后通用检测器评价这些样本；这不等于攻击者直接对AudioWMD做自适应攻击。

攻击约束希望保留内容并兼顾ViSQOL/SNR，但完整查询预算、白盒权重与优化超参未在正文统一披露。AudioWMD训练用部分已见方法，测试跨语言/来源/未见方法，部署阈值固定于验证集，不能在测试域重新调参。

现实性分析：开放来源与方法变化是合理场景；测试同时改变来源与方法时无法单独归因每一种变化。对未知语音整体判“有水印”也不能直接成为版权归属证据，仍需明确payload或密钥验证。

## 3. 方法设计与复现级解读

### 3.1 数据与样本层级（原文 §2）

基础原音频60,000条，每条5s，16kHz mono：LibriSpeech20k、CommonVoice20k、VCTK10k、AISHELL-1 10k；约83.4h。CommonVoice提供多语言覆盖，共25语言。每条原音频经过多嵌入、多参数、多攻击生成大量派生版本。

作者报告总规模91,090K样本和126,513.89h。这是派生变体总量，不能当成独立录音/独立说话人数据量。划分应按原始音频身份隔离，同源片段及其多个水印版本不能跨训练测试泄露。论文没有完整描述所有变体实际落盘与在线生成策略。

10种方法：LSB、QIM、Patchwork、Echo Hiding、Phase Coding、DSSS，以及AudioSeal、WavMark、Timbre、Perth。大部分显式消息16bit，Timbre10bit，Perth为隐式标记。统一标注不代表方法容量、嵌入强度与可听性完全相同。

### 3.2 扰动生成接口

No-box包含17类型与参数网格：time stretch .7–1.5、Gaussian5–40dB、背景5–35dB、echo .1–.9s、滤波、4–64bit量化、codec/Opus码率、动态范围、相位/抖动等。背景包含M109、F16、Babble等。参数的单位和变换实现应逐类核对，不能将所有范围当成同一物理强度。

Black-box：HSJA在波形与频谱两域，Square在频谱域，针对每方法200个水印样本。目标是原方法的水印移除，频谱攻击还需重建波形，正文没有充分统一相位处理和查询上限。

White-box：冻结解码器，移除目标采用翻转消息的BCE，伪造采用目标合法消息；用Adam在质量与目标间平衡，每方法200个样本。对传统算法的可微近似会改变梯度路径，需区别近似成功与原实现判决成功。

### 3.3 基础检测器

音频16kHz → log-mel → CNN → binary logit。图1依次展示两个含两次3×3卷积/BN/ReLU与MaxPool的块，再卷积与adaptive average pooling，得到单次水印分数。BCEWithLogits训练：

$$L_{base}=-y\log\sigma(l)-(1-y)\log(1-\sigma(l))$$

$y=1$是水印存在，$l$为logit，$\sigma$是sigmoid。该网络不接收payload，不输出消息位。图中未充分列出所有通道、mel帧数与STFT配置，不能照常见CNN自行补齐并称原文配置。

### 3.4 多次变换查询与稳定性统计（AudioWMD）

对同一输入做 $K=8$ 次内部查询：原音频加7个随机轻微变换，含stretch、pitch、noise、gain、masking。每次运行已训练基础网络，得到 $p_1,\ldots,p_K$。这里query是检测器内部重复前向，不是攻击者对外部服务的查询预算。

统计五维向量：均值、标准差、最大最小范围、超过阈值的占比、决策翻转比率。直观上均值提供总体证据，标准差/范围反映变换敏感性，占比/翻转率反映二值稳定性。把统计量送入logistic元分类器：

$$p_{meta}=\sigma(w^\top f+b)$$

基础CNN先学单次判别；元分类器再学这些分数的稳定性与标签关系。没有把全部变换与元分类器当成可微端到端联合训练。占比阈值（flip相对原始查询）、变换随机分布与元训练数据组织，应按代码确认，正文未完全展开。

部署输出一个存在性概率，使用验证集选定阈值，不在OOD测试重新校准。重复推理成本约随K增加，统计模型很轻但不能把整体称为单次低成本推理。

### 3.5 训练切分与适用边界

从34k原始训练池生成61,200样本，正负各30,600，分层留10%验证。训练包含LSB、QIM、DSSS、AudioSeal、Timbre、Phase Coding六种已见方法，没有扰动增强训练。测试1使用CommonVoice并排除英语/汉语；测试2用VCTK及未见Patchwork、Echo、WavMark。

训练与OOD测试同时改变多维条件，方法变化和语料变化不是完整因子化对照。Perth在统一数据生成中出现，但其在具体已见/未见检测划分的作用需核对，不能自动把所有10方法平均算进每张检测表。

### 3.6 复现配置与信息缺口

| 项目 | 原文配置 | 来源 |
|---|---|---|
| 原音频 | 60k，5s，16kHz mono，4语料，25语言 | §2 |
| 注入 | 10方法；16bit为主，Timbre10bit，Perth隐式 | §2 |
| 构造攻击 | 17 no-box；HSJA waveform/spec、Square spec；白盒BCE优化 | §2 |
| 攻击评测量 | 黑/白盒每方法200条 | 实验 |
| 检测训练 | 61,200、类别平衡、10%验证、6方法、无扰动训练 | §3 |
| AudioWMD | CNN log-mel；8查询；5统计；logistic元模型 | §3、图1 |
| 阈值 | 验证集固定，测试不重校准 | §3 |
| 缺失配置 | 完整STFT/CNN通道、训练lr/batch/epoch、元数据、攻击预算/权重、种子与成本 | 当前正文未充分提供 |

## 4. 与其他方法对比

| 方法类别 | 核心思想 | 优点 | 局限 | 本文变化 |
|---|---|---|---|---|
| 原方法解码器 | 已知算法/密钥恢复payload | 能直接验证特定归属 | 无法通用覆盖陌生方法 | 本文做通用存在性检测 |
| WMD基线 | 单次ConvNeXtV2及非对称损失 | 简单直接 | OOD分数漂移 | 多查询稳定性统计 |
| AudioWMD | CNN+变换统计+元分类器 | 加入预测稳定性 | 多次前向、阈值仍受域影响 | 与统一扰动基准配套 |
| VoxWatermark | 多方法多域统一生成/攻击 | 可比较攻击面 | 派生规模不等于独立多样性 | 系统化数据与协议 |

WMD与AudioWMD基础架构和损失也不同，因此性能差不能仅归功于稳定性聚合；需要同backbone对照才能隔离贡献。

## 5. 实验表现与优势

### 5.1 原始与OOD检测

| 设置 | WMD AUROC / Accuracy(%) | AudioWMD AUROC / Accuracy(%) |
|---|---:|---:|
| 验证 | 72 / 67 | 88.3 / 84 |
| Test1 | 57.1 / 55 | 63.8 / 53 |
| Test2 | 57.9 / 56 | 63.2 / 58 |

表4显示AUROC提升，但Test1固定阈值accuracy反而由55降到53；AudioWMD水印类recall在Test1仅9%、Test2为31%。AUROC是分数排序，不等于当前阈值可用。其贡献应表述为一定排序改善，而非成熟的通用检测能力。

### 5.2 扰动与白盒样本

表5中Test1 white-box AUROC由WMD48.63提升至AudioWMD77.15%，F1由45到53；Test2 white-box由41.18到70.02%，F1由36到57。No-box提升小得多：Test1 AUROC51.28→54.33，Test2 53.11→53.15。白盒样本上的提升明显，但no-box总体接近随机排序。

### 5.3 黑盒样本

表6的结果不支持统一优势。Test1 HSJA频谱样本上WMD TPR96.09%，AudioWMD3.91%；Square上为40.62%与77.34%。Test2 HSJA频谱为100%与2.5%，Square为84.15%与73.17%。总TPR在Test1为73.96→50.26%，Test2为93.42→51.44%。这揭示检测器依赖攻击类型，不能只给最有利白盒AUROC。

### 5.4 质量、容量与消融

no-box示例M109噪声ViSQOL3.188、SNR10；stretch.9为3.587、SNR-1.55；EnCodec1.5kbps为3.311、SNR1.15。时间变形后直接SNR很低可能包含对齐误差，不能简单当作严重可听噪声。基准提供攻击质量描述，并非本文新嵌入器的保真度优势。

不同方法容量不完全一致，通用检测器不解码payload，因此无统一bps恢复能力。未见完整K、统计特征、元模型、基础backbone的独立消融；缺少直接适应AudioWMD的攻击和误归因安全协议。

## 6. 学习与应用

[官方代码](https://github.com/wailywang/VoxWatermark)已核实对应论文。可先从小量原音频生成多方法版本，按源文件隔离切分，再复现单次CNN与相同CNN加统计的受控比较。应同时看AUROC、固定阈值TPR/FPR和类别recall；高AUROC不足以说明部署可用。

对个人水印研究，该基准更适合作为攻击参数和分布变化设计参考。检测“任意水印存在”与检测“自己的密钥/消息”应分开评估，不能用前者替代版权证明。91,090K规模的复现应先确认生成策略和存储成本，不必为了复现核心结论先构造全部变体。

## 7. 总结

用多变换稳定性检测水印。

## 8. 图表精读与证据链

数据流程图与攻击表定义统一基准；图1说明基础CNN与重复查询的接口；表4检验OOD并暴露阈值问题；表5显示部分白盒分数排序改善；表6给出黑盒失效类型。最强贡献是方法/语言/攻击统一组织，最弱外推是“robust baseline”的实际部署性能。实验应将方法域、语言域、来源域分开控制，避免泛化成因混淆。

## 9. 复现难度与适合人群

难度高（完整基准），中（小规模检测基线）。依赖多语料、10种水印实现、攻击库、质量工具及较大存储；公开代码有助于启动。最小版本可固定两个方法、一源训练一源测试、8查询，并准确校准固定阈值。适合通用水印检测、基准设计与跨域鲁棒性研究者。

## 10. 简短全面总结

VoxWatermark针对开放环境中水印方法与语音来源变化带来的通用检测困难，建立多语言、多方法和多攻击统一基准。它以六万条五秒原音频为基础，使用四种神经和六种传统水印生成派生样本，再加入no-box处理、HSJA/Square黑盒移除及白盒梯度攻击。AudioWMD先用log-mel CNN做存在性分类，对原音频和七次随机轻微变换重复预测，将均值、标准差、范围、正判占比及翻转率送入logistic元分类器。验证集AUROC达88.3%，OOD排序比WMD有所改善，部分白盒样本提升明显；但固定阈值下未知域水印recall很低，no-box接近随机，HSJA频谱攻击也出现显著失效。因此其主要价值是系统化评测和分数稳定性思路，不是可靠的未知水印版权归因。限制包括多维分布变化混杂、未独立隔离聚合贡献，以及缺少直接针对新检测器的自适应攻击。

## 11. 论文写作逻辑分析

引言把开放环境归因需求收缩成多方法与分布偏移缺口，基准和检测器两项贡献顺势出现。数据先于方法能让读者知道算法面对什么变化。实验分别覆盖clean/no-box/white/black，具有完整攻击类型框架。

问题在于摘要的鲁棒性表述与多张表中的低recall、接近随机AUROC及黑盒退化之间存在张力。写作应主动区分排序指标与部署阈值，并解释反例。可借鉴统一协议与多访问级别组织；避免用总派生小时数替代独立数据多样性论证，并为每个泛化主张提供单变量对照。

## 12. 论文信息

- Authors: Farnaz Sedaghati, Yuxi Wang, Zicheng Weng, Wei Rao.
- Venue: Proc. Interspeech 2026, pp.6886–6890.
- DOI: [10.21437/Interspeech.2026-1771](https://doi.org/10.21437/Interspeech.2026-1771).
- [Paper](https://www.isca-archive.org/interspeech_2026/sedaghati26_interspeech.html) | [PDF](https://arxiv.org/pdf/2606.15187) | [arXiv](https://arxiv.org/abs/2606.15187) | [Code & Data entry](https://github.com/wailywang/VoxWatermark).
- Citation: Sedaghati, F., Wang, Y., Weng, Z., & Rao, W. “VoxWatermark: A Large-Scale Benchmark for Audio Watermark Detection under Perturbations.” Proc. Interspeech 2026, 6886–6890. doi:10.21437/Interspeech.2026-1771.

## Overview 条目

```md
- [VoxWatermark: A Large-Scale Benchmark for Audio Watermark Detection under Perturbations](./Watermarking/attacks_and_benchmarks/2026-VoxWatermark.md)  
  *Proc. Interspeech 2026, pp. 6886–6890, 2026*  
  Citation: Farnaz Sedaghati, Yuxi Wang, Zicheng Weng, Wei Rao. “VoxWatermark: A Large-Scale Benchmark for Audio Watermark Detection under Perturbations.” Proc. Interspeech 2026, pp. 6886–6890, 2026. doi:10.21437/Interspeech.2026-1771.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2026/sedaghati26_interspeech.html) | [PDF](https://arxiv.org/pdf/2606.15187) | [DOI](https://doi.org/10.21437/Interspeech.2026-1771) | [arXiv](https://arxiv.org/abs/2606.15187) | [Project/Code](https://github.com/wailywang/VoxWatermark)  
  Code: 已验证官方仓库
```
