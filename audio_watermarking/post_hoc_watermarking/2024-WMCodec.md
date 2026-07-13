# WMCodec：把深度水印内生到神经语音编解码器

## 0. 翻译摘要原文

语音伪造的发展要求神经语音 codec 提供更强真实性验证。现有方案在压缩前嵌入数值水印、从重建语音提取，但水印与 codec 分开训练、跨模态信息融合不足，导致透明度、提取精度和容量受限。WMCodec 首次把压缩重建与水印嵌入提取端到端联合训练，同时优化水印不可感知性与可提取性；迭代 Attention Imprint Unit 进一步深度融合消息和语音特征，减轻量化噪声影响。在 6 kbps、16 bps 水印下，常见攻击后仍保持 99% 以上提取准确率，并在多数音质指标上优于 AudioSeal+Encodec，在恢复率上优于该基线及强化 TraceableSpeech。

## 1. 方法动机

普通流程先用独立水印器改波形，再交给已训练 codec；量化器被视为未见攻击，水印器无法适应其离散误差。简单加法/拼接又使数值消息只浅层进入 speech latent，低码率压缩后容易消失。

WMCodec 把水印放在 codec encoder 与 RVQ 之间，并让提取损失穿过量化和 speech decoder 联合优化。AIU 以水印为 key/value、语音为 query，反复把消息“压印”到每个时间特征，使其在有限 codebooks 中与载体共同编码。

## 2. 威胁模型解读

- 发送方用特定神经 codec 压缩并在码流对应的重建语音中留下验证标记；接收方从解码波形恢复数字消息，判断是否经过授权 codec 流程。
- 攻击/信道包括 RVQ 压缩本身、重采样、随机噪声、丢样、幅度衰减、回声、低通，以及删除三分之一后拼接的 resplicing。
- 模型保护目标是“重建语音携带正确标记”，但论文未给出认证协议：攻击者复制合法水印波形、重放旧消息或伪造 bit 时如何防护，需要外部密钥/签名。
- 实验没有 MP3/AAC、物理重录、变速、神经生成重合成或白盒移除；“真实性验证”实际证明的是水印可恢复，不等价于不可伪造。
- 编解码双方必须使用 WMCodec；它不是可后处理任意第三方 codec 的通用水印器。

## 3. 方法设计与复现级解读

### 3.1 全局流程

波形经 speech encoder 下采样为 $z_s$；m 位 base-b 数值消息经 embedding、拼接和两层线性网络成为单帧向量，再沿时间复制成 $z_w$；多个 AIU 以 cross-attention 把 $z_w$ 逐步融合到 $z_s$。融合 latent 进入 RVQ 产生离散码并恢复为 $z'$，speech decoder 输出水印重建语音 $x_w$。训练时 $x_w$ 经失真层，Mel 谱送入 ResNet 水印解码器，逐位分类。codec 的语音重建、量化、对抗和消息交叉熵联合更新。

### 3.2 消息编码与时间广播

每个 base-b 数字先查 embedding，所有数字均匀拼接并经两层线性得到 $z_o$，随后 Repeat 到与 speech latent 相同的 T：

$$z_w=\operatorname{Repeat}(ENC_w(w)).$$

4@16 表示四位十六进制，容量 $\log_2(16^4)=16$ bit；4@10 为 $\log_2(10^4)\approx13.3$ bit。论文将其换算为 bps，隐含每个评测单元约一秒/每秒一条消息。

### 3.3 迭代 Attention Imprint Unit

AIU 输入 speech $H_s$ 与 watermark $H_w$，先 layer norm；multi-head cross-modal attention 的 query 来自语音，key/value 来自水印：

$$H'_{w\to s}=MHCA(LN(H_s),LN(H_w))+H_s,$$
$$H_{w\to s}=FFN(LN(H'_{w\to s}))+H'_{w\to s}.$$

两次迭代让每个时刻根据自身语音状态选择消息特征，而不是固定拼接同一向量。非对称方向确保语音内容主导载体，水印逐层渗透，不反向改变消息表示。

### 3.4 RVQ 与水印解码

融合特征经 HifiCodec 风格 RVQ，使用 4/8/16 codebooks 对应 3/6/12 kbps。量化恢复的 $z'$ 经 speech decoder 生成波形。水印解码器不直接读取 codec bitstream，而从重建波形 Mel 谱经 ResNet221 得高维向量，再为每个数字做线性分类。这保证接收端只持有音频也能验证，但也丢弃了码流侧可用的显式认证信息。

### 3.5 联合训练目标

正文说明目标沿用 TraceableSpeech，包括生成器/判别器对抗损失、quantizer loss、speech reconstruction 相关损失及原/预测消息间交叉熵。其关键梯度路径是：消息 CE 从 watermark decoder 经过 Mel、speech decoder、RVQ 的训练近似、AIU 回到消息/语音编码器，使量化器主动保留消息。

论文没有在公式中完整展开各项权重和训练调度，因此忠实复现需参考其代码或 TraceableSpeech 实现；不能从“similar to”推断具体数值。

| 配置 | 论文设置 |
|---|---|
| 数据 | LibriTTS 585 h、24 kHz、2,456 speakers；随机 200 test utterances，每条嵌入 10 次 |
| 消息 | 4@16：16 bps；4@10：13.3 bps |
| AIU | 2 iterations，8 heads；speech/watermark dim 均 512 |
| RVQ | HifiCodec-like，group size 1；4/8/16 codebooks→3/6/12 kbps |
| 解码器 | ResNet221 on Mel spectrogram |
| 训练 | batch 32，150k updates |
| 基线 | AudioSeal+Encodec；重新安排量化位置并训练的 TraceableSpeech* |
| 指标 | PESQ、STOI、ViSQOL、MOS、数字 extraction accuracy |
| 缺失 | 优化器/lr、各损失权重、片段长度、随机种子、代码链接与完整安全协议 |

## 4. 与其他方法对比

| 方法 | codec 与水印关系 | 融合 | 优点 | 局限 |
|---|---|---|---|---|
| AudioSeal+Encodec | 独立串联 | 波形加性 | 使用现成模型 | 低码率恢复接近随机 |
| TraceableSpeech* | 联合训练 | 简单拼接 | 量化感知 | 消息渗透不足 |
| WMCodec | 原生端到端 | 迭代 cross-attention | 量化主动保水印 | 绑定特定 codec、协议未完备 |

## 5. 实验表现与优势

### 5.1 重建音质/不可感知性

3 kbps、16 bps 时 WMCodec PESQ/STOI/ViSQOL/MOS 为 2.606/0.898/3.549/$4.152\pm0.20$，AudioSeal+Encodec 为 2.134/0.894/3.643/$2.908\pm0.24$，ViSQOL 并非全面领先。6 kbps、16 bps 时 WMCodec 3.187/0.936/4.009/4.434，接近 TraceableSpeech* 3.206/0.933/4.014/4.458，并优于 AudioSeal+Encodec 多数指标。12 kbps 时本文在全部列上超过 AudioSeal 基线。

4@10 比 4@16 音质更高，例如 6 kbps PESQ 3.401、MOS 4.513，验证消息容量增加会损伤重建质量。

### 5.2 提取准确率与常规攻击

3 kbps、16 bps 时 WMCodec 平均提取 0.755，明显高于 AudioSeal+Encodec 的 0.525，但还不足可靠认证。6 kbps 时 WMCodec 4@16 在 normal、RSP、noise、dropout、amplitude、echo、low-pass、resplicing 上为 0.983–0.998，平均 0.995；AudioSeal 为 0.623，TraceableSpeech* 为 0.763。12 kbps 平均同为 0.995。

删除三分之一并重新拼接是最强条件之一：6 kbps 4@16 仍为 0.983，12 kbps 为 0.967。论文由此支持端到端量化和 AIU 的鲁棒性，但攻击类型仍有限。

### 5.3 AIU 消融

TraceableSpeech* 被作者视为直接拼接的消融：在 6 kbps、16 bps 下，平均提取从 0.763 提升到 WMCodec 的 0.995，而 PESQ/MOS 3.206/4.458 与 3.187/4.434 基本相当。说明 AIU 主要提升消息可提取性，没有明显追加音质代价。严格说两系统除融合模块外是否完全等价需以实现核对，正文没有更细 component ablation。

## 6. 学习与应用

复现应先固定 6 kbps、4@16，确认 CE 梯度能穿过 RVQ；再比较 concat 与 1/2/更多 AIU。真实性应用必须把 message 设计为带 nonce、时间戳和数字签名的短认证码，避免可复制水印被当作真实性证明。该结构可迁移到 EnCodec、DAC、SoundStream 等 codec，但每个量化器都需联合重训。

## 7. 总结

让量化器与水印联合学习并用注意力深度融合。

## 8. 图表精读与证据链

图 1 界定 codec 接收方验证场景；图 2 展示 AIU 位于 encoder 与 RVQ 之间；表 I 分带宽比较音质和容量；表 II 同时证明低/中/高码率与攻击鲁棒性。最强证据是 6 kbps 下 0.995 对 0.623/0.763 的恢复差距；缺口是消融不够细、训练目标未完整披露、攻击面和认证协议不足。

## 9. 复现难度与适合人群

难度高，需从头训练神经 codec、RVQ、判别器、AIU 和 watermark ResNet，150k steps 且许多超参数缺失。最小版本可基于开源 AcademiCodec/HifiCodec 重现 6 kbps 单配置。适合神经 codec、语音传输安全和端到端多模态融合研究者。

## 10. 简短全面总结

WMCodec 不再把 codec 当作水印后的外部攻击，而是在 speech encoder 与 RVQ 之间嵌入数字消息，让压缩重建与水印恢复端到端联合优化。四位 base-16/10 消息经时间广播，两层迭代 AIU 以语音为 query、水印为 key/value 深度融合，量化后由 speech decoder 重建波形，再从 Mel 谱恢复数字。LibriTTS 实验中，6 kbps、16 bps 下常规攻击和删除重拼后平均提取准确率 99.5%，远高于 AudioSeal+Encodec 62.3% 和直接拼接 TraceableSpeech* 76.3%，音质相近或更好。主要边界是 3 kbps 仅约 75.5%，训练细节不完整，且可恢复水印尚不足以构成抗伪造的真实性协议。

## 11. 论文写作逻辑分析

论文从“水印与 codec 分开训练导致误差累积”引出端到端框架，再由简单拼接不足引出 AIU，方法因果关系明确。实验按带宽同时控制音质、容量和恢复，并把重排后的 TraceableSpeech 作为融合消融，核心 claim 有数据支撑。写作不足是篇幅短导致损失函数、威胁模型和实现细节省略较多；“authenticity verification”也超出当前仅验证可提取性的证据，应增加密钥、重放和伪造分析。
