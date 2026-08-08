# DeepForgeSeal: Latent Space-Driven Semi-Fragile Watermarking for Deepfake Detection Using Adversarial Reinforcement Learning

## 0. 翻译摘要原文

生成式 AI 的快速发展使 deepfake 变得越来越逼真，对执法、公信力和媒体真实性带来持续挑战。现有被动 deepfake 检测器很难跟上新型伪造方式，主要原因是它们依赖特定伪造伪影，导致对新 deepfake 类型的泛化能力有限。基于水印的主动 deepfake 检测被提出用于识别高质量合成媒体，但这类方法通常难以同时做到：对良性失真保持鲁棒，对恶意篡改保持敏感。本文提出一种新的深度学习框架，利用高维 latent space 表示和 Adversarial Reinforcement Learning 范式，构建鲁棒且自适应的水印方法。具体而言，作者设计了一个在 latent space 中工作的可学习水印嵌入器，用于捕获图像高层语义，同时精确控制消息编码与提取。ARL 范式通过与一个 adversarial attacker agent 交互，使可学习水印模块在良性和恶意图像操作构成的动态 curriculum 中寻找鲁棒性与脆弱性之间的最优平衡。CelebA 和 CelebA-HQ 上的综合实验表明，该方法在困难篡改场景下持续优于现有 SOTA，在 CelebA 上提升超过 4.5%，在 CelebA-HQ 上提升超过 5.3%。

## 1. 方法动机

a) 作者提出 DeepForgeSeal 的背景是：deepfake 已经从低质量伪造发展到高保真、语义级编辑，传统被动检测方法依赖具体生成器留下的伪影，面对未知生成模型、换脸、属性编辑、匿名化等操作时泛化不稳定。因此论文选择主动防御路线：在真实图像进入传播链路前主动嵌入半脆弱水印，后续通过水印是否仍能恢复来判断图像是否被语义篡改。

b) 现有主动水印方法的主要不足是 resilience-fragility paradox。一个 deepfake 检测水印应该对 JPEG、裁剪、亮度变化、压缩、噪声等良性传播操作保持鲁棒，否则会产生大量误报；但面对换脸、表情/年龄/发色修改、人脸 reenactment 等语义篡改时，它又必须变脆弱，否则无法把恶意 deepfake 标出来。许多方法要么过于脆弱，对普通后处理也失效；要么过于鲁棒，对真正语义篡改仍能恢复水印，导致漏检。

c) 论文的核心 insight 是：水印不应只绑定低层像素，而应绑定到图像的高层语义 latent space。CLIP feature 位于单位超球面，适合用方向投影来编码消息。良性图像操作通常不会显著移动语义特征，因此水印应保持可恢复；恶意 deepfake 操作会改变身份、表情或语义属性，使 CLIP latent 沿语义方向明显漂移，从而破坏水印。ARL attacker agent 则用于不断生成更困难的攻击 curriculum，迫使水印器学习“对良性强、对恶意弱”的平衡点。

## 2. 威胁模型解读

a) 参与方与系统边界：防御者是水印系统部署者，能够在图像发布前运行 DeepForgeSeal，为 bonafide face image 嵌入水印。检测者持有共享密钥 $K$ 和训练好的 direction generator/extractor，用于在推理阶段从查询图像中恢复消息。攻击者处于传播链路或伪造链路中，可以对水印图像执行良性图像变换或恶意 deepfake 操作。

b) 攻击者目标：攻击者希望破坏水印提取，使真实水印图像被误判为篡改，或者让经过语义篡改的 deepfake 仍能恢复水印而被误判为真实。论文更关注第二类安全目标：在语义级恶意操作发生后，水印应主动失效，以便检测器将其标记为 deepfake。

c) 攻击者能力：论文中的 attacker agent 可以从预定义攻击集合中选择多个攻击，并控制每个攻击的强度。攻击集合包括良性传统操作，如 JPEG、noise、crop、jitter、affine，也包括 learned benign attacks，以及恶意语义操作，如 face swapping、attribute editing、face anonymization、reenactment。实验中的 deepfake 生成器包括 AttGAN、StarGAN2、InfoSwap、SimSwap、CIAGAN、DeepPrivacy，以及 SimSwap、UniFace、FaceDancer、HyperReenact 等换脸/reenactment 方法。

d) 攻击者知识：训练中的 attacker agent 能观察当前水印器在攻击下的失败情况，并通过 reward 更新策略，因此它是一个自适应训练对手。论文没有把真实外部攻击者设定为完全白盒攻击者，也没有证明攻击者知道 $K$、$\mu$、$\phi$、$\pi$、$\delta$ 参数后仍无法专门优化规避水印。

e) 攻击约束：良性攻击应不改变图像语义；恶意攻击应改变图像身份、属性或语义内容。水印嵌入本身必须保持视觉质量，论文用 PSNR 和 SSIM 衡量水印图像与原图的保真度。半脆弱目标用 Bit Recovery Accuracy 表示：良性操作下 BRA 越高越好，恶意操作下 BRA 越低越好。

f) 防御者能力与安全目标：防御者持有密钥 $K$，能生成 deterministic message $M$，并训练水印器 $\theta=\{\mu,\phi,\pi\}$ 和提取器 $\delta$。安全目标是在未篡改或良性处理图像上低 BER，在语义篡改图像上高 BER，并通过阈值 $\lambda$ 将高 BER 图像判定为 deepfake。

g) 现实性评估：该威胁模型适合主动媒体认证场景，例如平台、相机、发布服务或内容所有者先给图像加水印，再在传播后检测是否被语义篡改。它不适合完全事后检测未加水印图像的 deepfake，也不覆盖密钥泄露、白盒反向优化水印去除、物理拍摄重捕获等更强场景。

h) 与后文关系：方法中的 CLIP latent sphere 对应“语义绑定”；direction generator 和 shared key 对应“无需为每张图存储 watermark message”；attacker agent 对应“训练期自适应攻击者”；curiosity/proximity reward 对应“让攻击 curriculum 覆盖语义漂移和历史失败区域”；BER 阈值对应最终 deepfake 判定。

## 3. 方法设计与复现级理解

### 3.1 全局流程图式说明

DeepForgeSeal 的完整流程由四个核心模块组成：direction generator $\mu$、perturbation MLP $\phi$、decoder $\pi$、extractor $\delta$，以及一个训练期 attacker agent $\eta$。

训练阶段先从 FFHQ 中采样真实图像 $x$，用 CLIP image encoder 得到归一化语义特征 $f(x)$。共享密钥 $K$ 经过 CLIP text embedding 和 direction generator $\mu$，产生一组方向向量 $D=\{d_1,\ldots,d_L\}$。密钥同时确定 watermark message $M$。perturbation MLP $\phi$ 根据 $f(x)$ 和 $M$ 生成扰动 $p$，得到 perturbed latent $q=f(x)+p$，decoder $\pi$ 把 $q$ 解码成水印图像 $x'$。

随后 attacker agent $\eta$ 根据 $f(x')$ 选择攻击组合和强度，生成 attacked image $x_a$。extractor $\delta$ 分别尝试从 $x'$ 和 $x_a$ 中恢复消息。watermarker 通过监督损失更新，目标是保持图像质量、正确嵌入方向编码，并让良性攻击下消息可恢复、恶意语义攻击下消息失效。attacker 则通过 REINFORCE 更新，目标是最大化提取失败，并借助 curiosity/proximity reward 探索语义漂移和历史失败区域。

推理阶段没有 attacker agent。输入查询图像 $x''$ 后，系统用 CLIP encoder 得到 $f(x'')$，用相同 $K$ 和 $\mu$ 再生方向 $D$ 和消息 $M$，extractor 输出恢复消息 $M'$。若 $BER(M',M)>\lambda$，则判定该图像可能被语义篡改，是 deepfake。

### 3.2 Problem formulation：把 deepfake 检测转化为半脆弱水印判定

论文设输入图像为 $x$，CLIP semantic feature 为 $f(x)\in\mathbb{R}^{\zeta}$，并要求其 L2 归一化到单位超球面：

$$f(x)\in\mathbb{R}^{\zeta},\quad \|f(x)\|_2=1$$

共享密钥 $K$ 决定 watermark message $M$。水印器由三个部分组成：

$$\theta=\{\mu,\phi,\pi\}$$

其中 $\mu$ 产生方向，$\phi$ 产生 latent perturbation，$\pi$ 从 latent 表示重建水印图像。攻击者 $\eta$ 把水印图像 $x'$ 变成攻击后图像 $x_a$，提取器 $\delta$ 从任意查询图像 $x''$ 中恢复消息 $M'$。最终判定规则是：

$$f_{\mathrm{DEEPFAKE}}(x'')=\mathbf{1}[BER(M',M)>\lambda]$$

这个 formulation 的关键是：系统不直接训练一个二分类器判断图像真假，而是用水印提取失败作为语义篡改的代理信号。良性操作后 $BER$ 应低，语义恶意操作后 $BER$ 应高。

### 3.3 Directional embedding in latent sphere：用密钥生成可学习方向

这个模块解决的问题是动态水印如何被提取。如果每张图都自由选择不同 latent 方向和消息，检测器在没有原图或消息的情况下无法恢复。DeepForgeSeal 用共享密钥 $K$ 解决这个问题：$K$ 先投影到 CLIP text embedding，得到 $f(K)$，再输入 direction generator $\mu$，生成 $L$ 个单位方向：

$$D=\mathrm{reshape}(\mu(f(K)))\in\mathbb{R}^{L\times\zeta},\quad \|d_i\|_2=1,\forall i$$

水印图像的 CLIP feature 会投影到这些方向上：

$$P=Df(x')\in\mathbb{R}^{L},\quad P_i=\langle f(x'),d_i\rangle$$

对每个 bit $m_i\in\{0,1\}$，系统希望投影值接近不同目标幅度：

$$P_i^{target}=\begin{cases}\xi_1,&m_i=1\\\xi_0,&m_i=0\end{cases}$$

这里的设计理由是：CLIP feature 已经承载高层语义，单位球面约束避免 perturbation 通过改变 feature norm 产生明显异常；方向投影则提供可控的 bit 编码接口。方向由密钥确定，所以检测端可以复现同一组 $D$，不需要为每张图保存 message 或 watermark location。

### 3.4 Perturbative watermark embedding：在 CLIP latent 上生成扰动并解码图像

得到 message $M$ 和语义特征 $f(x)$ 后，perturbation MLP $\phi$ 将二者拼接，输出一个 $\zeta$ 维扰动：

$$p=\phi([f(x);M])$$

扰动加到 CLIP latent 上得到水印 latent：

$$q=f(x)+p$$

decoder $\pi$ 将 $q$ 解码回图像空间：

$$x'=\pi(q)$$

这一步的目标不是像传统像素水印那样在图像上直接加噪声，而是在语义 latent 中找“不明显改变人类感知语义、但能稳定编码消息”的方向。论文强调，视觉质量主要由 CLIP feature similarity loss 约束，没有使用显式 pixel-domain 或 perceptual reconstruction loss。

watermarker 的复合损失为：

$$L_W=\alpha L_{clip}+\beta L_{dir}+\gamma L_{ext}$$

其中保真度项是：

$$L_{clip}=\|f(x)-f(x')\|_2^2$$

方向/消息编码项是：

$$L_{dir}=CrossEntropy(M,\varpi)$$

半脆弱提取项写为：

$$L_{ext}=-BER(M,M')$$

需要注意 $L_{ext}$ 的符号含义。论文希望通过与攻击 curriculum 的交互，让水印器在良性与恶意变换之间形成条件性脆弱，而不是简单让所有攻击后都恢复或都失败。复现时必须区分良性与恶意样本的训练标签/目标，否则会把半脆弱目标写成普通鲁棒水印目标。

### 3.5 Watermark attacker：组合攻击与可学习强度

attacker agent $\eta$ 的输入是水印图像的 CLIP feature $f(x')$，输出两类量：攻击选择 logits 和强度参数 $\tau$：

$$(logits,\tau)=\eta(f(x'))$$

logits 经 sigmoid 得到每个攻击是否启用的概率：

$$B=\sigma(logits)\in[0,1]^{\Gamma}$$

每个攻击动作独立从 Bernoulli 分布采样：

$$a_l\sim Bernoulli(B_l),\quad \forall l\in\{1,\ldots,\Gamma\}$$

因此一次攻击不是单一固定操作，而是一个二进制组合向量 $a\in\{0,1\}^{\Gamma}$。同时，强度 $\tau_l$ 会映射到每种攻击的实际参数范围：

$$param_l=min_l+(max_l-min_l)\cdot\tau_l$$

这部分是 DeepForgeSeal 与固定 attack augmentation 的区别。固定攻击只能覆盖人工设定的局部扰动，随机组合缺少目标性；RL attacker 会根据当前水印器弱点动态选择攻击类型、顺序和强度，从而形成更贴近“训练对手”的 curriculum。

### 3.6 Attacker reward：失败、好奇心和失败记忆

attacker 的主目标是让 extractor 恢复失败，论文用 BCE 衡量提取消息与原消息之间的差异：

$$R_{failure}=BCE(\tilde{M},M)$$

只用失败奖励会导致攻击策略单一，因此论文加入 curiosity reward，鼓励攻击造成更明显的 CLIP semantic drift：

$$R_{curiosity}=\Delta\|f(x')-f(x_a)\|_2^2$$

这里 $x_a$ 是攻击后图像，$\Delta$ 是缩放系数。它的作用是让 attacker 不只找微小像素扰动，而是探索可能真正改变语义的攻击。

此外，论文维护 failure memory buffer $J$，存储历史上导致水印提取失败的 attacked images 的 latent feature。新攻击样本到失败记忆最近邻的距离为：

$$\rho(x_a)=\min_{f_j\in J}\|f(x_a)-f_j\|_2$$

proximity reward 定义为：

$$R_{proximity}=\frac{\nu}{\rho(x_a)+\epsilon}$$

这个奖励鼓励 attacker 向已知失败区域靠近，属于 exploitation；curiosity 则鼓励探索新语义漂移区域，属于 exploration。最终 attacker objective 为：

$$L_{\eta}=-\mathbb{E}_{a\sim\psi}\left[R_{failure}+R_{proximity}+R_{curiosity}-o\sum_{l=1}^{\Gamma}a_l\right]-r\sum_{l=1}^{\Gamma}H(B_l)$$

其中 $\sum a_l$ 惩罚过多激活攻击，$H(B_l)$ 是 Bernoulli action distribution 的 entropy，用于保持策略多样性。论文明确说明：watermarker 用监督损失训练，attacker agent 用 reinforcement learning 训练。

### 3.7 Watermark extraction and deepfake detection

提取阶段输入查询图像 $x''$，先用 CLIP image encoder 得到 $f(x'')$。检测端用相同 secret key $K$ 和 direction generator $\mu$ 重新生成方向集合 $D$。然后对每个方向做投影：

$$P_i''=\langle f(x''),d_i\rangle$$

投影向量 $P''=\{P_1'',\ldots,P_L''\}$ 输入 MLP extractor，得到 message logits $\varpi$，再二值化得到恢复消息 $M'$。最后计算 $BER(M',M)$。若高于阈值 $\lambda$，则说明水印被语义级操作破坏，图像被标记为 deepfake；否则认为图像未被恶意语义篡改。

这一设计的安全意义是：DeepForgeSeal 不依赖固定像素位置，也不直接使用被动 deepfake artifact。它依赖的是“发布时嵌入的语义水印是否仍与当前图像语义一致”。如果身份、表情或语义属性被恶意修改，CLIP latent 与原水印方向的关系被破坏，BER 上升。

### 3.8 训练、构造与推理分离

训练期运行完整 adversarial loop：watermarker 生成水印图像，attacker 生成攻击图像，extractor 尝试恢复消息，watermarker 用 $L_W$ 更新，attacker 用 REINFORCE 更新 $L_\eta$，失败样本进入 memory buffer $J$。

部署期嵌入只需要 $\mu,\phi,\pi$ 和 secret key $K$。输入原图，输出水印图像。

检测期只需要 CLIP encoder、$\mu$、$\delta$、secret key $K$ 和阈值 $\lambda$。attacker agent 不参与检测。这个分离使得 ARL 主要承担训练增强角色，而不是推理时的额外防御模块。

### 3.9 复现配置表

| 项目 | 论文明确设置 |
|---|---|
| 任务 | 主动 deepfake detection；semi-fragile image watermarking |
| 训练数据 | FFHQ，70,000 张 1024×1024；前 60,000 训练，后 10,000 验证 |
| 测试数据 | CelebA 与 CelebA-HQ，按官方 test split |
| CelebA | 10,000 个 identity，每个 identity 20 张图，128×128 |
| CelebA-HQ | 30,000 张高质量图，1024×1024 |
| 语义特征 | 预训练 CLIP image/text encoders，feature L2 normalize |
| 训练期可见恶意操作 | face swapping 与属性级修改，包括 hair color、expression、age |
| deepfake 评估生成器 | AttGAN、StarGAN2、InfoSwap、SimSwap、CIAGAN、DeepPrivacy |
| watermark 攻击评估 | JPG、Noise、Crop、Jitter、Affine、Zhao et al. learned attack、Lukas et al. learned attack |
| 恶意半脆弱评估 | SimSwap、UniFace、FaceDancer、HyperReenact |
| 图像质量指标 | PSNR、SSIM |
| deepfake 检测指标 | ACC、F1-Score、AUC |
| 半脆弱水印指标 | Bit Recovery Accuracy，良性变换下越高越好，恶意变换下越低越好 |
| 优化器/学习率/batch size/epoch | 主文未提供，称实现细节在 supplementary material |
| 代码 | 未发现官方公开代码仓库 |

### 3.10 复现风险与信息缺口

第一，主文没有给出完整网络结构、optimizer、learning rate、batch size、epoch、message length、阈值 $\lambda$ 选择方式、攻击集合参数范围等关键工程细节，这些信息可能在 supplementary material 中，但当前 PDF 主文不完整覆盖。

第二，$L_{ext}$ 的半脆弱训练目标需要谨慎实现。良性攻击和恶意攻击应有不同优化方向；如果简单对所有攻击最大化恢复率，会失去 deepfake 检测能力；如果简单对所有攻击最大化 BER，则良性鲁棒性会崩。

第三，CLIP latent 到图像空间的 decoder $\pi$ 是复现难点。论文报告仅用 CLIP feature-similarity loss 监督 decoder，但缺少架构细节时，很难稳定达到 48 dB 级 PSNR。

第四，attacker agent 的 attack ordering、组合执行方式、memory buffer 更新策略和 REINFORCE 方差控制都会影响训练稳定性。

## 4. 与其他方法对比

| 方法类别 | 代表方法 | 核心思想 | 优点 | 缺点 | DeepForgeSeal 的差异 |
|---|---|---|---|---|---|
| 被动 deepfake 检测 | BTS, CD, PF, RFM, SBI, FakeShield, CBO-DD | 从图像伪影中判断真假 | 不需要预先加水印 | 对未知伪造器泛化不稳 | 用主动水印一致性替代伪影依赖 |
| 像素/图像域主动水印 | FaceSigns, EditGard, OmniGuard, VINE | 在图像中嵌入可检测水印 | 可用于主动认证 | 对像素级良性变换或语义篡改平衡不足 | 在 CLIP latent sphere 中做语义方向嵌入 |
| 半脆弱水印 | FaceSigns, EditGard | 良性操作保留，恶意篡改破坏 | 目标契合 deepfake 检测 | 固定位置或固定攻击训练容易失衡 | 用 learnable directional embedding 和 ARL curriculum 学习平衡点 |
| DeepForgeSeal | 本文 | CLIP latent 方向水印 + ARL attacker | 良性 BRA 高，恶意 BRA 低，检测泛化强 | 需要预嵌入水印，复现细节缺失 | 把语义绑定和训练期自适应攻击结合 |

创新点主要有三层：第一，把水印放到 CLIP semantic latent sphere 的方向投影上；第二，用 secret key 生成可复现、可学习的方向集合，避免固定像素位置；第三，用 attacker agent 学习组合攻击 curriculum，通过 failure、curiosity、proximity reward 同时覆盖当前弱点、新语义漂移和历史失败区域。

## 5. 实验表现与优势

a) 视觉质量实验验证“语义 latent 水印是否足够隐蔽”。在 CelebA-HQ 上，DeepForgeSeal 的 PSNR 为 48.39，SSIM 为 0.97，高于 FaceSigns 的 32.03/0.92、Zhao et al. 的 38.32/0.94、EditGard 的 45.07/0.96、OmniGuard 的 46.15/0.97 和 VINE 的 46.83/0.97。论文认为这是 latent semantic space 嵌入带来的结果，并指出该结果只用 $L_{clip}$ 监督 decoder，没有显式 pixel-domain/perceptual reconstruction loss。

b) deepfake 检测实验验证“水印失效是否能作为语义篡改检测信号”。低分辨率 CelebA 和高分辨率 CelebA-HQ 上，DeepForgeSeal 对 AttGAN、CIAGAN、DeepPrivacy、InfoSwap、SimSwap、StarGAN2 均取得较高 ACC/F1。代表性结果包括 CelebA 上 CIAGAN 0.92/0.95、StarGAN2 0.99/0.99，CelebA-HQ 上 CIAGAN 0.95/0.96、StarGAN2 0.98/0.97。相比被动方法，DeepForgeSeal 对不同伪造器更稳定；相比主动 baseline，它在 DeepPrivacy、StarGAN2 等困难方法上下降更小。

c) 半脆弱水印实验验证“良性鲁棒、恶意脆弱”的核心目标。良性变换下 DeepForgeSeal 的 BRA 为 JPG 1.00、Noise 0.98、Crop 0.98、Jitter 0.99、Affine 1.00、Zhao learned attack 0.98、Lukas learned attack 0.99。恶意操作下 BRA 为 SimSwap 0.24、UniFace 0.11、FaceDancer 0.10、HyperReenact 0.06。这说明水印在普通编辑后大多可恢复，但在语义篡改后显著失效。

d) learnable directional embedding 消融验证 latent sphere 方向编码的必要性。无方向编码的 naive latent embedding 为 PSNR 46.23、SSIM 0.95、benign BRA 0.78、malicious BRA 0.42；固定方向 FD 为 46.58/0.95/0.86/0.19；完整 DeepForgeSeal 为 48.12/0.97/0.99/0.12。完整模型同时提升图像质量、良性鲁棒性和恶意脆弱性。

e) adversarial curriculum 消融验证 attacker agent 的贡献。没有 attacker 时结果为 PSNR 32.11、benign BRA 0.61、malicious BRA 0.59，说明训练严重失衡。固定单攻击、固定多攻击、随机组合、预定义 curriculum 都不如完整模型。完整 DeepForgeSeal 达到 PSNR 48.12、SSIM 0.97、benign BRA 0.99、malicious BRA 0.12，说明可学习组合攻击 curriculum 是主要增益来源之一。

f) reward 消融验证 curiosity/proximity 的互补性。去掉两者时 benign BRA 0.82、malicious BRA 0.35；只去 proximity 为 0.91/0.24；只去 curiosity 为 0.94/0.21；完整模型为 0.99/0.12。说明 curiosity 帮助探索新的语义漂移区域，proximity 帮助利用历史失败区域，两者组合最有效。

## 6. 学习与应用

a) 论文目前没有发现官方开源代码。若要实现，建议按三步走：先实现 CLIP latent 方向投影和 message extraction；再训练 $\phi$ 与 $\pi$ 生成水印图像并保证 PSNR/SSIM；最后加入 attacker agent 和 curriculum reward。

b) 工程上最需要注意：CLIP feature 归一化必须稳定；direction generator 输出方向要重新 normalize；message $M$ 必须由密钥 deterministic 生成；良性/恶意攻击样本的训练目标必须分开；failure memory buffer $J$ 要控制容量和更新条件；REINFORCE 训练需要 entropy 和 action-count regularization 稳定策略。

c) 迁移到其他任务时，DeepForgeSeal 的“语义 latent 半脆弱水印”思想可用于视频或多模态 deepfake 认证，但不适合直接迁移到音频水印。音频中的语义 latent、说话人信息、内容信息、信道扰动和 VC/TTS 重合成关系更复杂，需要重新定义“良性不变、恶意变化”的 latent space。

## 7. 总结

a) 一句话概括：把水印绑定到语义方向。

DeepForgeSeal 的核心价值是把 deepfake 检测从被动伪影识别转成主动语义一致性验证。它通过 CLIP latent sphere 中的方向水印实现可提取消息，再通过 ARL attacker 让水印器学会对普通后处理保持鲁棒、对语义篡改保持脆弱。

## 8. 图表精读与证据链

Fig. 1 展示三类图像：原始 bonafide 图像、经过 learned watermark attacks 后仍能恢复水印的图像、经过语义修改后水印失败的图像。它直观支撑半脆弱目标：不是所有变化都要破坏水印，只有语义改变应破坏水印。

Fig. 2 是方法总览图，说明 secret key 生成方向和消息，watermarker 在 CLIP semantic space 中嵌入水印，attacker 生成良性/恶意攻击 curriculum，extractor 恢复消息并用失败与否判断 tampering。它对应论文的系统边界和 threat model。

Table I 支撑视觉隐蔽性 claim。DeepForgeSeal 的 PSNR 48.39 为最高，SSIM 0.97 与 OmniGuard/VINE 持平，说明 latent embedding 没有明显牺牲质量。

Table II 支撑 deepfake 检测泛化 claim。DeepForgeSeal 在 CelebA 和 CelebA-HQ 的多种伪造器上保持高 ACC/F1，尤其在 CIAGAN、StarGAN2 等 baseline 易下降的场景中更稳。

Table III 是半脆弱性的核心证据。良性变换下 BRA 接近 1，恶意换脸/reenactment 下 BRA 降到 0.06–0.24，说明 resilience 和 fragility 被同时实现。

Table IV–VI 是消融证据链。Table IV 证明 learnable directional embedding 优于 naive/fixed direction；Table V 证明 learnable combinatorial attack curriculum 优于固定或随机攻击；Table VI 证明 curiosity reward 和 proximity reward 是互补的。

证据缺口主要在 supplementary 依赖和白盒安全性。主文多次把 hyperparameter、time complexity、limitation 放到 supplementary，但当前材料中没有完整展开；此外，论文没有充分证明密钥泄露或白盒访问下的抗规避能力。

## 9. 复现难度与适合人群

复现难度：高。原因是该方法同时涉及 CLIP latent 表示、可学习水印生成、图像 decoder、半脆弱训练目标、组合攻击环境、REINFORCE 策略优化、failure memory 和 deepfake 生成器评估。缺少官方代码时，训练稳定性和工程细节会成为主要障碍。

主要依赖：FFHQ、CelebA、CelebA-HQ、CLIP image/text encoder、人脸 deepfake 生成器或预生成数据、PyTorch、较强 GPU、攻击增强库、PSNR/SSIM/BRA/ACC/F1/AUC 评估脚本。

最小可复现版本：先只实现 CLIP latent direction watermark。固定 $D$ 或训练 $\mu$，用 $\phi$ 生成 perturbation，用简单 decoder 或已有重建器生成 $x'$，验证良性 JPEG/crop/noise 下 BRA 是否高。第二阶段加入语义攻击样本，训练 extractor 的半脆弱判定。第三阶段再加入 attacker agent 和 reward memory。

适合人群：适合研究主动 deepfake 检测、图像水印、生成式媒体认证、ARL 安全训练的人。若目标只是语音水印，这篇更适合作为“语义半脆弱水印”和“自适应攻击 curriculum”的参考。

## 10. 简短全面总结

DeepForgeSeal 是一篇面向 deepfake 检测的主动半脆弱图像水印论文。它认为被动检测依赖伪造伪影，难以泛化；已有主动水印又难以同时抵抗良性后处理并对语义篡改敏感。论文将图像映射到 CLIP 单位超球面，用密钥生成可学习方向集合，将 message 编码为特征在这些方向上的投影，再由扰动 MLP 和 decoder 生成水印图像。训练中，attacker agent 通过组合良性/恶意攻击并控制强度，利用 failure、curiosity 和 proximity reward 学习动态攻击 curriculum，迫使 watermarker 学到鲁棒性与脆弱性的平衡。实验在 FFHQ 训练、CelebA/CelebA-HQ 测试，显示 DeepForgeSeal 在视觉质量、deepfake 检测和半脆弱 BRA 上优于 FaceSigns、EditGard、OmniGuard、VINE 等方法。主要局限是需要预先嵌入水印，主文缺少完整复现超参数，且未充分覆盖白盒密钥泄露或针对性规避攻击。

## 11. 论文写作逻辑分析

a) Intro 的问题铺垫从 deepfake 风险切入，但没有停留在泛泛安全威胁，而是进一步指出被动检测依赖伪影、主动水印存在 resilience-fragility paradox。这个铺垫能自然引出“半脆弱水印”的必要性。

b) insight 的推导比较直接：良性操作通常不改语义，恶意 deepfake 操作会改语义，因此水印应该绑定语义 latent，而不是固定像素位置。CLIP sphere 的引入服务于这个 insight。

c) threat model 的承接体现在方法模块上。共享密钥解决检测端再生消息问题；latent direction 解决动态嵌入和固定位置脆弱问题；ARL attacker 解决固定攻击训练不能覆盖真实攻击复杂性的问题。

d) 方法叙事顺序清楚：先定义水印器和提取器，再解释方向嵌入，再解释扰动解码，随后引入 attacker 和 reward，最后给出检测判定。每个模块都有明确前后接口。

e) 实验呼应较完整：Table I 对应隐蔽性，Table II 对应 deepfake 检测效果，Table III 对应半脆弱目标，Table IV–VI 对应三个创新点的消融。整体 claim -> method -> metric -> result 链条比较闭合。

f) 证据最强的是半脆弱 BRA 和消融实验，因为它们直接验证论文核心矛盾。较弱的是复现细节和白盒安全边界，主文把多个关键细节留给 supplementary，且没有系统讨论密钥泄露、白盒优化、重捕获等更强攻击。

g) 可借鉴写法是：先把安全目标写成一个清晰悖论，再把方法模块逐一对应到悖论的两端，最后用主实验和消融分别证明“既鲁棒又脆弱”不是偶然结果。这种结构对主动水印和安全防御论文都很有效。

## Overview.md 条目

- [DeepForgeSeal: Latent Space-Driven Semi-Fragile Watermarking for Deepfake Detection Using Adversarial Reinforcement Learning](./Other_Security/2026-DeepForgeSeal.md)  
  *IEEE Transactions on Pattern Analysis and Machine Intelligence (TPAMI), 2026*  
  Citation: Fernando, T., Fookes, C., & Sridharan, S. “DeepForgeSeal: Latent Space-Driven Semi-Fragile Watermarking for Deepfake Detection Using Adversarial Reinforcement Learning.” *IEEE Transactions on Pattern Analysis and Machine Intelligence*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2511.04949) | [DOI](https://doi.org/10.1109/TPAMI.2026.3721108)
