# Spherical Watermark: Encryption-Free, Lossless Watermarking for Diffusion Models

## 0. 摘要翻译

扩散模型已经革新了图像合成，但也带来了内容来源和真实性方面的担忧。数字水印可以用于追踪生成媒体，但传统方案往往引入分布偏移并降低视觉质量。近期的无损方法直接把水印比特嵌入 latent Gaussian prior，且不修改模型权重，但仍需要为每张图像存储密钥，或依赖较重的密码学开销。本文提出 Spherical Watermark，一个免加密、无损的扩散模型水印框架，可无缝接入扩散架构。首先，二进制嵌入模块将重复水印比特与随机 padding 混合，形成高熵编码。其次，球面映射模块将该编码投影到单位球面，施加正交旋转，并用服从卡方分布的半径进行缩放，从而恢复精确的多元 Gaussian noise。作者从理论上证明，水印噪声分布在三阶矩以内保持目标 prior，并通过实验表明它与标准多元正态分布在统计上不可区分。基于 Stable Diffusion 的大量实验确认，Spherical Watermark 在保持高视觉质量的同时提升可追踪性、计算效率和攻击鲁棒性，优于有损与无损水印方法。

## 1. 方法动机

这篇论文解决的是 text-to-image 扩散模型的生成内容溯源问题。平台或模型开发者希望给每个 API 用户生成的图像嵌入可追踪身份，例如 user ID、timestamp；后续发现恶意图像时，可以从图像中恢复水印，定位来源用户。

已有路线主要有三类问题。第一，传统空间域、频域或神经网络后处理水印会修改图像分布，影响 FID 或被分类器检测出来；在强攻击下，水印特征还可能成为 adversarial evasion 的靶子。第二，训练式或微调式生成水印需要改模型权重，部署成本高，也不灵活。第三，近期 lossless watermark 能把比特映射到 Gaussian prior，但 Gaussian Shading 依赖每张图像独立 key/nonce，PRC Watermark 又引入复杂的伪随机纠错编码和 belief propagation decoding，带来存储、调参和延迟开销。

Spherical Watermark 的核心直觉是：只要能把用户水印比特可逆地映射成看起来仍然是标准 Gaussian 的初始噪声，就可以不动 Stable Diffusion 本身，并在不损失分布质量的情况下完成追踪。作者用“二进制高熵混合 + 球面 3-design + 正交旋转 + 卡方半径缩放”实现这个映射，目标是同时满足 losslessness、traceability 和低计算开销。

## 2. 威胁模型解读

论文从模型开发者/API 平台视角建模。开发者持有预训练扩散生成器 $G$、VAE encoder/decoder、固定签名 $K=(T,C)$，并在离线阶段构造可逆变换；在线阶段，API 为每个用户请求嵌入不同二进制水印 $m$，生成带水印图像 $O_w$；追踪阶段，开发者对嫌疑图像做 inversion，恢复 latent 并解码 $m$。

攻击者可以是恶意用户或下游传播者，目标是生成或传播恶意图像后逃避溯源。攻击方式包括常规图像后处理，例如 Gaussian noise、brightness、drop、Gaussian blur、JPEG、median filter、resize；也包括 adversarial attack WEvade，论文分别设置白盒与黑盒版本，并取平均结果。附录还讨论 re-generation 和 editing attack，但主实验的核心仍是 storage/transmission perturbation 与 WEvade。

攻击者知识方面，论文主要验证的是检测器可被训练区分时的风险，以及攻击者可利用 WEvade 规避水印检测的设定。签名 $K=(T,C)$ 被开发者保密；论文没有把“攻击者知道 $T,C$ 并直接优化 latent 或构造密钥恢复攻击”作为主 threat model，也没有评估攻击者拥有完整生成服务内部状态并修改初始噪声采样流程的情况。

防御目标有两个。第一是 undetectability/losslessness：水印 latent $z_w$ 与标准 Gaussian $z$ 对任意多项式时间判别器不可区分，生成图像分布也不可区分。第二是 traceability/exact extraction：从带水印图像反演出的 latent 中应能以高概率恢复原水印。

这个 threat model 对“平台给自己生成的图像做用户级追踪”是合理的，因为平台确实能控制采样初始噪声并保存固定签名；但它不适合“给外部已有图像补水印”，也不能证明面对泄露签名或强白盒 adaptive latent attack 仍安全。方法和实验都围绕这个边界展开：方法保证 Gaussian prior 形态，实验检验分布不可区分、常规攻击和 WEvade 下的追踪准确率。

## 3. 方法设计与复现级理解

### 3.1 全局流程：离线签名、在线嵌入、事后提取

Spherical Watermark 的流程分三段。

离线 build phase 中，开发者生成固定签名：

$$
K=(T,C)
$$

其中 $T$ 是二进制域上的可逆 embedding matrix，用于把重复水印和随机 padding 混合；$C$ 是实数域正交 rotation matrix，用于把离散二进制点旋转到更均匀的球面位置。

在线 runtime phase 中，API 得到用户水印 $m$ 后，先重复水印、拼接随机 padding，形成二进制向量 $x$；再经 $B$ 模块得到高熵二进制码 $z^{(1)}$；经 $S$ 模块映射到球面并缩放为 Gaussian-like noise $z_w$；最后把 $z_w$ 作为 Stable Diffusion 的初始噪声，生成带水印图像 $O_w$。

提取阶段中，开发者用 VAE encoder 和 DDIM/ODE inversion 从嫌疑图像 $\hat{O}_w$ 估计初始噪声 $\hat{z}_T$，再依次应用 $S^{-1}$ 与 $B^{-1}$，通过 majority vote 恢复水印 $\hat{m}$。

### 3.2 问题定义：Embed/Extract 的两个要求

固定预训练扩散生成器：

$$
G:\mathbb{R}^{l_x}\rightarrow I
$$

它把标准 Gaussian noise $z$ 映射为生成图像 $O$。扩散模型可近似反演，因此用 $G^{-1}$ 从生成图像恢复 latent 表示。水印长度为 $l_m$，论文要设计两个 latent-space 过程：

$$
Embed:m\in\{0,1\}^{l_m}\rightarrow z_w\in\mathbb{R}^{l_x}
$$

$$
Extract:\hat{z}_w\in\mathbb{R}^{l_x}\rightarrow \hat{m}\in\{0,1\}^{l_m}
$$

无损/不可检测要求是，对于任意多项式时间攻击者 $A$：

$$
\left|
Pr[A(z_w)=1]-Pr[A(z)=1]
\right|
\le negl(\rho)
$$

进一步，生成图像也应不可区分：

$$
\left|
Pr[A'(G(z_w))=1]-Pr[A'(G(z))=1]
\right|
\le negl(\rho)
$$

可追踪要求是：

$$
Pr[Extract(G^{-1}(O_w))=m]\ge 1-negl(\rho)
$$

这里的关键不是“水印肉眼不可见”，而是“水印噪声分布不偏离扩散模型原本的标准 Gaussian prior”。只要 $z_w$ 的分布仍然像 $N(0,I)$，图像质量就不会因为水印嵌入而出现系统性退化。

### 3.3 Watermark preprocessing：重复水印 + 随机 padding

论文把水印 $m$ 看成独立 Bernoulli 比特。为提升冗余和纠错能力，先把 $m$ 重复 $N$ 次，再追加随机 padding：

$$
r\in\{0,1\}^{l_r},\quad r_i\sim Bernoulli(1/2)
$$

拼接向量为：

$$
x=[m\ m\ \cdots\ m\ r]^\top\in\{0,1\}^{l_x}
$$

并满足：

$$
l_x=N\times l_m+l_r
$$

重复 $m$ 的作用是后续 majority vote；padding 的作用是打破重复水印带来的相关性。如果只重复水印而不引入随机 padding，二进制序列会有明显结构，后续即使映射到 Gaussian-like noise，也可能在高阶统计或分类器上被检测出来。

### 3.4 Binary embedding module：用可逆矩阵混合水印和 padding

二进制嵌入矩阵定义在 $\mathbb{F}_2$ 上：

$$
T=
\begin{bmatrix}
I_{l_{Nm}} & R\\
0 & I_{l_r}
\end{bmatrix},
\quad l_{Nm}=N\times l_m
$$

其中 $R\in\{0,1\}^{l_{Nm}\times l_r}$ 是稀疏二进制矩阵。每个水印 bit 的每个重复副本会与 $s$ 个 padding bit 混合；Algorithm 1 对每个原始 bit $j$ 生成随机排列 $\pi$，取前 $N\times s$ 个 padding 位置，并给 $N$ 个重复副本分配互不重叠的 padding 子集。

嵌入操作为：

$$
z^{(1)}=Tx
$$

所有运算都在 $\mathbb{F}_2$ 中进行。因为 $T$ 是上三角块矩阵且对角块为 identity，所以它可逆，并且论文给出：

$$
T^{-1}=T
$$

这一步解决的是“重复水印太有结构”的问题。$R$ 把 padding 的随机性注入水印副本，使 $z^{(1)}$ 的每个 bit 边际上仍是 Bernoulli(1/2)，并达到 2-wise 和 3-wise independence。参数 $s$ 控制混合强度：$s$ 越大，随机性更强，但反演出错时一个 padding 错误会影响更多水印副本，错误传播也更严重。

### 3.5 Spherical mapping module：二进制码到 Gaussian noise

得到 $z^{(1)}\in\{0,1\}^{l_x}$ 后，论文先把 bit 映射到 $\{-1,+1\}$：

$$
v=2z^{(1)}-1
$$

再归一化到单位球面：

$$
z^{(2)}=\frac{v}{\|v\|_2}
$$

由于 $v$ 的每个元素为 $\pm 1$，归一化后每个坐标取值：

$$
z_i^{(2)}\in\left\{-\frac{1}{\sqrt{l_x}},+\frac{1}{\sqrt{l_x}}\right\}
$$

接着施加正交旋转：

$$
z^{(3)}=Cz^{(2)}
$$

$C$ 通过采样 i.i.d. Gaussian 矩阵后做 QR decomposition 得到，并保留正交因子。理论上 $C^{-1}=C^\top$。正交旋转不会改变向量长度，但会把离散超立方体顶点的坐标方向打散，改善鲁棒性和统计均匀性。

最后采样半径 $r$，使：

$$
r^2\sim\chi^2(l_x)
$$

并缩放得到水印噪声：

$$
z_w=rz^{(3)}
$$

这一步对应多元 Gaussian 的极坐标分解：标准正态向量可以写成“卡方半径 × 球面均匀方向”。论文不能证明 $z^{(2)}$ 是完整球面均匀分布，但通过 3-wise independence 证明它形成 spherical 3-design，即最多三阶矩与球面均匀分布一致；正交旋转保持这个性质，卡方缩放后得到与 $N(0,I)$ 高度匹配的 latent noise。

### 3.6 Diffusion integration：把 $z_w$ 当作 Stable Diffusion 初始噪声

Stable Diffusion 使用 VAE 在图像空间和 latent 空间之间映射：

$$
E_{VAE}:I\rightarrow\mathbb{R}^{l_x},\quad
D_{VAE}:\mathbb{R}^{l_x}\rightarrow I
$$

扩散采样从初始噪声 $z_T$ 逐步去噪到 $z_0$：

$$
z_T\rightarrow z_{T-1}\rightarrow\cdots\rightarrow z_0
$$

概率流 ODE 写作：

$$
\frac{dz_t}{dt}
=
f_t(z_t)-\frac{1}{2}g_t^2\nabla_{z_t}\log p_t(z_t)
$$

score 由网络 $s_\theta(z_t,t)$ 近似。嵌入时，Spherical Watermark 直接设：

$$
z_T=z_w
$$

然后用 ODE solver 从 $t=T$ 积分到 $t=0$：

$$
z_0=ODESolve(z_T;s_\theta,cond,T,0)
$$

最后由 VAE decoder 生成图像：

$$
O_w=D_{VAE}(z_0)
$$

这一步没有训练 Stable Diffusion，也不改权重。只要 $z_w$ 的分布近似标准 Gaussian，生成模型就会像普通采样一样工作。论文实验中用 50-step DPM-Solver++ 生成图像，guidance scale 为 7.5。

### 3.7 Watermark extraction：反演初始噪声并逐级逆变换

给定嫌疑图像 $\hat{O}_w$，开发者先用 VAE encoder 得到：

$$
\hat{z}_0=E_{VAE}(\hat{O}_w)
$$

再用 DDIM inversion / ODE solve 从 $t=0$ 反推到 $t=T$：

$$
\hat{z}_T=ODESolve(\hat{z}_0;s_\theta,\emptyset,0,T)
$$

论文为了模拟真实场景，inversion 使用 empty prompt，而不是原始生成 prompt。这是一个重要设定：检测不依赖知道用户当时输入的文本。

之后执行逆球面映射和逆二进制嵌入：

$$
\hat{z}^{(2)}=C^{-1}\hat{z}_T
$$

$$
\hat{z}^{(1)}=
round\left(\frac{\hat{z}^{(2)}+1}{2}\right)
$$

$$
\hat{x}=T^{-1}\hat{z}^{(1)}
$$

$\hat{x}$ 前 $l_{Nm}$ 个位置对应 $N$ 份重复水印。对每个原始 bit，跨 $N$ 个副本做 majority vote 得到 $\hat{m}$。为了避免平票，$N$ 取奇数，默认 $N=31$。

### 3.8 理论保证：3-wise independence、spherical 3-design 与 Gaussian prior

Theorem 3.1 证明：若 $m$ 和 $r$ 都是独立 Bernoulli(1/2)，且 $R$ 按 Algorithm 1 构造，那么 $z^{(1)}$ 中每个坐标都是 Bernoulli(1/2)，并且 $z^{(1)}$ 具有 2-wise 和 3-wise independence。

Theorem 3.2 进一步证明：$z^{(2)}$ 位于单位球面 $S^{l_x-1}$，每个坐标取 $\pm1/\sqrt{l_x}$，且具有 2-wise/3-wise independence，因此有限点集构成 spherical 3-design。直观上，它不是完整均匀球面，但对三阶以内多项式统计量来说与均匀球面一致。

Lemma 3.3 说明固定正交旋转 $C$ 不破坏 spherical 3-design；坐标满足：

$$
\mathbb{E}[z_i]=0,\quad \mathbb{E}[z_i^2]=\frac{1}{l_x}
$$

当 $l_x\rightarrow\infty$，坐标边际趋近：

$$
z_i^{(3)}\Rightarrow N(0,1/l_x)
$$

Lemma 3.4 则使用 Gaussian 极坐标分解：若 $r^2\sim\chi^2(n)$，$u$ 在单位球面上均匀且 $r\perp u$，则：

$$
n=r\cdot u\sim N(0,I_n)
$$

因此，论文理论链条是：二进制高熵化保证低阶独立性，球面归一化得到 3-design，正交旋转保持球面性质，卡方半径缩放恢复 Gaussian prior。严格边界是：作者承认保证依赖 spherical 3-design，因此主要保持到三阶矩；高阶矩可能偏离 true prior。

### 3.9 复现配置表

| 项目 | 论文配置 |
|---|---|
| 任务 | text-to-image diffusion watermark / provenance tracing |
| 模型 | Stable Diffusion v1.5、Stable Diffusion v2.1 |
| 图像尺寸 | $512\times512$ 彩色图像 |
| latent 尺寸 | $4\times64\times64$，即 $l_x=16384$ |
| 生成采样 | 50-step DPM-Solver++，guidance scale 7.5 |
| 反演 | 50-step DDIM inversion，guidance scale 1.0，empty prompt |
| 默认参数 | $N=31,\ l_m=512,\ l_r=512,\ s=1$ |
| 水印容量 | 默认 512-bit；容量实验到约 4000 |
| 数据集 | COCO：MS-COCO val2017 随机 1000 prompts；SDP：Stable Diffusion Prompt dataset 随机 1000 prompts |
| 用户数 | tracing evaluation 中 100 distinct users |
| baseline | DwtDct、DwtDctSvd、RivaGAN、Tree-Ring、Gaussian Shading、PRC Watermark |
| 传统水印容量 | DwtDct/DwtDctSvd/RivaGAN 配置为 32-bit |
| latent 水印容量 | 除 Tree-Ring 仅支持 detection 外，其余默认 512-bit |
| 重复次数 | 所有指标报告 5 次运行均值和标准差 |
| 硬件 | 4 × NVIDIA RTX 4090 |
| 指标 | FID、latent/image classifier accuracy、ACC、TPR@1%FPR、embedding/extraction time |
| 攻击 | PNG、Gaussian noise、brightness、drop、Gaussian blur、JPEG、median filter、resize、WEvade |

### 3.10 复现风险与信息缺口

论文主体给出了关键超参数、模型、采样器、数据集、攻击类型和硬件，但完整复现仍依赖 supplementary/source code。正文没有给出所有实现细节，例如 classifier 训练完整配置、WEvade 训练/查询细节、不同容量下 $N,l_m,l_r,s$ 的精确联动、rotation matrix 存储/分块实现、QR decomposition 的随机种子、以及 DPM-Solver++ 与 DDIM inversion 的具体库版本。

此外，论文 PDF 的 reproducibility statement 说 source code included in supplementary material，但正文没有提供公开 GitHub 链接；我当前检索也没有确认到官方公开仓库。因此 Overview 中不应写 Code 链接，只能写 Paper/OpenReview，或备注 Code: supplementary。

## 4. 与其他方法对比

| 方法类别 | 代表方法 | 核心思想 | 优点 | 局限 | Spherical Watermark 的改进 |
|---|---|---|---|---|---|
| 传统图像水印 | DwtDct、DwtDctSvd | 空间/频域嵌入 bit | 可用于已有图像 | 有损，分布偏移明显，强攻击下 TPR 崩溃 | 直接在扩散 latent prior 中无损嵌入 |
| 神经水印 | RivaGAN | 训练 encoder/decoder 嵌入水印 | 鲁棒性较传统方法强 | 仍是后处理水印，可被检测/对抗攻击利用 | 避免在最终图像中注入可学习 artifact |
| pattern-based latent watermark | Tree-Ring | latent 频域 ring pattern 检测 | 可做生成图像检测 | 主要检测 presence，不支持大规模 bitwise provenance；可被分类器检测 | 支持 512-bit 用户级追踪 |
| Gaussian Shading | lossless latent Gaussian sampling + stream cipher | 理论 lossless | 每图 key/nonce 管理重；固定 key 时不再真无损 | 免 per-image key/nonce，降低存储复杂度 |
| PRC Watermark | fixed-key pseudorandom error-correcting code | 避免每图 key，具备无损性 | belief propagation decoding 慢，容量升高和强失真下性能下降 | 提取快约四个数量级，高容量下更稳 |
| Spherical Watermark | binary embedding + spherical mapping + chi-square scaling | 无需改模型，近似 Gaussian prior，支持追踪 | 高阶矩仍可能偏离；依赖 inversion | 在质量、效率、鲁棒和可追踪性间折中更好 |

本文贡献点主要在映射设计：它不是用密码学伪随机序列采样 Gaussian，而是把水印二进制序列变成 spherical 3-design，再通过卡方半径恢复 Gaussian noise。这降低了每图 key 管理和复杂纠错解码开销。

## 5. 实验表现与优势

实验首先验证 undetectability。FID 表中，Ours 与 Original 基本一致，例如 COCO + SD v1.5 上 Original 为 48.1256，Ours 为 48.1224；COCO + SD v2.1 上 Original 为 46.8146，Ours 为 46.8132。PRC Watermark 也接近 Original，而 DwtDctSvd、RivaGAN、Tree-Ring、Gaussian Shading 普遍带来更明显 FID 偏移。

分类器检测实验更直接。latent-level 两层 MLP 能把 Tree-Ring 和 fixed-key Gaussian Shading 检到 100% 和 97% 左右；image-level ResNet-18 也能检测 Tree-Ring 和 Gaussian Shading。PRC Watermark 和 Spherical Watermark 接近 50% 随机猜测，说明至少在论文的分类器设定下没有明显可学习分布 artifact。

计算效率实验比较 latent-based 方案在 watermark 与 latent noise 间转换的时间，不包含扩散采样和反演。论文报告 PRC Watermark 提取时间比本文慢约四个数量级，原因是 PRC 需要 belief-propagation decoding；Spherical Watermark 只需要矩阵逆、正交逆旋转、rounding 和 majority vote。

追踪准确率主表以 COCO + SD v2.1 为例：Ours 在 Clean 下 ACC 99.99、TPR 100.00；Post-Processing 下 ACC 95.02、TPR 97.50；Adversarial 下 ACC 98.12、TPR 99.83。PRC Watermark 在同表中 Post-Processing TPR 为 87.03，明显低于 Ours；Gaussian Shading 在 adversarial 下 TPR 99.23，但 post-processing ACC 98.43、TPR 99.97，整体很强，只是其固定 key 设置不保持 true losslessness，且有 key/nonce 管理问题。传统有损方法在 adversarial 下 TPR 大幅下降，例如 DwtDct/DwtDctSvd/RivaGAN 的 Adv TPR 分别约为 16.15、17.05、26.75。

附录多表进一步覆盖 COCO/SDP 与 SD v1.5/v2.1。总体规律一致：Ours 在 Clean 和 Post-Processing 设置下 ACC/TPR 多数超过 95%，在 adversarial 设置下也保持接近 99% TPR；有损方法在 adversarial 下显著崩溃；PRC 在 post-processing TPR 上明显弱于 Ours。

消融实验显示两个模块都必要。去掉 binary embedding 后，latent noise 很容易被检测，说明重复水印必须与 padding 混合以获得独立性；去掉 spherical mapping、改用 Gaussian Shading transform 后，brightness 等攻击下鲁棒性大幅下降。参数消融显示，$s$ 增大会带来错误传播，$N$ 减小会削弱 majority vote 冗余；默认 $s=1,N=31$ 是鲁棒性较强的设置。

采样器消融显示，DDIM、PNDM、DPM-Solver++ 在 PNG、brightness、blur、median、JPEG、resize 下 extraction accuracy 差异很小；timesteps 从 10 到 50 变化也没有明显破坏提取。这说明方法对中等 inversion 误差有一定容忍度，但论文限制部分也承认，针对 VAE encoder 或 ODE solver 的极强 inversion-breaking attack 仍可能破坏恢复。

## 6. 学习与应用

这篇论文适合归到 `Other_Security`，因为它是图像扩散模型水印，不属于语音/音频安全主线。它可以作为你综述里“无损生成式水印 / diffusion provenance”方向的参考，用来和音频生成式水印中的 LAW、GROOT、RIWF 等形成跨模态对照。

若要实现最小版本，建议先不处理完整 Stable Diffusion inversion，而是在 latent 空间验证三件事：第一，按 $m,r,T$ 得到 $z^{(1)}$，确认 bit 边际和低阶独立性；第二，执行球面映射和卡方缩放，检验 $z_w$ 的均值、方差、低阶矩和分类不可区分性；第三，对 $z_w$ 加 Gaussian perturbation 后执行逆映射与 majority vote，验证恢复率。确认 latent-level 逻辑后，再接 Stable Diffusion 生成与 DDIM inversion。

迁移到其他生成模型时，必须满足两个条件：初始噪声 prior 是 Gaussian 或可被类似球面/半径分解建模；生成过程有可用近似逆映射。若是离散 token 生成、非 Gaussian prior 或 inversion 不稳定的模型，本文方法不能直接套用。

## 7. 总结

把水印映射成无损 Gaussian 噪声。

## 8. 图表精读与证据链

图 1 给出完整 pipeline。它把 offline signature build、online API watermark embedding 和 developer extraction 三个角色分开，明确这是平台侧追踪方案，而不是用户端后处理水印。

表 1 支撑视觉质量与分布保持。Ours 的 FID 与 Original 几乎一致，证明水印没有像传统方法那样造成明显分布偏移。该表也说明 PRC 同样有良好 FID，因此本文优势不只是质量，而是鲁棒性与效率。

图 2 支撑 undetectability。Tree-Ring 与 Gaussian Shading 可被 latent/image classifier 学到，而 PRC 与 Ours 接近随机。这个证据对应理论中的 indistinguishability，但仍只是有限分类器实验，不能等同于任意攻击者证明。

图 4 支撑计算效率。PRC Watermark 的 extraction 明显慢，Ours 快约四个数量级，直接回应论文对 heavy cryptographic overhead 的批评。

表 2 支撑主追踪性能。Ours 在 COCO + SD v2.1 的 Clean/Post/Adv 三类条件下保持高 ACC/TPR，尤其 adversarial TPR 99.83，说明 lossless 水印比传统有损水印更不容易被 WEvade 利用。

图 5 和图 6(a) 展示攻击强度与容量变化。PRC 在强失真和容量接近 2000 后下降明显，而 Ours 维持较高检测率。该证据支持“更适合高容量用户级 provenance”的 claim。

表 3、图 6(b-d)、附录图 14-17 是消融证据链。它们分别说明 $s,N$ 的取舍、binary embedding 和 spherical mapping 的必要性、参数变化对不可检测性的影响，以及 inversion 噪声容忍度。

## 9. 复现难度与适合人群

复现难度：中高。理论部分需要理解有限域线性变换、spherical design、Gaussian polar decomposition；工程部分需要 Stable Diffusion、DPM-Solver++、DDIM inversion、WEvade 和多种 baseline。若只复现 latent mapping，难度中等；若复现完整 ICLR 实验，需要 4 张 RTX 4090 级别算力和完整攻击/评测 pipeline。

主要依赖包括 Stable Diffusion v1.5/v2.1、MS-COCO val2017 prompt、Stable Diffusion Prompt dataset、DwtDct/DwtDctSvd/RivaGAN/Tree-Ring/Gaussian Shading/PRC baseline、FID 与分类器训练工具、WEvade 实现。适合研究 diffusion watermark、AIGC provenance、图像取证、生成模型安全和无损水印理论的读者。

## 10. 简短全面总结

Spherical Watermark 面向 text-to-image 扩散模型的生成内容溯源，目标是在不修改模型、不降低图像分布质量、不过度依赖 per-image key 的情况下，把用户水印嵌入初始 Gaussian latent。方法先重复水印并拼接随机 padding，再用可逆二进制矩阵 $T$ 混合，使编码具有低阶独立性；随后把二进制码映射到单位球面，施加正交旋转 $C$，再用卡方半径缩放，得到近似标准多元 Gaussian 的 $z_w$；生成时直接用 $z_w$ 作为 Stable Diffusion 初始噪声，提取时通过 VAE/DDIM inversion 反推 latent，并用 $C^{-1},T^{-1}$ 与 majority vote 恢复水印。实验覆盖 SD v1.5/v2.1、COCO/SDP、传统/latent baseline、常规后处理和 WEvade，显示本文在 FID、分类不可检测性、追踪 ACC/TPR、计算效率和高容量鲁棒性上有优势。主要边界是理论保证只到三阶矩，且强白盒 inversion-breaking 或签名泄露攻击未被充分覆盖。

## 11. 论文写作逻辑分析

论文开头从 AIGC provenance 需求切入，再把现有方法明确分成有损水印、训练式生成水印、lossless latent watermark 三类。这个问题铺垫比较有效，因为它不是泛泛说“水印很重要”，而是把痛点落到 distribution shift、per-image key storage、cryptographic overhead 和 decoding latency。

方法叙事遵循从安全定义到构造的顺序：先定义 undetectability 与 exact extraction，再写 binary embedding、spherical mapping、diffusion integration，最后用 theorem/lemma 解释为什么最终噪声接近 Gaussian。这样的组织方式适合理论型安全论文，读者能看到每个模块服务于哪个安全目标。

实验与 claim 对应较完整。质量 claim 用 FID 和样例图；不可检测 claim 用 latent/image classifier；效率 claim 用 runtime；鲁棒 claim 用 ACC/TPR under post-processing 和 WEvade；模块必要性用 ablation。证据链最强的是与 PRC 的效率和高容量对比，以及有损方法在 WEvade 下崩溃。相对弱的是高阶统计不可区分性和强 adaptive attacker，只在 discussion 中承认 higher-order moments 和 inversion-breaking 风险。

可借鉴的写法是：将“无损”写成可检验的分布不可区分目标，而不是只报告视觉质量；同时把数学构造与工程 pipeline 放在同一节，减少理论和实现脱节。需要谨慎的是，文章标题中的 encryption-free 容易被误解为完全无密钥；实际上仍有固定 secret signature $K=(T,C)$，只是避免 per-image key/nonce 和重型密码学编码。
