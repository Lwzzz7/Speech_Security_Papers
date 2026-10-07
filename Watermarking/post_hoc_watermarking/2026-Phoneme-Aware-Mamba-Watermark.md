# Phoneme-Aware Mamba Watermark: An Active Defense System Against Purified Speech Deepfakes

> 精读依据：ISCA会议PDF全文与会议信息，核验日期2026-10-05。论文同时讨论净化与说话人适配克隆；两条实验链的统计与攻击顺序需分别理解。原文若有单位或配置冲突，以下保留并指出。

## 0. 摘要翻译

随着文本到语音与声音转换快速发展，伪造语音已达到高度逼真的水平，对声纹生物识别造成严重威胁。传统被动检测对未见合成模型的泛化有限。本文提出基于Mamba的音素感知主动水印防御，通过双列双向状态空间模型捕获长程依赖，在语音中隐蔽嵌入身份比特串。为抵抗PhonePuRe等扩散净化，作者引入音素指导嵌入，使水印与语音语义紧密耦合，旨在说话人适配时转移到伪造语音。实验报告在较强净化下有效溯源、约1秒检测长度与较高TPR。原文给出了源代码链接；本次该链接返回404，不能确认当前公开可复现。

## 1. 研究动机

被动深伪检测依赖合成伪影，面对新模型可能失效。主动水印先在受保护语音中注入身份，希望攻击者用这些数据学习说话人时把水印一起学入；但净化模型可能把细微扰动清除。

作者提出两项策略：用Mamba建模语音的长时序结构，用音素稳定区域限制嵌入，使标记不只是表面噪声。还通过PhonePuRe对抗微调适应净化。这里“音素/语义耦合”是机制主张，不能等同水印已成为不可删除的语言语义。

## 2. 威胁模型

| 维度 | 论文设定 |
|---|---|
| 受保护者 | 发布带身份水印的语音，用于追踪后续伪造来源 |
| 攻击者 | 获取该语音；可用PhonePuRe净化；可用水印数据对YourTTS、sv2TTS做说话人适配 |
| 攻击目标 | 生成目标声线并去掉可追踪身份 |
| 防御者能力 | 可训练波形水印编码器/解码器，持有XLSR、音素对齐及净化模块，可查询已知净化器用于微调 |
| 消息 | 说话人身份比特串，分段重复/纠错/多数投票 |
| 输入条件 | 16kHz语音；训练音素边界与字典来自MFA/语料 |
| 质量约束 | 低频谱扰动、重建/扰动损失，SNR及PESQ检查 |
| 主要证据 | 直接水印净化的比特准确率；VCTK适配克隆的TPR与检测长度 |

论文形式化要求为：

$$D(P(G(E(x,w))))=w$$

$E$嵌入、$G$克隆、$P$净化、$D$检测。该表达顺序是克隆后净化，但训练/实验也讨论直接净化水印语音及用水印样本适配。当前材料没有把所有链条组成同一完整 factorial 验证，不能把各实验数字拼成任意净化顺序都成功。

**现实性与边界：**说话人适配训练是目标水印转移的具体场景，不能直接当作源音频经VC或ASR-TTS清洗的鲁棒性。正文提到零样本克隆背景，但实测使用FINETUNING，不应将TPR写成零样本参考音频克隆结果。未知净化器、攻击者联合优化、真实录放与低码率codec没有充分验证。

## 3. 方法设计与复现级解读

### 3.1 全局流程

离线：MFA获得音素边界 → 构建逐音素平均频谱字典/稳定掩码。训练：音频进入冻结XLSR与STFT → 身份消息条件化 → 时序编码器预测频谱扰动 → 音素掩码/低频限制 → iSTFT合成水印波形 → CNN解码 → 多损失更新；第二阶段加入PhonePuRe样本微调。部署：给身份语音嵌入；对可疑音频按1秒分段 → CNN提取码字 → 纠错/相似度阈值 → 多段投票。

### 3.2 冻结语音特征与频谱扰动

输入 $x\in\mathbb{R}^{T}$，身份 $w\in\{0,1\}^{K}$。冻结XLSR-300M输出 $H\in\mathbb{R}^{B\times T'\times1024}$，16kHz下 $T'\approx\lfloor T/320\rfloor$。同时对输入做复数STFT。编码网络结合消息/说话人条件与时序特征，预测频谱扰动 $\Delta$。

谱扰动限制在100–1000Hz，使用nFFT512、hop128、Hann窗；论文实验称覆盖29个频率bin。结合掩码后：

$$\Delta_{spec}=\alpha_{wm}(\Delta\odot M)$$

$M$控制可嵌入时频位置，$\alpha_{wm}$控制强度。扰动与原谱合成后iSTFT产生 $\tilde x$。复数谱/幅度谱处理、相位沿用或修改、XLSR时间步到STFT帧的上采样接口、消息条件拼接维度，正文没有全部明示。这些必须标为待实现核对，而不是凭经验补造。

### 3.3 双向与多头Mamba

比较三种时序编码器：BiLSTM基线；双列BiMamba沿正向与反向扫描后拼接；MH-Mamba使用三个并行选择性状态空间头、门控融合与四层堆叠。双向结构帮助编码器利用前后音素上下文，多头意在建模不同动态。

音素表示参与门控，原文概括为：

$$g_t=\sigma(W_p P_t),\quad h_t=g_t\odot SSM(h_{t-1},x_t)$$

$P_t$为对应音素表示，$g_t$调节时序状态。此式说明条件路径，但不等于完整Mamba离散化实现；状态维度、head维度、选择性步长与双向融合细节不充分。XLSR冻结，水印编码器/解码器可训练，两者不共享参数；正文又提“共享front-end extractor”，需区分共享冻结特征与共享可训练解码网络，材料描述未完全统一。

### 3.4 音素稳定区域掩码

MFA以约10ms边界给出ARPAbet音素段，预先统计每音素平均谱/稳定性。掩码倾向于音素内部稳定低方差区域，降低过渡处嵌入。目标是把标记放到合成/净化更倾向保留的语音结构中。

正文关于“stable≤1.5、unstable≥0.2”的描述存在范围重叠，没有完整无歧义的掩码计算公式。复现需确认方差/能量标准化、阈值单位、是否软掩码及如何处理边界。不能擅自调整成不重叠阈值后当作原方法。

若嵌入需要MFA与转录，部署时是否先ASR对齐、离线预计算或使用预测音素，都影响系统可用性；正文未充分交代该接口。检测CNN未描述必须重新做MFA。

### 3.5 解码、纠错与分段验证

解码器从低频log-magnitude区域读取标记，使用8层CNN；实验描述通道64→128→256、BN、LeakyReLU、MaxPool2d、adaptive pooling和256→36线性输出。初始训练使用 $t=(2w-1)\times5$ 作为目标，说明网络输出更像实数logit/回归值，不可直接当成概率；bit阈值还需核对。

原文声称Hamming(6,3)把16位身份编码成36位。码率1/2下16位直接对应32位；若先补齐到18位才得到36位，但补位/分块映射原文未说明。这里保留该矛盾，不替作者补造编码规则。

每1秒片段嵌同一码字，检测以Hamming相似度阈值.7并对多段多数投票。重复提高短段证据量，但负样本/多身份误匹配必须单独测。原文定义UFL为undetected forgery length，并称理论上限2秒；它不是GPU处理时间。UFL的表头为秒且带向上箭头，部分正文却以百分比描述，以下按表头标记并指出单位/方向问题。

### 3.6 损失与两阶段训练

初始目标：

$$L_{enc}=\|\Delta_{spec}\|_2+MSE(\hat w,t)+0.1MSE(\tilde x,x)$$

扰动项限制强度，消息MSE把解码输出推向正负目标，波形MSE控制重建。一次训练为取音频/身份→特征/谱→编码扰动→合成→解码→算损失→只更新编码/解码器，XLSR冻结。

对抗微调加入概率.3的PhonePuRe（RevDiffWave+spectral refiner）。总式为：

$$L=\lambda_{wm}L_{wm}+\lambda_\delta L_\delta+\lambda_{recon}L_{recon}+\lambda_{org}L_{org}+\lambda_{pur}L_{pur}$$

原文说明$L_{wm}$约束消息恢复，$L_\delta$约束扰动幅度，$L_{recon}$保持波形一致，$L_{org}$惩罚干净语音假阳性，$L_{pur}$鼓励净化后恢复。给出权重wm1、δ.1、org.3、pur.5，但未清楚给出所有项的精确数学实现及 $\lambda_{recon}$。净化路径写stop-gradient，而图1描述differentiable purification：若完全切断净化输出至嵌入器，净化损失可训练检测器却未必训练嵌入器抗净化；是否使用代理梯度未说明。复现不能自行假设STE或可微净化。

### 3.7 复现配置

| 项目 | 原文配置/缺口 | 来源 |
|---|---|---|
| 训练数据 | LibriSpeech train-clean-100，100h/253人；身份实验又称选100人 | 实验 |
| 克隆测试 | VCTK11人，每人15条，YourTTS/sv2TTS适配 | 实验 |
| 预处理 | 16kHz；512FFT/128hop/Hann；100–1000Hz | 方法/实验 |
| 特征 | 冻结XLSR-300M；1024维 | 方法 |
| 初始训练 | AdamW1e-4、cosine、batch16、200epoch | 实验 |
| 微调 | lr1e-5、batch12、50epoch、净化概率.3 | 实验 |
| 检查点 | 初始训练选验证bit accuracy；微调选purified accuracy | §4.2/4.3 |
| 检测 | 1s分段、36输出、Hamming相似度.7、多数投票 | 方法/实验 |
| 缺口 | 完整掩码、ECC补位、谱/特征对齐、损失定义、梯度路径、种子/算力/成本 | 当前原文未充分提供 |

## 4. 与其他方法对比

| 方法 | 核心思想 | 优点 | 局限 | 本文变化 |
|---|---|---|---|---|
| 被动ADD | 识别合成伪影 | 无需先保护原语音 | 新合成器域偏移 | 主动植入身份 |
| 对抗扰动防克隆 | 使攻击者难以模仿声线 | 可阻止部分克隆 | 净化可能删除扰动 | 标记转移后追踪 |
| Timbre/VoiceMark类 | 让音色相关标记进入克隆 | 来源可追踪 | 具体克隆/净化设定依赖 | 音素掩码+Mamba+净化微调 |
| 本文三编码器 | 同系统比较BiLSTM/BiMamba/MH | 分析时序结构 | 非完整跨方法共同预算比较 | 多头与双向结构 |

## 5. 实验表现与优势

### 5.1 直接净化的比特恢复

| 编码器/训练 | 原Bit Acc.(%) | 净化后Bit Acc.(%) | Retention(%) |
|---|---:|---:|---:|
| BiLSTM base | 77.07 | 48.26 | 62.62 |
| BiLSTM adv | 82.51 | 67.18 | 81.42 |
| BiMamba base | 84.77 | 75.53 | 89.10 |
| BiMamba adv | 89.68 | 80.25 | 89.48 |
| MH-Mamba base | 89.67 | 80.19 | 89.43 |
| MH-Mamba adv | 91.56 | 84.55 | 92.34 |

Retention是净化后bit accuracy除以原始accuracy，例如84.55/91.56≈92.34%，不是92.34%身份整串恢复。即使最强系统clean bit accuracy也未达到100%，需要纠错/投票才能得到后续身份判断。

### 5.2 克隆溯源与检测长度

表2报告BiLSTM/BiMamba/MH-Mamba TPR为99.9±.2、100±.1、99.9±.1%，UFL为1.2±.4、1.1±.2、1.0±.1s。测试11人×15条=165条时，单次原始utterance比例的步长约.606%，无法自然给出99.9%；若采用片段、重复运行或其他汇总，统计粒度应明确。原文未给足该信息，不能替其解释成完整165句全部成功。

缺少固定FPR或大规模非目标身份混淆检验，也没有每种克隆模型分项统计。因此高TPR不能单独建立低误归因安全性。UFL部分描述单位不一致，按表格可理解为短检测长度，不应当作处理延迟或实时计算耗时。

### 5.3 保真度

每编码器100对音频：BiLSTM SNR34.12±4.23dB/PESQ3.53±.41；BiMamba35.13±3.48/3.55±.39；MH35.67±4.54/3.54±.57。正文概括“全部超过35dB”与表中BiLSTM34.12不符，采用表格数字。质量支持扰动较轻，但无人工MOS或codec后统一质量评估。

### 5.4 消融、常规攻击与容量

表1对比架构与净化微调，显示MH与adv版本更高；音素指导的图示消融提供方向性支持，但当前缺少可精确提取的逐点表格，未补造数字。没有广泛DSP、低码率神经codec、VC/ASR-TTS的独立矩阵。16bit身份/36bit码字与1秒重复是固定配置，没有容量—质量—鲁棒性曲线。

## 6. 学习与应用

原文[代码地址](https://github.com/Silence-ai423/phoneme-aware-mamba-watermark)在2026-10-05 HTTP核验返回404。可能未公开、已移动或私有，当前无法判断；不能据论文一句available宣称仓库可复现。

最小实现应先固定BiLSTM与简单可定义掩码，准确完成clean解码、bit目标、纠错及误报测试，再加Mamba与净化。优先澄清stop-gradient路径、ECC映射和克隆顺序，避免架构训练完成后才发现协议不一致。研究迁移时把目标适配克隆、直接净化、源音频重合成三类链条分开，使用相同身份阈值与负例集。

## 7. 总结

将身份水印耦合音素稳定区。

## 8. 图表精读与证据链

系统图连接冻结语音特征、音素掩码、时序编码与低频解码；表1说明架构和净化微调对比特恢复的影响；音素消融图服务耦合机制；表2服务适配克隆溯源与短片段验证；表3提供SNR/PESQ质量。证据应分链读取：表1的Retention不是表2的TPR，表2的1s不是推理延迟。最缺的控制是共同FPR、多身份负例、明确统计单位和不同净化/克隆顺序。

## 9. 复现难度与适合人群

难度高。需要XLSR、MFA、Mamba实现、净化模型、TTS适配训练、纠错与分段判决；代码当前不可访问，关键公式/接口缺口较多。适合主动语音保护、目标参考克隆和净化对抗研究者。作为强benchmark直接比较前，应先核对其协议与自己的攻击链是否一致。

## 10. 简短全面总结

本文针对被动深伪检测泛化不足以及主动扰动易被扩散净化删除的问题，提出音素感知Mamba身份水印。编码器结合冻结XLSR特征、身份码字和音素稳定区域掩码，在100–1000Hz频谱内预测微弱扰动，使用双向或多头状态空间模型建模时序；CNN检测器按一秒片段提取码字，并通过纠错、Hamming相似度和多段投票判定来源。训练先兼顾消息恢复与波形质量，再加入PhonePuRe净化微调。LibriSpeech上MH-Mamba对抗训练后的净化bit accuracy为84.55%，相对原91.56%的保留比为92.34%；VCTK说话人适配克隆实验报告高TPR及约一秒检测长度，PESQ约3.54。贡献是音素结构与长程建模结合的主动追踪思路，但ECC映射、净化梯度和统计单位存在缺口，且未给固定FPR和广泛重合成攻击验证，代码链接当前不可访问。

## 11. 论文写作逻辑分析

引言按被动检测失效、主动扰动被净化、需要可转移身份三步推进，动机明确。音素耦合与Mamba分别回应结构绑定和时序依赖，净化微调补足已知强攻击；方法叙事有基本因果关系。

论证较弱处是从音素稳定区域直接提升到语义绑定的表述，以及不同实验链的数字被并列用于强安全结论。原文单位、ECC与质量概括的矛盾影响可信度；复现写作应主动列出具体链条、分母、负例、梯度路径和缺失项。可借鉴其把净化作为训练威胁的做法，但在自己的论文中必须用受控消融证明音素机制，而非仅比较不同复杂度时序网络。

## 12. 论文信息

- Authors: Yanda Shao, Mengke Zhang, Zhixin Lin, Tianyi Yang.
- Venue: Proc. Interspeech 2026, pp.6891–6895.
- DOI: [10.21437/Interspeech.2026-2105](https://doi.org/10.21437/Interspeech.2026-2105).
- [Paper](https://www.isca-archive.org/interspeech_2026/shao26_interspeech.html) | [PDF](https://www.isca-archive.org/interspeech_2026/shao26_interspeech.pdf).
- Code（原文链接，当前404）: [phoneme-aware-mamba-watermark](https://github.com/Silence-ai423/phoneme-aware-mamba-watermark).
- Citation: Shao, Y., Zhang, M., Lin, Z., & Yang, T. “Phoneme-Aware Mamba Watermark: An Active Defense System Against Purified Speech Deepfakes.” Proc. Interspeech 2026, 6891–6895. doi:10.21437/Interspeech.2026-2105.

## Overview 条目

```md
- [Phoneme-Aware Mamba Watermark: An Active Defense System Against Purified Speech Deepfakes](./Watermarking/post_hoc_watermarking/2026-Phoneme-Aware-Mamba-Watermark.md)  
  *Proc. Interspeech 2026, pp. 6891–6895, 2026*  
  Citation: Yanda Shao, Mengke Zhang, Zhixin Lin, Tianyi Yang. “Phoneme-Aware Mamba Watermark: An Active Defense System Against Purified Speech Deepfakes.” Proc. Interspeech 2026, pp. 6891–6895, 2026. doi:10.21437/Interspeech.2026-2105.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2026/shao26_interspeech.html) | [PDF](https://www.isca-archive.org/interspeech_2026/shao26_interspeech.pdf) | [DOI](https://doi.org/10.21437/Interspeech.2026-2105) | [Project/Code](https://github.com/Silence-ai423/phoneme-aware-mamba-watermark)  
  Code: 原文链接2026-10-05返回404
```
