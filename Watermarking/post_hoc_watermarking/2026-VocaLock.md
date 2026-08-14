# VocaLock：基于水印的零样本语音转换操纵检测与音色归因

## 0. 翻译摘要原文

零样本语音转换（ZSVC）可以在无需微调的情况下，把未见说话人的音色迁移到另一段语音上，因此带来了音频滥用和版权侵权风险。为保护潜在受害者，论文提出 VocaLock：一种基于水印的系统，用于在 ZSVC 攻击下检测伪造音频并归因说话人音色。VocaLock 在用户音频中嵌入单个水印，并训练两个具有不同鲁棒性目标的解码器：鲁棒解码器用于音色归因，半鲁棒解码器用于伪造音频检测。当 ZSVC 将被操纵内容归因到受害者时，鲁棒解码器能够成功提取水印，而半鲁棒解码器会失效。为了抵抗 VC 引入的结构性失真，VocaLock 将水印嵌入 STFT 频谱，并引入 post-VC loss，使水印嵌入与音色特征空间对齐，从而让水印痕迹随音色特征一起迁移到伪造音频中。通过跨域失真训练，方法进一步增强了对黑盒 VC 模型的鲁棒性和泛化能力；同时结合额外的结构与训练策略平衡鲁棒性和音频保真度。实验表明，VocaLock 对多种 ZSVC 模型和常见后处理具有较强鲁棒性和泛化能力，并保持了良好的音频质量。

## 1. 方法动机

传统音频水印默认信道是压缩、重采样、噪声、裁剪等信号级操作；但 ZSVC 不是简单信道，它会先从源语音中抽取内容，再从目标语音中抽取音色，最后重新合成语音。这个过程会丢弃大量原始波形细节，所以 AudioSeal、WavMark 这类通用鲁棒水印在 VC 后容易退化到随机水平。

VoiceMark / Timbre Watermarking 关注 voice cloning，但 VocaLock 指出的关键差异是：ZSVC 的操纵对象不是“用 prompt 生成文本对应语音”那么简单，而是“源内容 + 目标音色”的重组合成。版权或法证场景里只知道可疑音频像某个用户，平台需要回答两个问题：第一，它是否被 ZSVC 操纵；第二，音色是否来自该用户。仅能做归因不够，因为真实上传音频也会携带该用户水印；仅能做伪造检测也不够，因为不能证明音色来源。

VocaLock 的核心设计因此是同一个水印产生两个判别信号：鲁棒解码器要在原音频、后处理音频、VC 伪造音频里都读出用户水印，用于追踪音色来源；半鲁棒解码器只需要抵抗普通后处理，但要对 VC 结构性失真敏感，用于区分真实传播音频和 ZSVC 伪造音频。

## 2. 威胁模型解读

- 平台给每个注册用户分配唯一二进制水印 $w_{ori}$，并在用户上传音频 $x_t$ 发布前嵌入。平台维护水印与用户身份的映射，后续由平台后端验证。
- 攻击者可把受保护音频作为目标音色参考，通过 ZSVC 生成“任意源内容 + 受害者音色”的伪造音频。论文关注的是受害者发现疑似音频后提交平台检测，而不是实时阻止攻击者生成。
- 攻击信道包含白盒 TriAAN-VC 训练信道，以及黑盒 AdaIN-VC、S2VC、FragmentVC；还考虑 VC 后再叠加重采样、MP3、裁剪等平台后处理。
- 验证端不是开放集声纹识别，而是针对认证用户取出其平台水印，再比较两个解码器输出与该水印的距离。
- 论文假设攻击者不知道平台水印与解码器细节，未系统评估白盒水印移除、对抗性去水印、物理重录或扩散式语音编辑。结论主要覆盖 ZSVC 和常规音频后处理。

平台验证逻辑如下。若可疑音频为 $x_{sus}$，鲁棒解码器和半鲁棒解码器分别输出 $w^r_{sus}$ 与 $w^s_{sus}$：

$$w^r_{sus}=D_r(x_{sus}),\quad w^s_{sus}=D_s(x_{sus}).$$

与用户原始水印 $w_{ori}$ 比较后得到四类结果：

| 半鲁棒解码器 | 鲁棒解码器 | 判定 |
|---|---|---|
| 匹配 | 匹配 | 合法/真实音频，且可归因到该用户 |
| 不匹配 | 不匹配 | 未发现该用户水印 |
| 不匹配 | 匹配 | ZSVC 伪造，音色归属于该用户 |
| 匹配 | 不匹配 | 理论上不符合鲁棒性层级，视为异常或不确定 |

## 3. 方法设计与复现级解读

### 3.1 总体流程：一个 encoder，两个 decoder，三阶段训练

VocaLock 的部署流程是：平台用 encoder $E(\cdot)$ 把用户水印嵌入上传音频，得到水印音频 $x_{wm}$；如果该音频后来被用作 ZSVC 的目标音色，生成的伪造音频仍应携带可被鲁棒解码器读出的水印；但由于 VC 对内容、时序和频谱结构做了重构，半鲁棒解码器应无法稳定恢复原水印，从而给出“伪造”信号。

训练分三步：

1. Stage 1 联合训练 encoder 和鲁棒解码器，让水印在音质可接受的前提下穿过白盒 VC 信道。
2. Stage 2 冻结 encoder，只训练鲁棒解码器，使其额外抵抗白噪声、低通滤波等普通后处理，同时避免遗忘 VC 鲁棒性。
3. Stage 3 冻结 encoder 与鲁棒解码器，只训练半鲁棒解码器，使它对普通后处理鲁棒、对 VC 输出敏感。

这个流程的重点不是“两个解码器结构不同”，而是训练目标不同。两个 decoder 架构相同，差异来自鲁棒性层级的监督设计。

### 3.2 STFT 幅度谱嵌入：把水印放在更接近音色的表示里

多数 ZSVC 模型从 mel-spectrogram 或 SSL 表示中抽取目标音色。mel 本身来自 STFT，因此作者选择 STFT magnitude 作为水印载体，希望水印更接近音色相关谱包络，而不是易被重建过程抹掉的波形相位细节。对目标音频做 STFT：

$$S_t,P_t=\mathrm{STFT}(x_t).$$

$S_t$ 是幅度谱，$P_t$ 是相位。VocaLock 只在幅度谱中嵌入水印，最后 ISTFT 时复用原相位。论文选择 $w_l=1024,h_l=256,f=1024$，理由是较高频率分辨率更能表示谐波和音色结构，同时提供更多冗余嵌入空间；窗口过短会绑定局部瞬态，容易被 VC 的内容重构破坏。

encoder 先用 ConvBNReLU 提取局部谱特征，再接 SENet 做通道重加权：

$$S^f_t=F_{SE}(F_{conv1}(S_t)).$$

水印消息 $w_{ori}$ 先映射到 $\{-1,1\}$，经线性层投影到频谱维度，并沿时间轴复制，以获得时间不变性：

$$W_t=\mathrm{Repeat}(F_{linear}(w_{ori}),T).$$

随后通过卷积处理水印特征并与 STFT 特征拼接：

$$F_t=F_{conv2}(W_t)\oplus S^f_t.$$

最终由卷积块输出水印幅度谱，并用原相位重建：

$$x_{wm}=\mathrm{ISTFT}(F_{conv3}(F_t),P_t).$$

### 3.3 双解码器：同构网络，不同鲁棒性目标

鲁棒解码器 $D_r$ 和半鲁棒解码器 $D_s$ 使用相同结构：输入音频先转 STFT magnitude，再经卷积块和 SENet 提取局部谱特征，随后压缩通道、时间平均池化，最后线性映射到水印长度：

$$S^f_{wm}=F^{de}_{SE}(F^{de}_{conv1}(S_{wm})),\quad w_{de}=F_{linear}(\mathrm{mean}(F^{de}_{conv2}(S^f_{wm}))).$$

这里的平均池化很重要：它迫使水印从全局时间维度恢复，而不是依赖某个固定片段。对于 VC、crop、frame-level 改写等场景，过度依赖局部位置会降低泛化。

### 3.4 跨域 VC distortion：让 STFT 水印穿过 CPC 语音转换

作者没有直接在前向路径里寻找一个“通用音色空间”再嵌入水印。消融实验显示，显式把水印嵌入 CPC 或 mel 分离特征会过拟合某类 VC 表示，黑盒泛化差。VocaLock 改用间接策略：水印嵌在 STFT 域，但训练信道用 CPC 表示域的 TriAAN-VC，让梯度反向迫使水印学到可跨表示迁移的痕迹。

具体地，先从水印目标音频 $x_{wm}$ 和源音频 $x_s$ 提取 CPC 表示：

$$cpc_{wm},cpc_s=\mathrm{CPC}(x_{wm},x_s).$$

再提取源音频的基频特征 $lf0_s$，送入 VC 模型生成伪造 mel：

$$M^{st}_{wm}=\mathrm{VC}(cpc_{wm},cpc_s,lf0_s).$$

最后用 ParallelWaveGAN vocoder 得到伪造水印音频：

$$x^{st}_{wm}=\mathrm{VOC}(M^{st}_{wm}).$$

“cross-domain”不是指嵌入和提取在不同域，而是指水印嵌入/提取在 STFT 域，训练失真由 CPC 表示域的 VC 产生。这样鲁棒解码器不能只记住某种 STFT 级扰动，而必须适配表示空间变化、音色/内容分离与重合成导致的结构性变化。

### 3.5 普通后处理 distortion：只在后两阶段加入

普通后处理层随机执行白噪声、低通滤波或 identity：

$$\mathcal T\sim P(\pi_{wn},\pi_{lpf},\pi_{id}),\quad \pi_{wn}+\pi_{lpf}+\pi_{id}=1.$$

Stage 2 中采样比例为 clean:white-noise:low-pass = $0.4:0.3:0.3$。作者没有把所有后处理一开始就混入端到端训练，是因为 VC 结构性失真和信号级失真目标冲突，容易导致收敛不稳和音质下降。先让 encoder 学会“能穿过 VC 的嵌入”，再冻结 encoder 扩展 decoder 的后处理鲁棒性，是这篇可复现时最关键的训练顺序。

### 3.6 损失函数：鲁棒归因与半鲁棒检测分开优化

Stage 1 的 fidelity loss 包含 waveform MSE 和 AudioSeal 风格的 TF-Loudness 感知损失：

$$L_{mse}=\mathrm{MSE}(x_{wm},x_t).$$

$$L_{TF}=\mathbb E_{b,m}\left[\mathrm{Softmax}\left(\frac{L^{(b,m)}_{noise}-L^{(b,m)}_{ref}}{\tau}\right)\cdot \mathrm{ReLU}(L^{(b,m)}_{noise}-L^{(b,m)}_{ref})\right].$$

$$L_{en}(x_t,w_{ori})=L_{mse}+L_{TF}.$$

鲁棒解码器既要从原水印音频中读出水印，也要从 VC 后音频中读出水印：

$$L_{de}(x_t,w_{ori})=\mathrm{MSE}(D_r(x_{wm}),w_{ori})+\mathrm{MSE}(D_r(x^{st}_{wm}),w_{ori}).$$

Stage 1 总损失为：

$$L_{S1}(x_t,w_{ori})=\lambda_{en}L_{en}+\lambda_{de}L_{de}.$$

论文使用 progressive loss schedule：0–40 epoch 为 $\lambda_{en}:\lambda_{de}=1:10$，40–50 epoch 为 $100:1$；50–100 epoch 设 $\lambda_{en}=1000$，100 epoch 后每 50 epoch 增加 1000，$\lambda_{de}=1$ 保持不变。这个 schedule 体现了训练逻辑：前期先保证水印能被读出，后期强力压音质损失。

Stage 2 冻结 encoder，鲁棒解码器同时看普通后处理和 VC：

$$L_{S2}(x_t,w_{ori})=\mathrm{MSE}(D_r(\mathcal T(x_{wm})),w_{ori})+\mathrm{MSE}(D_r(x^{st}_{wm}),w_{ori}).$$

Stage 3 的半鲁棒解码器要实现“普通后处理能读、VC 后读不回原水印”。作者利用消息映射到 $\{-1,1\}$ 后再 sign 恢复的特点，构造全 1 pseudo watermark $w_p$，只作为 VC 输出的训练目标，不参与推理验证：

$$L_{S3}(x_t,w_{ori},w_p)=\mathrm{MSE}(D_s(x^{st}_{wm}),w_p)+\mathrm{MSE}(D_s(\mathcal T(x_{wm})),w_{ori}).$$

这个设计比简单把 VC 后输出推向 0 更有效。因为最终 bit 由符号决定，数值靠近 0 仍可能保留正确符号；推向全 1 会系统性破坏原始随机水印的 bitwise 恢复，使 VC 后准确率接近 0.5。

### 3.7 复现配置与实现注意点

| 配置 | 论文设置 |
|---|---|
| 数据集 | VCTK 主训练/测试；LibriSpeech 做 unseen-domain 泛化 |
| VCTK 划分 | 1,000 target + 1,000 source 训练；500 target + 500 source 测试 |
| LibriSpeech | 500 target + 500 source 测试 |
| 音频长度 | 10 秒切片 |
| payload | Ours10 与 Ours15，即 10 bit / 15 bit |
| STFT | window 1024，hop 256，FFT 1024 |
| 训练框架 | PyTorch，NVIDIA A800 |
| 优化器 | Adam，batch size 8，初始学习率 0.0008 |
| ZSVC | TriAAN-VC 白盒训练；AdaIN-VC、S2VC、FragmentVC 黑盒测试 |
| benign distortion | 白噪声、低通、重采样、MP3、裁剪等 |
| baseline | AudioSeal、WavMark、VoiceMark、Timbrewatermark |

复现时最容易出问题的是三点。第一，TriAAN-VC 与 vocoder 必须在训练图里产生可反传的 post-VC loss，否则“水印随音色迁移”的效果会弱很多。第二，半鲁棒解码器必须单独训练，不能与鲁棒解码器共享同一个最终 checkpoint。第三，评价时不要只看平均 bit accuracy；要同时检查 $D_r$ 在 VC 后高、$D_s$ 在 VC 后接近 0.5、二者在 benign 后都高，三者同时成立才符合 VocaLock 的设计目标。

## 4. 与其他方法对比

| 方法 | 主要目标 | 对 ZSVC 的机制 | 能否归因 | 能否检测伪造 | 局限 |
|---|---|---|---|---|---|
| AudioSeal | 语音克隆/局部检测水印 | 主要面向普通信号处理与生成检测 | 可检测水印 | 不区分 ZSVC 伪造与真实传播 | VC 后接近随机 |
| WavMark | 通用鲁棒音频水印 | 训练鲁棒信道偏普通后处理 | 可恢复消息 | 无半鲁棒伪造判别 | VC 后接近随机 |
| Timbrewatermark | 声音克隆攻击检测 | 面向 TTS/克隆相关音色变化 | 有音色相关性 | 对 ZSVC 不稳定 | 不覆盖 ZSVC 内容-音色重组合成 |
| VoiceMark | 零样本 VC/克隆 prompt 保护 | 修改 speaker-specific latents | 可归因 | 主要通过水印存在性判别 | 需要特定 tokenizer/latent 假设 |
| VocaLock | ZSVC 伪造检测 + 音色归因 | STFT 嵌入 + CPC-VC 跨域训练 + 双解码器 | 强 | 强 | 当前未系统评估扩散式语音编辑与白盒去水印 |

VocaLock 与 VoiceMark 最接近，但侧重点不同。VoiceMark 更像“把水印放进 speaker latent，使零样本生成继续携带水印”；VocaLock 更像“让同一水印在两个解码器上产生不同鲁棒性响应”，因此能够把“真实的水印音频传播”和“使用该音色生成的伪造内容”分开。

## 5. 实验表现与优势

### 5.1 ZSVC 鲁棒性

在 10 bit payload 下，VocaLock 的鲁棒解码器在四种 VC 上保持明显高于 baseline 的准确率：AdaIN-VC 0.977、S2VC 0.836、FragmentVC 0.925、TriAAN-VC 1.000。相比之下，AudioSeal、WavMark、Timbrewatermark 在多数 VC 后约为 0.47–0.64，接近随机猜测；VoiceMark 在部分 VC 上有提升，但不稳定，如 AdaIN-VC 0.752、FragmentVC 0.597、TriAAN-VC 0.528。

半鲁棒解码器在 VC 后保持接近 0.5：AdaIN-VC 0.502、S2VC 0.503、FragmentVC 0.497、TriAAN-VC 0.492。这正是伪造检测所需的“失效”行为，而不是方法失败。

### 5.2 VC 后再后处理

Table III 测试了 VC 输出再经过 resampling、MP3、crop 的情况。10 bit 下 $D_r$ 仍能在 AdaIN-VC、S2VC、FragmentVC、TriAAN-VC 上保持较高准确率，例如 Resample1 后分别为 0.973、0.831、0.920、0.997；MP3 后分别为 0.975、0.817、0.906、0.981。对应 $D_s$ 仍接近随机，说明检测边界没有被普通后处理显著破坏。

### 5.3 保真度

客观质量用 PESQ 和 SNR，主观质量用 10 名参与者 MOS。论文结论是 WavMark 和 AudioSeal 的保真度最高，但不抗 VC；VocaLock 在低 payload 下 SNR 接近 AudioSeal，并在相同 payload 下优于 Timbrewatermark，整体明显优于 VoiceMark。消融中 TF-Loss 使 PESQ 从 2.437 提升到 2.973，同时 ZSVC 鲁棒性也略有提升，说明感知损失不是只“美化音质”，还可能减少 VC 对水印痕迹的抑制。

### 5.4 跨数据集泛化

在 LibriSpeech unseen-domain 测试中，10 bit 模型达到平均 PESQ 3.64、SNR 接近 30 dB。VC 后 $D_r$ 仍保持较高准确率：AdaIN 0.969、S2VC 0.807、FragmentVC 0.911、TriAAN 0.988；$D_s$ 在 VC 后仍约 0.48–0.50。跨数据集下 S2VC 比 VCTK 更低，作者认为可能来自 VC 模型跨数据集本身的不稳定。

### 5.5 关键消融

- 显式特征解耦不可靠：把水印直接放到 CPC 或 mel 分离特征里，在白盒模型上能读，但黑盒泛化差，且音质很差；CPC 设置 PESQ 只有 1.06，mel 设置 PESQ 1.53。
- 跨域训练有效：in-domain AdaIN-VC 训练在 AdaIN 上可达 0.902，但 S2VC、FragmentVC、TriAAN-VC 分别只有 0.493、0.517、0.543；cross-domain TriAAN/CPC 训练提升到 0.977、0.836、0.925、1.000。
- pseudo watermark 有效：半鲁棒解码器用全 1 伪水印训练后，VC 后准确率接近随机；简单推向 0 或推大 MSE 距离在黑盒 VC 上不够稳定。
- STFT 参数有效：P2，即 1024/256/1024，整体优于 P1，即 400/160/400。原因是高频率分辨率更贴近音色与谐波结构，且给同一 payload 提供更分散的嵌入冗余。

## 6. 学习与应用

这篇对后续语音水印很有价值的点是：它没有把“鲁棒性”当成单一目标。真实版权验证里，一个水印系统可能同时需要“某些变换后必须存在”和“某些变换后必须表现出异常”。VocaLock 用 $D_r$ 和 $D_s$ 把这两个目标拆开，避免了单 decoder 无法区分真实传播和伪造生成的问题。

如果迁移到自己的语音水印系统，可以借鉴三点：

- 水印载体应贴近攻击模型会保留的语义/音色表示，而不是只追求对普通信号处理鲁棒。
- 可以通过跨域失真训练逼迫水印穿过不同表示空间，而不是显式押注某一个“正确音色空间”。
- 对检测任务可设计不同鲁棒性等级的解码器，让鲁棒性差异本身成为判据。

对 Qwen/生成式语音水印方向，VocaLock 的启发是：如果希望水印跨 VC/ASR-TTS/语音编辑传播，训练信道应包含表示域变化和重合成，而不只是 waveform augmentation。另一方面，如果任务是检测是否经过某类生成式篡改，则可以设计半鲁棒分支，使其在目标篡改下故意失效。

## 7. 总结

VocaLock 是一篇针对 ZSVC 音色滥用的主动法证水印论文。它的核心不是提出一个更强的通用音频水印，而是把“音色归因”和“伪造检测”拆成两个鲁棒性层级：$D_r$ 负责穿过 VC 读出水印，$D_s$ 负责在 VC 后读不出水印但在普通后处理后仍能读出。方法通过 STFT magnitude 嵌入、CPC-domain VC 跨域训练、三阶段优化和 pseudo watermark 监督，把水印痕迹尽量绑定到音色迁移过程中。

论文强点在于问题定义清晰、判决逻辑完整、消融能支撑关键设计。局限在于当前主要覆盖传统 ZSVC 模型，未系统评估扩散式语音编辑、强白盒去水印、物理重录和更大规模身份索引。

## 8. 图表精读与证据链

| 图表 | 说明 | 证据作用 |
|---|---|---|
| Fig. 1 | 平台保护场景：上传前嵌入、发现疑似音频后提交验证 | 说明 VocaLock 是平台后端法证系统，不是本地端实时拦截工具 |
| Table I | 对比 AudioSeal、WavMark、Timbrewatermark、VocaLock 的 ZSVC 鲁棒性和检测能力 | 论证现有方法缺少“ZSVC 后归因 + 伪造检测”双能力 |
| Fig. 2 | 三阶段训练流程，包含 real audio route 与 forged audio route | 是复现方法的主图，应按 Stage 1/2/3 理解，而不是按模块孤立阅读 |
| Table II | ZSVC 与普通后处理下的 bit accuracy | 证明 $D_r$ 高、$D_s$ 在 VC 后低、二者在 benign 后高 |
| Table III | VC 后再叠加普通后处理 | 证明伪造音频二次传播后仍可归因和检测 |
| Fig. 3 / Fig. 5 | 客观 PESQ/SNR 与主观 MOS | 证明鲁棒性提升没有造成不可接受音质损失 |
| Table IV | LibriSpeech 跨数据集泛化 | 证明不是只在 VCTK 上有效 |
| Table V | 显式 CPC/mel 特征解耦消融 | 反证“直接找音色空间嵌入”并不可靠 |
| Table VI | in-domain vs cross-domain distortion training | 支撑跨域训练是黑盒泛化关键 |
| Table VIII | pseudo watermark 训练策略消融 | 支撑半鲁棒解码器为何能对 VC 敏感 |
| Table IX | STFT 参数消融 | 支撑高频率分辨率更适合音色绑定 |

## 9. 复现难度与适合人群

复现难度：较高。主要难点不是 encoder/decoder 本身，而是需要接入可训练或至少可用于训练信道的 VC 模型、vocoder、CPC 特征提取，并严格复现三阶段冻结策略。

适合人群：

- 做语音水印、音色版权保护、VC/voice cloning 法证的研究者；
- 需要区分“真实音频传播”和“音色被拿去生成新内容”的平台方；
- 想研究多鲁棒性层级水印、fragile/semi-fragile watermark 的同学。

不太适合只想要通用音频水印工具的人。VocaLock 的系统假设较强：平台分发水印、平台保存身份映射、平台执行验证，而且重点是 ZSVC 语音转换场景。

## 10. 简短全面总结

VocaLock 解决的是 ZSVC 下的“音色被盗用后如何证明”：用户上传音频先被平台嵌入水印；若攻击者用该音频作为目标音色生成伪造语音，鲁棒解码器仍应读出水印，从而归因音色来源；半鲁棒解码器则应在 VC 后读不出原水印，从而判断该音频不是正常传播的原始水印音频。方法通过 STFT 幅度谱嵌入、CPC-domain TriAAN-VC 跨域训练、三阶段冻结优化和 all-ones pseudo watermark，使同一个水印同时服务于归因与伪造检测。实验显示它在多种黑盒 ZSVC 和普通后处理下明显优于 AudioSeal、WavMark、VoiceMark、Timbrewatermark，但未来仍需验证扩散式语音编辑、白盒去水印和物理信道。

## 11. 论文写作逻辑分析

这篇论文的写作逻辑很清楚，适合作为“问题定义驱动方法设计”的样例。

第一步先把风险从泛泛的 deepfake detection 收窄到 ZSVC 音色滥用。作者强调 ZSVC 的危害不只是“生成假音频”，还涉及音色产权、职业声音资产和法律归因。

第二步指出单一能力不足：只检测伪造无法证明音色来自谁，只做水印归因又无法说明音频是否被重新合成。因此论文自然引出“双目标”：forgery detection + timbre attribution。

第三步把挑战拆成四个工程问题：VC 会破坏波形水印；不同 VC 的音色表示不统一；VC 结构性失真与普通后处理冲突；两个相同结构 decoder 天然鲁棒性相似但系统需要它们表现不同。

第四步逐个回应挑战：STFT magnitude 嵌入对应音色相关载体；CPC-domain surrogate VC 对应跨域泛化；三阶段训练对应稳定性和音质；pseudo watermark 对应半鲁棒解码器的敏感性塑形。

第五步实验也按设计链条组织：主表证明整体能力，VC+后处理证明真实传播场景，跨数据集证明泛化，feature disentanglement / cross-domain / TF-Loss / pseudo watermark / STFT 参数消融逐项支撑方法选择。整体上，这篇论文不是靠堆模型赢，而是靠判决逻辑和训练目标设计把系统需求闭环。

## Overview.md 条目

- [VocaLock: Watermark-Based Detection of Zero-Shot Voice Conversion Manipulation and Timbre Attribution](./Watermarking/post_hoc_watermarking/2026-VocaLock.md)  
  *IEEE Transactions on Information Forensics and Security, 2026*  
  Citation: Zhang, Y., Ye, D., Tondi, B., & Barni, M. “VocaLock: Watermark-Based Detection of Zero-Shot Voice Conversion Manipulation and Timbre Attribution.” *IEEE Transactions on Information Forensics and Security*, 2026.  
  Links: [DOI](https://doi.org/10.1109/TIFS.2026.3723197) | Code: Not found
