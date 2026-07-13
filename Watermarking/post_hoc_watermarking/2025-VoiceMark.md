# VoiceMark：利用说话人专属潜变量抵抗零样本声音克隆

## 0. 翻译摘要原文

抗声音克隆水印用于追踪并阻止未经授权的克隆。现有方法依赖用水印语音训练克隆模型，能够追踪传统 VC，却无法处理零样本 VC：模型仅以音频 prompt 推理，不发生训练。VoiceMark 首次把水印嵌入说话人专属潜变量，使水印可以随零样本克隆过程进入合成语音；同时加入 VC 模拟增强和基于 VAD 的损失以提高抗失真能力。多个零样本 VC 模型上的实验显示，VoiceMark 合成后检测准确率超过 95%，显著高于现有方法约 50% 的随机水平。

## 1. 方法动机

AudioMarkNet、Timbre Watermarking 等依赖“训练模型会学习水印”。CosyVoice、F5-TTS、MaskGCT 只用几秒 prompt 即可克隆，水印没有机会写入模型参数；合成内容、时长、速度又与 prompt 完全不同，普通波形扰动会被生成过程过滤。

零样本 VC 必须从 prompt 提取并保留说话人信息。VoiceMark 因而不把水印分布在任意波形细节，而是与音色一起写进 RVQ 中排除内容层后的 speaker-specific latents。只要 VC 依赖这些属性生成目标音色，水印就有机会跨内容迁移。

## 2. 威胁模型解读

- 权利人给可被拿作 prompt 的语音嵌入 16 bit 水印；攻击者使用黑盒或开源零样本 VC，以水印 prompt 和任意文本生成新内容。
- 验证者只访问合成波形，需恢复消息并在 100 个候选身份中正确归因；FAR 衡量误归属，而非单纯 bit ACC。
- 攻击/信道包括 CosyVoice、F5-TTS、MaskGCT，以及 EnCodec、重采样、幅度、滤波、白噪声和 MP3。
- 训练从未调用目标 VC 模型，而用静音遮罩、内容打乱、局部还原、神经 codec 和常规扰动模拟 VC 对内容和水印的改变，强调跨模型泛化。
- 方法假设 SpeechTokenizer 的第 1 RVQ 层主要是内容、2–8 层主要是说话人。若目标 VC 使用完全不同、强归一化或显式去说话人细节的表示，迁移可能下降。
- 未测试已知 VoiceMark 参数的白盒水印移除、对抗 prompt、物理重录或候选池大幅扩展。

## 3. 方法设计与复现级解读

### 3.1 全局流程

预训练 SpeechTokenizer 将输入波形压缩为 8 层 RVQ token。第 1 层在 HuBERT 语义教师蒸馏下承担内容，保持不变；第 2–8 层作为说话人潜变量，与十六进制消息通过交叉注意力融合，再由 codec decoder 重建水印波形。训练时波形经过 VC 模拟增强，重新编码并减去第 1 层内容潜变量，Transformer decoder 从剩余 speaker latents 全局恢复消息和逐帧水印/VAD 概率。部署时嵌入端处理 prompt，零样本 VC 位于系统外，验证端只对其合成音频运行同一潜变量提取与解码。

### 3.2 说话人潜变量分离

输入 $x$ 经编码器得 $l\in\mathbb R^{t\times d}$，8 层 RVQ 输出 $z_1,\ldots,z_8$。$z_1$ 由 HuBERT 指导承载内容，$z_2$–$z_8$ 被视为说话人专属。水印只修改后七层，避免直接绑定原句音素；重建时求和 $\hat z=z_1+\sum_{i=2}^8\hat z_i$。

这种“专属”是由 tokenizer 训练目标促成的解耦，不是严格独立性证明。水印能否迁移取决于后层确实被目标 VC 的 speaker encoder/生成器保留。

### 3.3 交叉注意力嵌入

16 bit 先转为 4 个十六进制符号 $w\in\{0,\ldots,15\}^4$，投影成 $w'\in\mathbb R^{4\times d}$。四层 Transformer decoder 以 speaker latents 为 query、消息为 key/value：

$$\{\hat z_2,\ldots,\hat z_8\}=W_e(\{z_2,\ldots,z_8\},w').$$

交叉注意力让每个时间位置从全局短消息读取条件，而消息长度从 16 个 bit 降为 4 个分类 token，缓解长序列融合。

### 3.4 VC 模拟增强

作者用五类变换替代真实 VC 训练：20% 概率把某些帧置零，模拟静音无水印；50% 概率以 50 ms 窗打乱内容；50% 概率把 50 ms 段替换回原音，模拟局部过滤；用 EnCodec 编解码；以 10% 概率施加速度、幅度、滤波或重采样。目标不是逼真生成新语音，而是切断水印对固定内容和局部连续位置的依赖。

### 3.5 全局解码与 VAD 引导

增强音频 $\tilde x$ 重新编码为 $l$，减去第一层 $z_1$ 得 $l_s=l-z_1$。四个可学习 CLS token 与 $l_s$ 拼接，经 8 层 Transformer encoder 输出 $4\times16$ 十六进制分类概率 $\hat w$，同时输出每帧水印概率 $\hat p$。VAD 双阈值把“同时有语音和水印”的帧标 1，静音、遮罩或还原帧标 0，使模型聚焦真正会被 VC 携带的有声区域。

### 3.6 优化目标

$L_{vad}$ 是帧概率 BCE；$L_{cos}$ 保持嵌入前后 speaker latents 余弦相似；$L_{mel}$ 与对抗损失保持波形质量；$L_{dec}$ 对四个十六进制符号做交叉熵：

$$L=\lambda_{vad}L_{vad}+\lambda_{cos}L_{cos}+\lambda_{mel}L_{mel}+\lambda_{adv}L_{adv}+\lambda_{dec}L_{dec}.$$

一次训练迭代包含潜变量分解、消息交叉注意力、codec 重建、随机 VC 增强、再次编码、全局消息/VAD 解码和联合反传。预训练 SpeechTokenizer 是否完全冻结，正文未明确，应查代码确认。

| 配置 | 论文设置 |
|---|---|
| 训练/测试 | VCTK 训练；VCTK 2,000 + LibriSpeech 2,600 测试，共 150 位未见说话人 |
| tokenizer | 预训练 SpeechTokenizer，8 层 RVQ |
| 消息 | 16 bit→4 个 base-16 token |
| Embedder | 4 层、1 head、256-d Transformer Decoder |
| Decoder | 8 层、1 head、512-d Transformer Encoder |
| 优化 | Adam，lr $5\times10^{-5}$，30 epochs |
| 损失权重 | $\lambda_{vad}=1,\lambda_{cos}=2,\lambda_{mel}=2,\lambda_{adv}=1,\lambda_{dec}=1$ |
| VC 测试 | CosyVoice、F5-TTS、MaskGCT |
| 开源/演示 | `https://huggingface.co/spaces/haiyunli/VoiceMark` |

## 4. 与其他方法对比

| 方法 | 水印载体 | 穿过 VC 的机制 | 优点 | 局限 |
|---|---|---|---|---|
| AudioSeal/WavMark | 通用波形扰动 | 无专门机制 | 常规编辑强 | 零样本 VC 后接近随机 |
| Timbre | 音色相关扰动 | 训练式/音色迁移 | 针对克隆 | 零样本结果仍弱 |
| AudioMarkNet | 训练数据低频水印 | 微调 TTS 学入参数 | 可归因训练式克隆 | 无适配则失效 |
| VoiceMark | RVQ speaker latents | 随 prompt 音色直接迁移 | 首次覆盖零样本 VC | 依赖潜变量解耦与 codec 音质 |

## 5. 实验表现与优势

### 5.1 零样本 VC 主结果

测试为同一水印 prompt 经 CosyVoice、F5-TTS、MaskGCT 生成新文本。VoiceMark 的 ACC/FAR 为 0.964/0.112、0.979/0.070、0.957/0.141；AudioSeal、WavMark、Timbre 的 ACC 多为 0.499–0.539、FAR 0.95–1.0。该差距直接支持 speaker-latent 载体是跨零样本生成的关键。

FAR 仍为 7%–14%，明显高于 bit ACC 看起来的 2%–4% 错误，因为候选匹配会放大少量 bit 错误。真实更大身份池下可能更困难，论文只模拟 100 个候选。

### 5.2 常规编辑

EnCodec、重采样、幅度、滤波、白噪声、MP3 下 ACC 为 0.965–0.995，FAR 0.014–0.109。相比之下传统方法部分条件可达 1.0，VoiceMark 的优势主要在 EnCodec 和零样本 VC，不是所有普通编辑。

### 5.3 消融

把 latent embedder 换为 AudioSeal 波形嵌入但保留 VoiceMark decoder，CosyVoice ACC/FAR 从 0.964/0.112 降至 0.663/0.906，说明载体选择贡献最大。去 VAD loss 降至 0.478/0.980，甚至低于随机附近；去增强为 0.626/0.924。三个核心设计都有直接且幅度很大的验证。

### 5.4 音质

VoiceMark PESQ/SI-SNR/STOI/SMOS 为 2.20/2.01/0.89/$4.25\pm0.13$，低于直接波形 AudioSeal 的 4.32/26.69/0.99/4.67，但接近预训练 SpeechTokenizer 的 2.58/1.64/0.89/4.63，并显著好于 EnCodec 的主观分。水印效果因此带有神经 codec 重建上限，不能只把相对 AudioSeal 的下降归因于水印扰动。

## 6. 学习与应用

复现应先验证 SpeechTokenizer 分层解耦：以 speaker/content probing 检查 $z_1$ 与 $z_{2:8}$，再训练水印。迁移到新的 codec 或 VC 时，应找到其说话人条件分支，而不是机械修改后几层 codebook。部署可用于艺人 prompt 发布和零样本 TTS/VC 追踪，但 16 bit 和 100 候选 FAR 需要更强纠错、密钥和开放集阈值。

## 7. 总结

把水印写入零样本克隆必传的音色潜变量。

## 8. 图表精读与证据链

图 1 区分训练式与零样本克隆；图 2 是 speaker latent—增强—全局解码链；表 1 把零样本 VC 与普通编辑并列，证明优势来自任务匹配；表 2 对应载体、VAD、增强三项消融；表 3 说明 codec 音质代价。最强证据是三个真实零样本模型，缺口是更大候选池、未知语言、白盒去水印和物理信道。

## 9. 复现难度与适合人群

难度高，需要 SpeechTokenizer、三套大零样本 VC、VCTK/LibriSpeech 和主观听测。最小版本可只用 CosyVoice，复现 latent embedder 与去增强消融。适合零样本 TTS/VC 安全、语音 codec 表示和主动溯源研究者。

## 10. 简短全面总结

VoiceMark 解决训练式水印无法覆盖零样本声音克隆的问题。它利用 SpeechTokenizer 的八层 RVQ，把第一层作为内容、后七层作为说话人潜变量，只在后者中通过交叉注意力写入 16 bit 消息，使水印随 prompt 的音色条件进入新文本合成语音。训练不依赖特定 VC，而以静音遮罩、内容打乱、局部还原、EnCodec 和信号扰动模拟生成过程，再用全局 Transformer 与 VAD 引导恢复消息。CosyVoice、F5-TTS、MaskGCT 后 ACC 均超过 95%，传统水印约为随机；但音质受神经 codec 上限影响，100 候选 FAR 仍为 7%–14%，白盒移除和更大身份池尚未验证。

## 11. 论文写作逻辑分析

论文用传统 VC 与零样本 VC 的数据流差异明确划出缺口，从“零样本只保留说话人条件”自然推导 speaker-specific carrier。增强和 VAD 分别回应内容变化、局部过滤与静音，模块出现顺序合理；三项消融的下降巨大，证据链很强。写作可改进处是对 RVQ 层解耦假设的验证较少，音质损失也应更明确区分 tokenizer 重建与额外水印造成的部分。
