# ROBIN++：面向扩散模型的版权保护与篡改定位双域协同水印

## 0. 翻译摘要原文

数字水印为验证生成内容的来源和完整性提供了一种有前景的方案。然而，现有方法通常受限于版权鲁棒性和定位敏感性之间的固有折中，这是因为相互冲突的信号被纠缠在同一个域中。本文提出 ROBIN++，将范式从单域纠缠转向双域协同设计。核心 insight 是：在注入阶段显式解耦鲁棒性和脆弱性，把它们分配到不同域中；而在验证阶段重新耦合，使二者相互增强。具体而言，作者引入对抗优化算法，在生成过程中注入频域版权水印，在不产生视觉伪影的情况下最大化提取鲁棒性。作为补充，作者将内容感知扰动生成器集成到 latent space 中，注入空间域定位水印，使其对空间篡改敏感，并尽量减少对版权水印的干扰。在验证阶段，作者提出双分支协同检测器，实现 Frequency-to-Spatial Synergy：鲁棒频谱先验作为局部异常检测的全局锚点。反过来，Spatial-to-Frequency Refinement 使用预测篡改 mask 修正被污染的 latent 特征。大量实验表明，ROBIN++ 在版权恢复和篡改定位准确率上显著优于现有方法，有效缓解了两个任务之间长期存在的折中。

## 1. 方法动机

a) 生成式图像扩散模型让高保真图像生成变得廉价，但也带来两个安全问题：一是生成图像的版权来源难验证，二是生成图像可被局部 inpainting、splicing、editing 后用于误导传播。因此平台或模型拥有者希望同一套水印系统同时回答两个问题：这张图是否来自我的生成模型，以及它是否被局部篡改、篡改区域在哪里。

b) 现有方法的核心痛点是 dual-function conflict。鲁棒版权水印希望在 JPEG、旋转、模糊、重建、扰动移除后仍能被检测；篡改定位水印则希望一旦局部内容被改动，局部信号立即破坏并暴露位置。前者需要稳定、全局、抗变换；后者需要脆弱、局部、空间对齐。如果把这两类信号都塞进同一个空间域或同一个 latent 位置，就会相互压制：增强鲁棒性会降低定位敏感性，增强脆弱性会破坏版权恢复。

c) ROBIN++ 的关键 insight 是“注入时解耦，验证时再耦合”。注入阶段把鲁棒版权水印放到频域中低频结构，把脆弱定位水印放到空间/高频局部扰动；验证阶段不再把二者独立读出，而是让频域水印帮助定位器区分全局良性退化和局部恶意篡改，再让定位器输出的篡改 mask 反过来屏蔽污染区域，提高版权水印提取可靠性。

## 2. 威胁模型解读

a) 参与方与系统边界：防御者是扩散模型或图像生成服务拥有者，控制离线水印优化、在线生成过程和验证器。输入是文本 prompt 与初始噪声，输出是带双水印的生成图像。验证端接收一张可能被压缩、降质或局部篡改的图像，输出版权认证结果和篡改 mask。

b) 攻击者目标：攻击者可能希望移除版权水印，使模型来源无法验证；也可能对图像局部篡改，例如 inpainting、splicing、替换局部区域，同时希望篡改不被定位；还可能同时进行良性/恶性混合处理，如 JPEG 后再局部编辑。

c) 攻击者能力：实验覆盖全局退化与安全攻击，包括旋转 75 度、JPEG quality 25、Gaussian blur radius 5、adversarial noise、frequency noise、low pass、frequency smoothing、latent removal、diffusion reconstruction、VAE reconstruction、Rinse 2x、perturbation defeat、latent forgery，以及 SD Inpaint、SDXL、Kandinsky、ControlNet、LaMa 生成式 inpainting。

d) 攻击者知识：论文主要评估黑盒或非自适应攻击集合，没有设定攻击者完全知道 $w_{id}$、$w_p$、$G_{loc}$、$D_{loc}$、频域 mask $\Omega$ 和阈值后进行端到端白盒优化。black-box forgery 作为攻击类型出现，但不是完整 adaptive white-box removal threat model。

e) 攻击约束：攻击后图像仍应保持可用和视觉自然。论文用 PickScore 评估生成质量，用 Bit Accuracy/AUC 评估版权水印可分性，用 F1/AUC/IoU 评估定位精度。局部篡改区域在实验中约覆盖图像 1% 到 60%。

f) 防御者能力与安全目标：防御者可以修改扩散生成过程，在中间 timestep 注入频域版权水印，在最终 latent 注入定位水印；可以离线优化水印和 prompt guidance，也可以训练双分支定位器。安全目标是三分类：Authentic & Intact、Authentic & Tampered、Unverified or Destroyed。

g) 现实性评估：该模型贴近“生成平台发布带水印图像，后续第三方压缩、编辑、再传播”的场景。强假设是防御者必须能控制生成过程，且验证端可执行 DDIM/DPM 反演。弱覆盖是自适应白盒去水印、针对双分支 detector 的对抗样本、截图/打印拍摄、真实社交平台多轮转码和大规模密钥管理。

h) 与后文关系：频域中低频注入对应版权鲁棒性；CAP 高频空间扰动对应局部敏感性；F2S 对应“良性全局退化不应被误判为局部篡改”；S2F 对应“局部篡改不应导致整张图版权验证失败”。实验中的 combined degradation + tampering 正是对这个 threat model 的核心验证。

## 3. 方法设计与复现级理解

### 3.1 全局流程图式说明

ROBIN++ 分成三个阶段。

第一阶段是离线优化。给定扩散模型 $\epsilon_\theta$、VAE 编解码器、训练图像/提示词集合，先优化频域版权水印 $w^*_{id}$ 和隐藏 prompt guidance $w^*_p$。$w^*_{id}$ 负责强版权认证，$w^*_p$ 负责让扩散模型吸收频域扰动、减小视觉伪影。随后训练 Content-Aware Perturbation 生成器 $G_{loc}$ 和定位检测器 $D_{loc}$，用于空间域定位水印。

第二阶段是在线生成。扩散采样从 $z_T$ 开始正常去噪；到注入 timestep $t_{inj}$ 时，在 latent 的频域 mask $\Omega$ 内嵌入 $w^*_{id}$；之后用原 prompt 和 $w^*_p$ 共同引导继续去噪。到最终 latent $z^*_0$ 后，$G_{loc}$ 根据内容生成脆弱定位扰动 $w_{loc}$，得到 $z_w=z^*_0+w_{loc}$，再经 VAE decoder 输出水印图像 $x_w$。

第三阶段是验证。版权分支对可疑图像 $\hat{x}$ 做 DDIM inversion，回到注入 timestep 的 latent，再从频域 mask 中提取 $\hat{w}_{id}$ 并与参考水印比较。定位分支用 RGB + FFT 双分支 U-Net 预测篡改概率图 $\hat{P}$ 和 mask $\hat{M}$。如果检测到篡改，S2F refinement 把 latent 中被 mask 标记的污染区域替换为 intact 区域统计量，再进行频域版权提取，以降低局部篡改对全局认证的污染。

### 3.2 Task Definition：双信号、三种验证状态

论文将干净生成图记为 $x\in R^{H\times W\times C}$，目标是注入两个信号：鲁棒版权水印 $w_{id}$ 和脆弱定位水印 $w_{loc}$。组合注入函数为：

$$x_w=I(x,w_{id},w_{loc}).$$

传播过程被建模为复合退化：

$$\hat{x}=T(x_w),\quad T=T_{benign}\circ T_{tamper}.$$

$T_{benign}$ 对应压缩、缩放、模糊等全局变换；$T_{tamper}$ 对应 inpainting、splicing 等局部恶意篡改。验证器包含版权提取器 $D_{id}$ 和定位器 $D_{loc}$。系统判决分三类：若 $\hat{w}_{id}=w_{id}$ 且 $\hat{M}=0$，则 authentic & intact；若 $\hat{w}_{id}=w_{id}$ 且 $\hat{M}\ne0$，则 authentic & tampered；若 $\hat{w}_{id}$ 与 $w_{id}$ 距离超过阈值 0.5，则 unverified or destroyed。

### 3.3 Dual-Domain Injection：频域版权水印与空间定位水印分开注入

版权水印注入在扩散中间阶段。论文选择 $t_{inj}\in[200,300]$，因为早期 timestep 决定全局语义布局，过早注入会影响语义；晚期 timestep 主要做局部细节，过晚注入则模型没有足够去噪自由度吸收频域扰动。中间 detail formation 阶段语义已基本稳定，同时仍能把频域扰动吸收到图像细节中。

频域嵌入形式为：

$$z^*_{t_{inj}}=\mathrm{FreqEmbed}(z_{t_{inj}},w^*_{id},\Omega).$$

$\Omega$ 是频域 mask，默认覆盖 70% 频域，集中在中低频。中低频比纯高频更抗 JPEG、blur、noise 和部分重建攻击，但如果覆盖过大又会影响生成质量。

注入后去噪使用带 prompt guidance 的噪声预测：

$$\hat{\epsilon}_{\theta}(z^*_t,t,\psi(p),w^*_p)=\eta_1\epsilon_{\theta}(z^*_t,t,\psi(p))+\eta_2\epsilon_{\theta}(z^*_t,t,w^*_p)+(1-\eta_1-\eta_2)\epsilon_{\theta}(z^*_t,t,\varnothing).$$

这里 $w^*_p$ 不是文本语义 prompt，而是离线优化得到的隐藏 guidance 信号。它的作用是让模型在后续去噪中把 $w^*_{id}$ 融入图像，而不是留下明显伪影。

定位水印注入在最终 latent：

$$z_w=z^*_0+w_{loc}=z^*_0+G_{loc}(z^*_0).$$

这样做的理由是定位水印要求严格空间对齐。若在扩散轨迹中太早注入，随机去噪会破坏像素/latent 对应关系；放在 $z^*_0$ 后再经 VAE decoder，相当于用 decoder 将局部 latent 扰动上采样到局部像素邻域，使其对局部编辑敏感。

### 3.4 Adversarial Optimization：同时优化强水印和隐藏 guidance

离线优化阶段同时求 $w^*_{id}$ 和 $w^*_p$。$w_p$ 的目标是保持图像，$w_{id}$ 的目标是在图像保持约束下尽量增强水印幅度。图像保持损失为：

$$L_{ret}=\mathrm{MSE}(x'^*_0,x_0).$$

其中 $x'^*_0$ 是由带水印 latent 和 guidance 预测出的最终图像：

$$x'^*_0=\mathrm{Dec}\left(\frac{z^*_t-\sqrt{1-\bar{\alpha}_t}\epsilon_\theta(z^*_t,t,\psi(p),w_p)}{\sqrt{\bar{\alpha}_t}}\right).$$

为了避免 $w_p$ guidance 过强导致 DDIM inversion 误差增大，作者加入约束项：

$$L_{cons}=\mathrm{MSE}(\epsilon_\theta(z^*_t,t,w_p)-\epsilon_\theta(z^*_t,t,\psi(\varnothing))).$$

最终两个优化目标为：

$$L_{w_p}=\alpha L_{ret}+\beta L_{cons}.$$

$$L_{w_{id}}=\alpha L_{ret}+\beta L_{cons}-\lambda\|w_{id}\|.$$

注意 $w_{id}$ 的目标中有负的强度项 $-\lambda\|w_{id}\|$，所以优化会在视觉保持与 inversion 约束允许的范围内把水印做强。算法采用 alternating optimization：固定一个变量更新另一个变量，而不是一次性联合更新。

### 3.5 CAP Generator：内容感知的空间脆弱扰动

CAP 的任务不是做鲁棒水印，而是让局部空间变化能被检测出来。$G_{loc}$ 是轻量 CNN，输入最终 latent $z^*_0$，输出 residual $w_{loc}$。论文强调 content-aware 的原因是不同图像区域纹理、边缘、平坦度不同，统一强度扰动容易在平坦区域可见，或在复杂区域检测不到；根据内容自适应调制扰动强度能同时提高隐蔽性和可检测性。

为了不干扰中低频版权水印，作者对 $w_{loc}$ 施加 high-pass filtering，使其能量主要集中在高频。质量约束为：

$$L_{qual}=\|\mathrm{Dec}(z_w)-\mathrm{Dec}(z^*_0)\|^2_2+\gamma\cdot\mathrm{LPIPS}(\mathrm{Dec}(z_w),\mathrm{Dec}(z^*_0)).$$

这里第一项控制像素差异，LPIPS 控制感知差异。论文设置 $\gamma=1$，并用扰动预算 $\xi=0.5$ 防止定位水印过强。

### 3.6 Self-Supervised Splicing：不用人工篡改标注训练定位器

定位器训练需要“被篡改区域”的 mask。ROBIN++ 不依赖人工标注篡改数据，而是构造 hard negative splicing。先生成带定位水印图：

$$x_w=\mathrm{Dec}(z_w),\quad x^*_0=\mathrm{Dec}(z^*_0).$$

然后采样随机二值 mask $M\in\{0,1\}^{H\times W}$，把一部分区域替换为不含定位水印的对应图像：

$$x_{spliced}=M\odot x^*_0+(1-M)\odot x_w.$$

在这个定义里，$M$ 表示 fragile localization watermark 被移除/替换的篡改区域，$(1-M)$ 区域仍带定位水印。由于 $x_w$ 与 $x^*_0$ 视觉上几乎相同，splicing 边界非常隐蔽，检测器不能依赖明显颜色或语义伪影，只能学习定位水印被破坏的内在信号。这比直接用真实 inpainting 数据训练更符合“主动水印定位”的逻辑。

### 3.7 Dual-Branch Synergistic Detector：RGB 空间分支 + FFT 频域分支

$D_{loc}$ 采用 U-Net 风格结构，encoder 有两个分支。空间分支处理 RGB 图像，用于捕获局部不连续、边界和纹理异常；频率分支处理 FFT-based spectral features，用鲁棒版权水印的频域一致性作为全局锚点。两个分支特征 concat 后送入 decoder，输出像素级篡改概率：

$$\hat{P}=D_{loc}(x_{spliced}),\quad \hat{M}=1(\hat{P}>\tau_{loc}).$$

F2S synergy 的实际含义是：JPEG、blur、noise 这类全局良性退化会改变整张图，但不应产生局部篡改 mask；局部 inpainting 会破坏空间/频谱局部一致性，应该被定位。频域分支提供“全局水印仍在”的先验，让定位器不把全局压缩误判成局部攻击。

定位损失结合 BCE 和 Dice：

$$L_{seg}(\hat{M},M)=\lambda_{bce}L_{bce}(\hat{M},M)+\lambda_{dice}L_{dice}(\hat{M},M).$$

$$L_{bce}(\hat{M},M)=-\frac{1}{HW}\sum_i\sum_j\left[M_{i,j}\log(\hat{M}_{i,j})+(1-M_{i,j})\log(1-\hat{M}_{i,j})\right].$$

$$L_{dice}(\hat{M},M)=1-\frac{2\sum_{i,j}(M_{i,j}\hat{M}_{i,j})+\epsilon}{\sum_{i,j}M_{i,j}+\sum_{i,j}\hat{M}_{i,j}+\epsilon}.$$

总训练目标为：

$$L_{loc}=\lambda_{seg}L_{seg}(\hat{M},M)+\lambda_{qual}L_{qual}.$$

论文设置 $\epsilon=1e-5$，$\lambda_{bce}=0.2$，$\lambda_{dice}=0.8$，$\lambda_{seg}=\lambda_{qual}=1$。

### 3.8 Watermark Validation：DDIM 反演认证与 S2F 修正

版权认证时，验证器对 $\hat{x}$ 做 DDIM inversion，恢复注入 timestep 的 latent $\hat{z}_{t_{inj}}$。由于 blind verification 中原 prompt 通常不可用，论文按前人做法使用 null-text embedding，guidance scale 为 1.0。随后在 $\Omega$ 内提取频谱水印 $\hat{w}_{id}$，用归一化 L1 距离判断：

$$D(w_{id},\hat{w}_{id})=\frac{1}{|\Omega|}\sum_{k\in\Omega}|w^k_{id}-\hat{w}^k_{id}|\le\tau_i.$$

如果图像有局部篡改，直接反演会让污染区域在频域中扩散，导致版权认证误判。S2F refinement 用定位 mask 引导 latent 修正。先把像素 mask 下采样为 latent mask $\hat{M}_z$，再用未篡改区域的空间平均特征 $\bar{z}_{intact}$ 替换被篡改区域：

$$\tilde{z}^{(i,j)}_0=\hat{z}^{(i,j)}_0\ (\hat{M}^{(i,j)}_z=0),\quad \tilde{z}^{(i,j)}_0=\bar{z}_{intact}\ (\hat{M}^{(i,j)}_z=1).$$

作者选择 mean filling 而不是 zero filling，是为了避免人为产生高频边缘，进一步污染频域水印。这个步骤就是 Spatial-to-Frequency：空间定位结果反过来提高频域版权验证。

### 3.9 训练、构造与推理分离

离线阶段运行两类优化：第一类优化 $w^*_{id}$ 和 $w^*_p$，需要扩散模型前向、VAE decoder、训练 prompt 和生成图；第二类训练 $G_{loc}$ 和 $D_{loc}$，需要 MS-COCO 训练图和自监督 splicing。在线生成阶段不再训练，只在扩散采样中执行频域注入、guided denoising、CAP 扰动和 VAE decode。验证阶段不需要生成 prompt，但需要 inversion 和 detector。

这一区分对复现很关键：$w^*_{id}$ 和 $w^*_p$ 是模型级/配置级准备成本，不是每张图都重新优化；$G_{loc}$ 和 $D_{loc}$ 是训练好的网络；在线每张图只增加注入和检测开销。

### 3.10 复现配置表

| 项目 | 论文设置 |
|---|---|
| 基础模型 | Stable Diffusion-v2；另评估 SD v1.4、SD v2.0、SDXL、PixArt-alpha 泛化 |
| prompt 数据 | Gustavosta/Stable-Diffusion-Prompts |
| CAP 训练数据 | MS-COCO training set，自监督，无人工篡改标注 |
| 评测图像 | 1,000 watermarked + 1,000 clean 用于鲁棒性/AUC |
| 定位评测 | StableGuard dataset 的 5,000 image-mask pairs |
| 篡改模型 | Stable Diffusion Inpaint、SDXL、Kandinsky、ControlNet、LaMa |
| 采样器 | DPM-Solver |
| 推理设置 | guidance scale 7.5，50 steps |
| 频域水印优化 | 50 generated images；$w_{id}$ lr 0.8；$w_p$ lr 5e-4；1,000 rounds |
| 频域覆盖 | 默认 70% |
| CAP 输入尺寸 | 512×512 |
| 优化器 | AdamW，weight decay 1e-2 |
| 学习率 | generator 1e-3，detector 1e-4 |
| 损失权重 | $\gamma=1$，$\lambda_{bce}=0.2$，$\lambda_{dice}=0.8$，$\lambda_{seg}=\lambda_{qual}=1$ |
| 扰动约束 | $\xi=0.5$ |
| 定位阈值 | $\tau_{loc}=0.5$ |
| 硬件 | NVIDIA GeForce RTX 4090 |
| 成本 | 离线准备约 50.2 小时；在线生成 +0.14 s/img 和 +1.0 GB VRAM；验证 0.78 s/img |

### 3.11 最小可复现伪代码

```text
离线优化:
  初始化 w_id, w_p
  repeat until converged:
    采样图像/提示词 (x, p)，采样 t in [200, 300]
    前向扩散得到 z_t
    在频域 mask Omega 中嵌入 w_id 得到 z_t*
    用 w_p 和 prompt 预测 x_0'，计算 L_ret 与 L_cons
    更新 w_id 以最小化 alpha L_ret + beta L_cons - lambda ||w_id||
    更新 w_p 以最小化 alpha L_ret + beta L_cons

  repeat until converged:
    从带 w_id 的扩散轨迹取得 z_0*
    w_loc = G_loc(z_0*)
    z_w = z_0* + w_loc
    x_w = Dec(z_w), x_0* = Dec(z_0*)
    采样随机 mask M，构造 x_spliced = M x_0* + (1-M) x_w
    D_loc 预测 mask，计算 L_seg 和 L_qual
    更新 G_loc 与 D_loc

在线生成:
  从 z_T 开始按 prompt 正常去噪
  到 t_inj 时执行 FreqEmbed(z_t, w_id*, Omega)
  之后用 prompt 与 w_p* guided denoising
  在 z_0* 上注入 G_loc(z_0*)，VAE decode 得到 x_w

验证:
  对 query image 做 DDIM inversion，提取频域水印并比较 L1 距离
  用 D_loc 预测 tamper mask
  若有篡改，用 mask 修正 latent 后再次做版权提取
  输出 Authentic & Intact / Authentic & Tampered / Unverified
```

### 3.12 复现风险与信息缺口

论文给出了主要超参数，但仍有几个复现风险。第一，Freq-Embed 的具体频域 ring/mask 构造、频谱归一化和 watermark 初始化细节需要代码确认。第二，$w_p$ 作为 prompt guidance 的实现形式、与 text embedding 的维度接口、是否逐层/逐 timestep 共享，正文没有完全展开。第三，自监督 mask $M$ 的采样形状、尺度分布、边界平滑策略未在正文充分描述。第四，附录中提到的 adaptive timestep、SDXL/PixArt-alpha 详细结果、效率拆分未全部出现在当前 PDF 正文抽取中。第五，ROBIN++ TPAMI 代码当前仓库标注为 Coming soon，因此完整复现要等待代码或自行补齐上述工程细节。

## 4. 与其他方法对比

| 方法 | 类别 | 核心思想 | 优点 | 局限 | ROBIN++ 改进 |
|---|---|---|---|---|---|
| Tree-Ring / frequency watermark | 扩散频域版权水印 | 在初始噪声或频域结构中嵌入鲁棒水印 | 版权认证鲁棒 | 不做精细篡改定位 | 保留频域鲁棒性并加入空间定位水印 |
| EditGuard | 主动图像编辑保护 | 通过水印检测编辑/篡改 | 可用于局部编辑检测 | 多退化下鲁棒性不稳 | 双域解耦降低鲁棒/脆弱冲突 |
| OmniGuard / WAM / TAG-WM | 生成式图像水印 | 多数在同一域中联合优化鲁棒与定位 | 能覆盖部分双功能 | 同域纠缠导致 trade-off | 注入域和验证阶段都显式拆分 |
| StableGuard | 统一版权保护与定位 | 在解码阶段联合注入/检测 | 定位强 | 版权鲁棒性受限，跨模型转移弱 | CAP 放在 shared latent，频域版权独立 |
| ROBIN++ | 双域协同水印 | 频域版权 + 空间 CAP + F2S/S2F | 同时高鲁棒和高定位 | 依赖生成过程可控与反演 | 解决 dual-function trade-off |

ROBIN++ 的创新点不只是“多加一个分支”，而是完整地把两个目标在注入、训练、验证三个层面重新组织：注入时用不同域避免互扰；训练时用自监督 splicing 学定位水印破坏；验证时让两个任务互相提供先验。

## 5. 实验表现与优势

a) 实验设计覆盖三类目标：版权鲁棒性、篡改定位、图像质量。鲁棒性用 Bit Accuracy/AUC，定位用 F1/AUC/IoU，质量用 PickScore。基线包括 EditGuard、OmniGuard、WAM、TAG-WM、StableGuard，以及被动篡改检测 MVSS-Net、HDF-Net。

b) 主表 Table I 显示，ROBIN++ 在 Clean、Rotation、JPEG、Blur 下平均 Bit Acc/AUC 为 0.973/0.984。尤其 Rotation 场景，许多 baseline 接近随机，而 ROBIN++ 达到 0.892/0.935。定位平均精度达到 0.985，PickScore 为 19.124，与最高 baseline 差距 0.230。

c) Table II 的高级攻击中，ROBIN++ 在 imperceptible adversarial noise、frequency attacks、GenAI removal、perturbation removal 上整体保持竞争或最优表现。例如 latent removal 为 0.975/0.995，diffusion reconstruction 和 VAE reconstruction 为 1.000/1.000，black-box forgery FSR 为 0.411，低于其他主动水印方法。

d) Table III 的生成式 inpainting 定位中，ROBIN++ 在五类 inpainting 模型上均达到最高或接近最高：SD Inpaint F1/AUC/IoU 为 0.988/1.000/0.976，SDXL 为 0.984/1.000/0.969，LaMa 为 0.986/1.000/0.973。平均定位精度优于 StableGuard 的 0.978。

e) combined degradation + tampering 是最贴近 threat model 的实验。Fig. 6 报告 ROBIN++ 在混合退化和篡改下平均 localization IoU 为 0.879，verification accuracy 为 0.996，说明 F2S 和 S2F 的互相增强不是只在单一干净条件下成立。

f) 消融实验清楚支撑模块贡献。直接频域水印到 adversarial optimization，平均鲁棒性从 0.973 提升到 0.979；加入 CAP 后 localization IoU 从 0.543 大幅升至 0.947；加入 F2S 后 localization IoU 进一步升至 0.971，同时鲁棒性为 0.970；加入 S2F 后鲁棒性从 0.970 升到 0.973。最大贡献来自 CAP，因为它把定位从被动统计异常变成主动脆弱水印检测。

g) 泛化方面，Table VI 显示从 SD v2.1 zero-shot 转到 SD v1.4 时，StableGuard F1 只有 0.007，而 ROBIN++ 有 0.765；转到 SD v2.0 时两者都较高，ROBIN++ 为 F1 0.974、AUC 0.999、IoU 0.956。作者还称在 SDXL 和 PixArt-alpha 上也有较好泛化，但详细表在附录，当前正文只给出概述。

h) 局限性方面，论文承认当前验证仍需要 diffusion inversion，这带来计算成本和对反演稳定性的依赖；并且当图像经历灾难性破坏时，版权认证会进入 unverified or destroyed。更强的白盒自适应水印移除和真实物理传播未被充分覆盖。

## 6. 学习与应用

a) 论文对应代码仓库为 `https://github.com/Hannah1102/ROBIN`。截至当前查看，仓库说明当前发布的是 NeurIPS 2024 ROBIN，TPAMI 2026 ROBIN++ 代码标注为 Coming soon。因此如果要复现 ROBIN++，目前不能完全依赖开源代码。

b) 实现关键步骤：先复现频域版权水印的 adversarial optimization，包括 $w_{id}$、$w_p$、$\Omega$ 和 $t_{inj}$；再实现 CAP generator 和 self-supervised splicing；最后实现 RGB/FFT 双分支 U-Net 与 DDIM inversion 版权提取。建议先做最小版：固定频域 mask + 固定 $t_{inj}$ + 简化 CNN CAP + 单一 inpainting 评测。

c) 超参数上，$t_{inj}\in[200,300]$、频域覆盖 70%、$\xi=0.5$、$\lambda_{bce}=0.2$、$\lambda_{dice}=0.8$ 是最应优先对齐的设置。若换模型到 SDXL/PixArt-alpha，需要重新确认 latent 尺寸、VAE decoder 行为、FFT 特征尺度和 inversion 稳定性。

d) 迁移方向：该思想可迁移到其他生成式模型水印，例如视频扩散模型可把鲁棒版权信号放到时间稳定频域/低频运动结构，把定位信号放到帧级局部高频；音频生成水印也可借鉴“鲁棒来源认证 + 脆弱篡改定位”的双域设计，把鲁棒信号放到较稳定谱包络，把脆弱信号放到局部时频细节。

## 7. 总结

双域解耦注入，双向协同验证。

## 8. 图表精读与证据链

| 图表 | 作用 | 证据链 |
|---|---|---|
| Fig. 1 | 展示三种验证状态：完整、被篡改、不可验证 | 对应 Section 2 的系统输出定义 |
| Fig. 2 | 鲁棒性与定位精度二维比较 | 支撑 ROBIN++ 缓解 trade-off |
| Fig. 3 | 对比单域纠缠与双域协同 | 解释为什么要把频域版权和空间定位分离 |
| Fig. 4 | 总框架图 | 是方法复现主图，覆盖离线优化、在线注入、验证协同 |
| Table I | 鲁棒性、定位、质量总对比 | 主证据：ROBIN++ 同时高 AUC、高定位、高 PickScore |
| Table II | 高级攻击和移除攻击 | 支撑版权水印在强攻击下仍可恢复 |
| Table III | 五类生成式 inpainting 定位 | 支撑定位不仅适配单一篡改模型 |
| Fig. 5 | 定位可视化 | 证明 mask 边界更贴近 GT，减少碎片和误报 |
| Fig. 6 | 混合退化 + 篡改 | 最贴近真实传播，支撑 F2S/S2F 协同价值 |
| Table IV | 模块消融 | CAP 是定位最大贡献，F2S/S2F 证明双向协同不是口号 |
| Table V | $\xi$ 与频域覆盖率消融 | 支撑默认 0.5 与 70% 的折中选择 |
| Fig. 7 | 双水印频谱分离 | 证明中低频版权与高频 CAP 互扰较少 |
| Fig. 8 | F2S 效果 | 证明频域分支可区分 JPEG 和局部 inpainting |
| Fig. 9 | S2F 效果 | 证明 mask-guided latent filling 可降低版权提取误差 |
| Table VI | SD v2.1 到其他 SD 版本迁移 | 支撑 latent/CAP 设计比改 decoder adapter 更可转移 |

证据最强的是 CAP/F2S/S2F 消融和主表联合指标。证据相对不足的是 adaptive white-box attacker；论文没有证明攻击者在知道双域结构后无法同时去除中低频版权水印和高频 CAP。

## 9. 复现难度与适合人群

复现难度：高。原因是它同时依赖扩散采样控制、频域 latent 操作、prompt guidance 优化、VAE latent 注入、DDIM inversion、U-Net segmentation、自监督 splicing 和多类生成式 inpainting 评测。即使不复现所有攻击，最小闭环也需要较完整的 Stable Diffusion 工程栈。

主要依赖：Stable Diffusion-v2、VAE、DPM-Solver/DDIM inversion、Gustavosta prompt 数据、MS-COCO、StableGuard 评测集、inpainting 模型、RTX 4090 级别 GPU。离线准备约 50.2 小时。

最小可复现版本：只在 SD v2 上做 512×512 图像；固定 $t_{inj}=250$ 和 70% 频域 mask；先复现版权水印 AUC，再训练简化 CAP + U-Net 做 self-supervised splicing，最后只测 JPEG、rotation 和 SD Inpaint。

适合人群：适合做生成式图像水印、扩散模型安全、图像篡改定位、主动法证的研究者。不适合作为初学者第一篇扩散水印复现论文，因为工程耦合较重，且 ROBIN++ 完整代码尚未发布。

## 10. 简短全面总结

ROBIN++ 解决扩散生成图像中的双重验证问题：既要证明图像来自某个模型/服务，又要定位后续局部篡改。论文指出鲁棒版权水印和脆弱定位水印在单域内天然冲突，因此提出双域协同范式：生成中段在 latent 频域注入中低频版权水印，并用对抗优化和隐藏 prompt guidance 提高鲁棒性与隐蔽性；最终 latent 中由 CAP 生成高频空间定位扰动，通过自监督 splicing 训练双分支 U-Net 检测篡改。验证时，频域水印帮助空间定位器区分全局良性退化和局部恶意篡改，预测 mask 又反过来修正 latent 以提升版权提取。实验在 Stable Diffusion、五类 inpainting、强退化和攻击下显示 ROBIN++ 同时取得高版权 AUC、高定位 F1/IoU 和可接受 PickScore。主要局限是依赖可控生成与反演，且未充分覆盖白盒自适应去水印。

## 11. 论文写作逻辑分析

a) Intro 从生成模型带来的版权和篡改风险切入，没有只谈“水印鲁棒性”，而是强调 unified verification：来源和完整性需要同时验证。这让论文问题比单一版权水印更具体。

b) Motivation 的关键是 trade-off 图式。作者不是简单说现有方法差，而是指出鲁棒信号和脆弱信号在同一域纠缠，目标函数相互冲突。这个分析自然推出 dual-domain decoupling。

c) Threat model 与方法承接较紧：版权水印要抵抗全局退化和移除攻击，所以放到频域中低频并在扩散轨迹中注入；定位水印要对局部修改敏感，所以放到最终 latent 的空间/高频扰动；混合场景需要互相增强，所以设计 F2S 和 S2F。

d) 方法叙事顺序合理：先定义任务，再讲在线注入总流程，再讲离线优化，再讲 CAP 训练，最后讲验证。读者可以看到每个模块出现是为了解决哪一类冲突，而不是简单堆模块。

e) 实验组织与 claim 对齐：主表证明总体优越，高级攻击表证明版权鲁棒，定位表证明精细篡改定位，质量表证明视觉代价，混合退化图证明协同价值，消融逐项验证 Freq Adv、CAP、F2S、S2F。

f) 证据链最完整的是“CAP 提升定位”和“F2S/S2F 互相增强”。证据弱点是安全边界：如果攻击者知道频域覆盖、CAP 高频分布和 detector，能否训练去水印/反检测网络，论文没有充分实验。

g) 可借鉴写法：先用一个结构性矛盾定义问题，再让方法模块逐项解除矛盾；实验不只报告平均 SOTA，而是设置 combined degradation + tampering 这种能检验核心 insight 的场景。这种写法适合安全论文，因为它把 threat model、method design 和 evidence chain 连成了闭环。

## Overview.md 条目

- [ROBIN++: Unified Copyright Protection and Tamper Localization for Diffusion Models via Dual-Domain Synergistic Watermarking](./Other_Security/2026-ROBIN++.md)  
  *IEEE Transactions on Pattern Analysis and Machine Intelligence, 2026*  
  Citation: Huang, H., Zeng, S., Wang, Q., Du, B., & Wu, Y. “ROBIN++: Unified Copyright Protection and Tamper Localization for Diffusion Models via Dual-Domain Synergistic Watermarking.” *IEEE Transactions on Pattern Analysis and Machine Intelligence*, 2026.  
  Links: [DOI](https://doi.org/10.1109/TPAMI.2026.3723036) | [Code](https://github.com/Hannah1102/ROBIN) *(ROBIN++ code marked Coming soon in repository)*
