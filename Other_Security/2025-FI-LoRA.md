# Scalable Dual Fingerprinting for Hierarchical Attribution of Text-to-Image Models

## 0. 摘要翻译

生成式人工智能的商业化催生了由模型开发者、服务提供商和消费者组成的多层生态。因此，可追踪性变得非常关键，因为服务提供商可能侵犯知识产权，消费者也可能生成有害内容。然而，已有方法局限于单层归因场景，无法同时跨多个层级追踪。为此，本文提出一种面向 text-to-image（T2I）模型的可扩展双指纹方法，同时实现对服务提供商和消费者的追踪。具体来说，作者提出 2-head Fingerprint-Informed Low-Rank Adaptation（FI-LoRA），其中每个 head 都由一个二进制指纹控制，并能把指纹引入生成图像。实践中，一个 FI-LoRA head 由开发者使用，用于给每个服务提供商分配唯一指纹；另一个 head 开放给服务提供商，用于在图像生成过程中嵌入消费者特定指纹。实验表明，该方法适用于多个 T2I 模型的多种图像生成与编辑任务，并能对两个指纹都达到 99.9% 以上的提取准确率。该方法还对图像级攻击和白盒模型级攻击表现出良好鲁棒性。代码开放在 `https://github.com/jumpycat/FI-LORA`。

## 1. 方法动机

FI-LoRA 关注的是 GenAI 商业生态中的分层责任追踪。现实部署中，模型开发者把 T2I 模型授权给服务提供商，服务提供商再把在线生成服务提供给消费者。这里存在两个不同层级的风险：服务提供商可能私自再分发、滥用或侵犯模型开发者知识产权；消费者可能使用服务生成 deepfake、有害图像或违法内容。单一水印只能追踪其中一层，无法同时回答“这张图来自哪个服务商”和“该服务商下哪个消费者生成了它”。

已有模型水印方法通常把同一个 fingerprint 固化进模型，使所有生成图像都携带同一来源标识。这适合开发者追踪模型泄露，但无法区分消费者。已有 in-generation image watermarking 能给每张图嵌入用户级水印，但默认模型拥有者和服务提供方是同一主体；当模型是由开发者授权给第三方服务商时，它又缺少 business-level 归因。FI-LoRA 的目标就是把这两类能力合并成一个层级化框架。

论文的关键 insight 是：LoRA 具有可插拔、低秩、易合并的工程属性，适合把“不同角色控制的指纹”写进模型权重中。作者进一步把 LoRA 改成 fingerprint-informed 形式，让二进制指纹通过小型编码网络生成中间矩阵 $Z$，再参与低秩权重更新；用 2 个 head 分别承载 business fingerprint 和 consumer fingerprint，使两个指纹共存且可独立解码。

## 2. 威胁模型解读

系统中有三类角色。开发者拥有原始 T2I 模型，负责训练 FI-LoRA、分配 business-level fingerprint $m_b^i$，并维护服务商指纹数据库。服务提供商拿到授权模型或服务模块，负责给其消费者分配 consumer-level fingerprint $m_c^{j|i}$，并维护消费者指纹数据库。消费者只能调用服务生成图像，理论上不控制任何指纹。

攻击目标分两类。服务提供商层面的恶意行为包括未经授权部署、泄露或修改开发者模型；消费者层面的恶意行为包括生成有害内容后试图否认来源。归因时，开发者希望用 business decoder $W_b$ 提取 $m_b$ 判断服务商来源；服务商或开发者希望用 consumer decoder $W_c$ 提取 $m_c$ 判断具体消费者。

攻击者能力也分层。消费者能对生成图像做 image-level fingerprint removal，例如 cropping、JPEG compression、horizontal flip、Gaussian blur、Gaussian noise、brightness enhancement 等。服务商拥有更强的 white-box model access，可以对模型做 model-level modification，例如 weight pruning、quantization、weight noise addition，也可以继续 fine-tune 模型以削弱水印。论文还讨论 business collusion attack，即多个服务商合谋平均/组合模型试图去除业务级指纹。

防御者能力是开发者能在 VAE decoder 中插入并训练 2-head FI-LoRA，能训练两个独立 fingerprint decoder，并能把业务级 head 合并进授权模型，同时把消费者级 head 留给服务商在生成时使用。服务商可以控制消费者指纹 head，但不应知道或篡改业务级指纹 $m_b$。

这个 threat model 贴近“模型授权 + API 服务 + 下游用户”的商业部署。它的强假设是：开发者和服务商会按协议保存指纹数据库，且消费者不能直接绕过服务商调用未指纹化模型。论文验证了图像级和模型级攻击，但对强合谋攻击只给出 logit bias 作为追踪线索，并承认方法本身不直接鲁棒；对完全移除/重训 VAE decoder 或替换生成后处理链的强攻击也没有系统覆盖。

## 3. 方法设计与复现级理解

### 3.1 目标定义：层级控制、双指纹有效性、效率、保真和鲁棒

论文先把 dual fingerprinting 的需求拆成 5 个目标。

第一是 hierarchical control。开发者控制服务商集合 $B$ 的指纹：

$$
m_b^i
$$

其中 $i$ 表示第 $i$ 个 business。服务商只能控制自己消费者集合 $C_i$ 下的消费者指纹：

$$
m_c^{j|i}
$$

其中 $j$ 表示第 $i$ 个 business 下的第 $j$ 个 consumer。

第二是 effectiveness。生成图像 $x_{ij}$ 中的两个指纹要能被独立提取：

$$
P(F_b(x_{ij}),m_b^i)\ge \tau
$$

$$
P(F_c(x_{ij}),m_c^{j|i})\ge \tau
$$

$P$ 表示匹配 bit 数，$\tau$ 是判定阈值。也就是说，两个 decoder 不能互相干扰，也不能只能提取其中一个。

第三是 efficiency。开发者需要能为大量服务商快速初始化不同指纹模型，不能每换一个服务商就完整重训一遍生成模型。第四是 fidelity，带双指纹模型生成的图像要尽量接近无指纹模型：

$$
Q(x_{ij},x_{clean})\le \epsilon
$$

第五是 robustness。面对攻击集合 $A$，两个指纹中较弱的那个也应超过阈值：

$$
\mathbb{E}_{A}[\min \{P(m_b^i,W_b(A(x_{ij}))), P(m_c^{j|i},W_c(A(x_{ij})))
\}]\ge \tau
$$

这个目标设计决定了后续方法必须是双头、可插拔、可顺序授权，并且训练时要同时优化图像质量和两个指纹解码。

### 3.2 全局流程：开发者训练双头，服务商使用第二头

FI-LoRA 的 pipeline 可以按角色理解。

开发者阶段：开发者拿预训练 T2I 模型，主要在 LDM 的 VAE decoder $D$ 中插入 2-head FI-LoRA，并冻结 VAE encoder $E$。训练时随机采样图像 $x$、业务指纹 $m_b$、消费者指纹 $m_c$，让 VAE decoder 重建出带双指纹但视觉质量接近原图的 $x'$；同时训练两个 fingerprint decoder $W_b$ 与 $W_c$，分别预测两个指纹。

授权服务商阶段：开发者为每个 business 分配唯一 $m_b^i$，并将对应的业务级 FI-LoRA head 合并或绑定到授权模型中。消费者级 head 预留给服务商，服务商在消费者请求生成时输入对应 $m_c^{j|i}$。

在线生成阶段：消费者正常输入 prompt 或编辑指令；T2I 模型的 U-Net 正常去噪，得到 latent；最终 VAE decoder 在 FI-LoRA 权重作用下输出图像。生成图像同时携带 business fingerprint 和 consumer fingerprint。

取证阶段：给定可疑图像，开发者/服务商分别用 $W_b$ 和 $W_c$ 提取两个 32-bit 指纹，与数据库比对，实现层级归因。

### 3.3 普通 LoRA 到 FI-LoRA：用指纹控制低秩权重

标准 LoRA 把预训练权重 $W_0$ 的更新写成低秩形式：

$$
W_0+\Delta W=W_0+B\cdot A
$$

其中：

$$
A\in\mathbb{R}^{r\times k},\quad B\in\mathbb{R}^{d\times r},\quad r\ll d
$$

FI-LoRA 在 $B$ 和 $A$ 中间插入一个由指纹生成的矩阵：

$$
Z=f(m)
$$

其中：

$$
f:\{0,1\}^{d_m}\rightarrow \mathbb{R}^{r\times r}
$$

最终权重变为：

$$
W_m=W_0+\Delta W=W_0+BZA
$$

展开某个权重元素：

$$
\Delta W_{i,j}=\sum_{k=1}^{r}\sum_{l=1}^{r} B_{i,k}\cdot Z_{k,l}\cdot A_{l,j}
$$

这个展开式说明 $m$ 不是只控制少数参数，而是通过 $Z$ 与 $B,A$ 相乘，影响 $\Delta W$ 的每个位置。实际实现中，$Z$ 是可插拔的：换一个指纹 $m$，通过 $f(m)$ 得到不同 $Z$，即可得到不同低秩权重更新，而不必完整重训模型。

### 3.4 2-head FI-LoRA：业务指纹和消费者指纹分离

为了同时嵌入两层指纹，论文设计 2-head FI-LoRA。两个 head 共享同一个 $B$，但拥有各自的 $A_b,A_c$ 和 fingerprint encoding network $f_b,f_c$：

$$
W_0+\Delta W = W_0+Bf_b(m_b)A_b+Bf_c(m_c)A_c
$$

共享 $B$ 的好处是减少参数量并让两个 head 在同一低秩子空间内协调；分离 $A_b,A_c,f_b,f_c$ 则让两个指纹可以独立控制、独立解码。开发者可把：

$$
Bf_b(m_b)A_b
$$

集成到预训练权重中，用于 business-level attribution；同时保留：

$$
Bf_c(m_c)A_c
$$

给服务商做 consumer-level attribution。论文强调这种设计能防止服务商知道或篡改 $m_b$，因为服务商只需要使用消费者 head，不直接控制业务 head。

这里的关键不是“简单加两个 LoRA”，而是把两个指纹的控制权映射到不同角色：开发者控制两个 head 的训练和业务 head，服务商只控制消费者 head，消费者不控制任何 head。

### 3.5 训练对象：只微调 VAE decoder，不动 U-Net

FI-LoRA 插入到预训练 T2I 模型的 VAE decoder $D$ 中。训练时，VAE encoder $E$ 冻结，U-Net 不参与训练；输入图像 $x$ 先经冻结 encoder 得到 latent：

$$
E(x)
$$

然后 decoder 在由 $m_b,m_c$ 参数化的 2-head FI-LoRA 权重下重建图像：

$$
x'=D(E(x);m_b,m_c)
$$

这样设计有两个工程优势。第一，只改 VAE decoder，比重训整个 T2I 模型便宜。第二，推理阶段 U-Net 的文本生成能力不受指纹训练直接影响；去噪出的 latent 仍按原流程进入 decoder，只是最终像素空间被写入指纹。

同时，论文训练两个独立 fingerprint decoder $W_b,W_c$。两个 decoder 都以输出图像 $x'$ 为输入，分别预测业务指纹和消费者指纹。为增强鲁棒性，训练时在 decoder 前加入 attack simulation layer，对图像做 JPEG、noise、blur 等增强，让 $W_b,W_c$ 学会从失真图像中恢复指纹。

### 3.6 图像重建损失：控制可见质量退化

图像质量损失由 MSE 和 LPIPS 组成：

$$
L_i=
\mathbb{E}_{x\sim X,\ m_b,m_c\sim\{0,1\}^{d_m}}
[
\lambda_1\|x-D(E(x);m_b,m_c)\|_2^2
+
\lambda_2 LPIPS(x,D(E(x);m_b,m_c))
]
$$

MSE 约束像素级重建，LPIPS 约束感知相似性。论文后续解释 FI-LoRA 在 FID 上表现较好，与训练中给 LPIPS 更高权重有关。这个损失保证 VAE decoder 学到“在图像里写入指纹”的同时，不把图像内容明显改坏。

### 3.7 指纹解码损失：两个 decoder 同时训练

指纹损失采用 bit-wise binary cross entropy。攻击模拟层记为 $A$，对输出图像 $x'$ 做增强后送入两个 decoder：

$$
W_b(A(x')),\quad W_c(A(x'))
$$

论文公式写成平均逐 bit BCE，可整理为：

$$
L_m=
\mathbb{E}_{x,m_b,m_c}
\sum_{i=1}^{d_m}
\left[
m_b^i\log\sigma(W_b(A(x')))_i)
+
(1-m_b^i)\log(1-\sigma(W_b(A(x')))_i)
\right.
$$

$$
\left.
+
m_c^i\log\sigma(W_c(A(x')))_i)
+
(1-m_c^i)\log(1-\sigma(W_c(A(x')))_i)
\right]
$$

严格来说，常规 BCE 优化通常最小化负对数似然；论文公式少了负号，但文字说明是计算 BCE loss。复现时应按标准 BCEWithLogitsLoss 或等价负号实现，不能直接最大化上式。

总损失为：

$$
L=L_i+\lambda_3L_m
$$

$\lambda_1,\lambda_2,\lambda_3$ 平衡图像重建和指纹可解码性。论文未在正文给出具体 $\lambda$ 数值，这是复现时需要从代码或补充材料确认的关键细节。

### 3.8 推理与归因流程：标准生成，不额外输入给用户

推理阶段，消费者不需要知道指纹，也不需要额外输入水印图。T2I 或 I2I 模型按原本流程生成/编辑图像，差异只在 VAE decoder 中的 FI-LoRA 权重已经由业务指纹和消费者指纹参数化。

生成后，图像中隐含两个 32-bit 指纹。取证时，给定可疑图像 $x'$，分别计算：

$$
\hat{m}_b=W_b(x')
$$

$$
\hat{m}_c=W_c(x')
$$

再用 bit matching 判断是否超过阈值 $\tau$。论文默认每个指纹 32 bit，总 payload 是 64 bit；在 FPR 实验中，用 $\tau=28$ 并在 32,000 张图像上评估，所有方法报告 0.0% FPR。

### 3.9 复现配置表

| 项目 | 论文配置 |
|---|---|
| 任务 | T2I generation 与 I2I text-guided image editing |
| 模型 | Stable Diffusion SD2-base、SD2-inpainting、InstructPix2Pix |
| 微调位置 | LDM 的 VAE decoder $D$；VAE encoder $E$ 冻结；U-Net 不参与训练 |
| FI-LoRA | 2-head；共享 $B$，独立 $A_b,A_c,f_b,f_c$ |
| fingerprint encoder | $f_b,f_c$ 为 2-layer fully connected network |
| fingerprint decoder | EfficientNet-b0 |
| 指纹长度 | $m_b$ 32 bit，$m_c$ 32 bit，总 payload 64 bit |
| 训练数据 | MS-COCO-2017 train set 微调 LDM decoder |
| 测试数据 | MS-COCO-2017 val、ImageNet、MagicBrush、InstructPix2Pix |
| 图像尺寸 | 随机裁剪到 $512\times512$ |
| 质量指标 | PSNR、SSIM、LPIPS、FID、CLIP Score |
| 指纹指标 | Bit Acc、FPR、TPR；阈值 $\tau=28$ |
| 指标样本量 | 除 FID 用 10k 外，其余基于 3,200 image pairs |
| FPR 测试 | 32,000 images |
| image-level attack | crop、JPEG、horizontal flip、Gaussian blur、Gaussian noise、brightness enhancement |
| model-level attack | weight pruning、quantization、Gaussian weight noise、adversarial fine-tuning |
| 对比方法 | Per. Norm、Stable Signature、WOUAF |

### 3.10 复现风险与信息缺口

论文没有在正文列出 LoRA rank $r$、$\lambda_1,\lambda_2,\lambda_3$、batch size、learning rate、训练 epoch、optimizer、attack simulation layer 的采样概率和强度范围、每个 VAE layer 的插入位置、训练随机种子等细节。代码已公开，因此复现应优先对照仓库配置。

另一个细节是公式 (6) 的符号。正文称其为 BCE loss，但写法像 log likelihood，缺少负号。实现时应按标准最小化 BCE 的方向处理。最后，business collusion attack 部分承认方法不直接鲁棒，只能从 pirated model 的 fingerprint logits bias 中获得 traceability evidence；这不是完整的合谋防护。

## 4. 与其他方法对比

| 方法类别 | 代表方法 | 核心思想 | 优点 | 局限 | FI-LoRA 改进 |
|---|---|---|---|---|---|
| 单层模型指纹 | Stable Signature、WOUAF、Per. Norm | 模型生成图像携带固定指纹 | 可追踪模型/服务商来源 | 所有消费者共享同一指纹，无法追踪具体用户 | 同时嵌入 business 和 consumer 两层指纹 |
| in-generation 图像水印 | Tree-ring、Gaussian Shading、ProMark 等 | 在生成输入或中间特征中加入水印 | 可按图像/用户动态嵌入 | 往往假设模型拥有者和服务商是同一主体 | 支持授权生态中的层级归因 |
| adapter-based 水印 | WMAdapter 等 | 水印作为额外条件输入模块 | 灵活控制水印 | 推理时需要额外水印输入，流程更复杂 | FI-LoRA 合并后可按标准推理 |
| LoRA/weight modulation 指纹 | OmniMark、WOUAF | 通过权重调制编码身份 | 可扩展到多个模型副本 | 多为单层 attribution | 2-head 结构把控制权分配给不同角色 |
| FI-LoRA | 本文 | 指纹生成 $Z=f(m)$，控制低秩更新 $BZA$ | 双层指纹、可插拔、低开销、支持白盒模型攻击测试 | 容量有限，合谋防护不足 | 解决多角色 GenAI 生态的层级追踪 |

FI-LoRA 的本质创新是“层级归因结构”而不是单纯 LoRA 水印。它把业务级和消费者级指纹写进同一个 VAE decoder 输出过程，使两个指纹共存，并通过两个 decoder 独立提取。

## 5. 实验表现与优势

图像质量实验覆盖 COCO、ImageNet、MagicBrush、InstructPix2Pix 四个数据集/任务。FI-LoRA 在重建和生成质量上整体优于或接近 baseline。以 FID 为例，COCO T2I 中 No fingerprint 为 23.265，Ours 为 23.272，几乎不变；ImageNet T2I 中 No fingerprint 为 24.239，Ours 为 24.240；MagicBrush I2I 中 No fingerprint 为 19.922，Ours 为 19.979；InstructPix2Pix 中 Ours FID 为 12.953，甚至低于 No fingerprint 13.216。CLIP Score 基本不变，说明 prompt-image semantic alignment 没明显受损。

指纹有效性实验显示，FI-LoRA 的两个指纹都能高精度提取。每个指纹 32 bit，总 payload 64 bit；相比 baseline 的 32 bit，负载翻倍。COCO 上 $m_b$ Bit Acc 为 99.96，$m_c$ 为 99.91；ImageNet 上分别 99.93 和 99.91；MagicBrush 上分别 99.94 和 99.95；InstructPix2Pix 上分别 99.90 和 99.94。阈值 $\tau=28$ 下，32,000 张图像的 FPR 为 0.0%。

图像级鲁棒性实验包括 crop、JPEG、flip、blur、noise、brightness。加入 attack simulation augmentation 后，鲁棒性明显提升。例如 JPEG compression 或高强度 Gaussian noise（Std=0.2）下，未增强版本 Bit Acc 可接近随机，而增强版能接近 100%。强 brightness attack 可把 Bit Acc 降到 80% 以下，但此时 PSNR 低于 10，图像质量已严重破坏；这说明去除指纹需要付出可见质量代价。

模型级白盒攻击实验包括 pruning、quantization、weight noise 和 adversarial fine-tuning。FI-LoRA 对 WOUAF 和 Stable Signature 的优势明显，例如模型权重 Gaussian noise std=0.02 时，Ours Bit Acc 仍接近 100%，而 WOUAF/Sta. Sig. 降到 70% 以下。极强攻击如 50% pruning 或 std=0.05 weight noise 下，FI-LoRA 略低于 Per. Norm 约 5%，但此时生成模型功能显著退化，PSNR 低于 15。

对抗微调实验中，攻击者只用 image loss 继续 fine-tune 模型。所有方法随步数增加都会下降，10k steps 后 baseline 多数低于 70%；FI-LoRA 虽下降但趋于稳定，10k steps 仍高于 80%，比对比方法高 10% 以上。该实验是对近期“Stable Signature 可被 fine-tuning 移除”这类攻击的回应。

消融/限制方面，论文明确指出容量是瓶颈。由于图像中包含两个指纹，每层有效 bit 是总容量的一半。单个 64-bit 指纹时 fingerprinting loss 可能不收敛；使用两个 32-bit 指纹则可兼顾层级归因和可靠性。因此 FI-LoRA 适合中等长度身份标识，不适合在当前设置下直接扩展到很高 payload。

## 6. 学习与应用

这篇适合放入 `Other_Security`，因为它是图像 T2I 模型的层级指纹/归因，不是语音或音频水印。它对你的整理有两个参考价值：第一，可作为生成式模型水印的跨模态对照；第二，它提出的“多角色层级追踪”在语音 TTS 服务生态中也有迁移意义，例如模型开发者追踪服务商、服务商追踪终端用户。

复现最小版本可以先只在 Stable Diffusion 的 VAE decoder 上插入 1-head FI-LoRA，验证单指纹能否被 EfficientNet-b0 decoder 提取；再扩展到 2-head，共享 $B$、分离 $A_b,A_c,f_b,f_c$；最后加入 attack simulation layer 和模型级攻击评估。不要一开始就做完整 T2I/I2I 四数据集评估，否则排错成本会很高。

实现层面最需要确认的是 LoRA rank、插入层位置、loss 权重和攻击增强策略。由于论文给出 GitHub 代码，建议以代码配置为准补全正文缺失超参。

## 7. 总结

用双头 LoRA 做层级图像归因。

## 8. 图表精读与证据链

图 1 是问题定位图。它比较单层模型指纹、单层图像水印和 FI-LoRA 的能力，说明现有方法只能追踪一个角色，而本文要同时追踪 business 和 consumer。

图 2 是系统 pipeline。它明确开发者先注册 business fingerprint，服务商再注册 consumer fingerprint，最后通过两个 decoder 分别验证来源。该图证明本文不是简单把两个水印叠加，而是按授权链条组织控制权。

图 3 是训练流程图。冻结 VAE encoder，把 2-head FI-LoRA 插入 VAE decoder，并用两个指纹 decoder 监督输出图像。它说明 U-Net 不参与训练，推理时生成能力主要由原 T2I 模型保持。

表 1 支撑 fidelity claim。Ours 在 PSNR/SSIM/LPIPS/FID/CLIP 上整体保持或优于 baseline，尤其 FID 接近 no fingerprint，说明双指纹没有明显破坏图像质量。

表 2 支撑 effectiveness claim。两个指纹在四个数据集/任务上都超过 99.9% Bit Acc，FPR 为 0.0%，同时在 crop/JPEG 下保持较高准确率。它是“dual fingerprints coexist and independently extractable”的核心证据。

图 5 支撑 image-level robustness。攻击强度增加时 Bit Acc 下降，但增强训练显著提升鲁棒性；高强度攻击需要牺牲 PSNR，说明指纹与图像内容有一定绑定。

图 6 支撑 model-level robustness。FI-LoRA 在 pruning、quantization、weight noise、adversarial fine-tuning 下整体优于 WOUAF 和 Stable Signature，说明 LoRA-merged 权重并不容易被简单白盒修改移除。

弱证据点是 collusion attack。论文承认方法不直接鲁棒，只说 pirated model 的 fingerprint logits bias 可提供追踪证据；这部分还不足以支撑强合谋场景下的完整安全性。

## 9. 复现难度与适合人群

复现难度：中高。单看 FI-LoRA 公式不复杂，但完整复现需要改 Stable Diffusion/InstructPix2Pix 的 VAE decoder，训练 EfficientNet-b0 指纹 decoder，跑 T2I/I2I 四数据集，复现 image-level 和 model-level 攻击。

主要依赖包括 MS-COCO-2017、ImageNet、MagicBrush、InstructPix2Pix、Stable Diffusion SD2-base、SD2-inpainting、InstructPix2Pix 权重、EfficientNet-b0、LoRA 插入代码、FID/CLIP/LPIPS 评估工具和 512×512 图像训练资源。

适合生成模型水印、模型指纹、AIGC 归因、图像取证、T2I 服务安全和 LoRA 安全研究者阅读。若是语音方向读者，重点看层级归因架构和双头权重调制思想，而不是图像指标本身。

## 10. 简短全面总结

FI-LoRA 面向商业化 GenAI 的多层责任链，解决已有水印只能追踪模型提供方或消费者单一层级的问题。方法在 T2I 模型的 VAE decoder 中插入 2-head Fingerprint-Informed LoRA：每个 head 用二进制指纹经小型网络生成矩阵 $Z=f(m)$，再形成低秩更新 $BZA$；业务 head 由开发者控制，用于追踪服务提供商，消费者 head 留给服务商在生成时控制，用于追踪终端用户。训练时冻结 VAE encoder 和 U-Net，同时优化 MSE/LPIPS 图像重建损失与两个指纹 decoder 的 bit-wise BCE，并通过 attack simulation layer 提升图像后处理鲁棒性。实验覆盖 SD2-base、SD2-inpainting、InstructPix2Pix、COCO、ImageNet、MagicBrush 和 InstructPix2Pix 数据集，两个 32-bit 指纹均达到 99.9% 以上 Bit Acc，并在图像级与模型级攻击下保持较强鲁棒。主要限制是容量有限，单个 64-bit 指纹可能不收敛，且对服务商合谋攻击只提供有限证据。

## 11. 论文写作逻辑分析

论文 intro 的问题铺垫比较清楚：先说明 GenAI 商业化形成 developer-business-consumer 三层生态，再指出责任风险也分层，最后批评已有 model fingerprinting 和 image watermarking 各自只能追踪一层。这使“dual fingerprinting”不是为了堆功能，而是由现实角色链条自然推出。

方法叙事按目标到实现展开。Section 3.1 先列 hierarchical control、effectiveness、efficiency、fidelity、robustness 五个目标；Section 3.2 再给出 FI-LoRA。这样的结构让读者能判断每个设计是否回应目标：LoRA 对应 efficiency，2-head 对应 hierarchical control，两个 decoder 对应 effectiveness，LPIPS/MSE 对应 fidelity，attack simulation 和模型攻击评估对应 robustness。

实验组织也与目标对应：表 1 验证 fidelity，表 2 验证 dual extraction 和 FPR，图 5 验证 image-level robustness，图 6 验证 white-box model-level robustness。证据链最强的是双指纹 99.9%+ 准确率和四任务质量保持；较弱的是 collusion attack 和容量扩展，只在限制中简要说明。可借鉴的写法是把“多角色控制权”写成方法目标，而不是只把它作为应用场景背景。
