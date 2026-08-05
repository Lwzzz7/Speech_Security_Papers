# SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking

## 0. 翻译摘要原文

生成式图像水印的鲁棒性通常通过提取抗失真的水印特征来实现。近期处于领先地位的 inversion-based 生成式水印框架在非几何失真下表现出较强鲁棒性，但在面对几何失真时仍然容易失败。为了解决这一问题，作者提出 SynTag，通过向生成图像中注入 synchronization tag 特征来增强几何鲁棒性。SynTag 特征对几何失真敏感，会随几何变换发生可预测变化，因此检测阶段可以利用这些特征重建失真轨迹，并在提取水印前对图像进行几何校正。论文聚焦 latent diffusion models，具体做法是微调 VAE decoder，使其在生成图像中嵌入不可见的 SynTag 特征，同时训练一个预测网络从失真图像中提取 SynTag 并估计几何校正参数。进一步地，作者提出 dither compensation 来提升校正精度。实验表明，SynTag 可以兼容已有 inversion-based 水印方法，在保持非几何鲁棒性和图像质量的同时，显著提高旋转、平移、缩放、shear 等几何攻击下的检测率和 bit accuracy。

## 1. 方法动机

a) 作者提出 SynTag 的直接动机，是 inversion-based generative watermarking 在几何失真下存在系统性短板。已有方法如 TreeRings、Gaussian Shading 通过修改扩散初始噪声或 latent 分布来携带水印，检测时再用 DDIM inversion 反推 latent 并解码水印。这种范式对 JPEG、噪声、滤波等非几何攻击较强，但一旦图像被旋转、缩放、平移或 shear，反推 latent 的空间对齐关系就会被破坏。

b) 现有方法的痛点不是没有把水印嵌入进去，而是检测端缺少同步机制。几何攻击改变图像坐标系后，水印仍可能残留在图像里，但提取器从错误坐标位置读取证据，导致 bit accuracy 和 detection TPR 下降。传统后处理水印中常见的同步模板思想，在当前 latent diffusion inversion-based 水印中没有被充分利用。

c) 论文的核心 insight 是：与其让水印本身同时承担“携带消息”和“抵抗几何错位”两件事，不如额外注入一种对几何变换敏感的隐形同步特征。检测端先利用该同步特征估计几何变换，把图像校正回接近原始坐标系，再执行原 inversion-based 水印提取。SynTag 因此更像是一个几何同步增强模块，而不是替代原水印算法的新编码器。

## 2. 威胁模型解读

a) 参与方与系统边界：防御者是生成模型或水印系统的部署者，能够控制 latent diffusion 生成流程，并在部署前微调/替换 VAE decoder。用户或外部传播链路可能拿到生成图像并进行编辑。检测者拥有目标 watermark sequence 或候选水印集合，用于判定图像是否由受保护模型生成，或进一步追踪来源模型。

b) 攻击者目标：攻击者希望在尽量保持图像可用性和视觉质量的前提下破坏水印检测，尤其是让检测端无法从几何变换后的图像中恢复正确 watermark bits。论文关注的攻击目标主要是水印去除/规避检测，而不是伪造另一个来源水印。

c) 攻击者能力：攻击者可以对生成图像执行后处理和传播编辑，包括 rotation、translation、scaling、cropping/padding、shear、JPEG compression、Gaussian noise、median filtering、dropout、cropout、brightness adjustment。论文还评估 autoencoder reconstruction 和 purification attack，属于更强的内容重构式攻击。

d) 攻击者知识：主实验更接近常规水印鲁棒性评估，不要求攻击者知道 SynTag 的训练细节、decoder 参数或检测阈值。自适应攻击部分用重构和 purification 检验更强移除能力，但论文没有展开完整白盒攻击者可访问 $D_{Syn}$、$P_{Syn}$ 或密钥时的最坏情况分析。

e) 攻击约束：攻击应尽量保持图像语义和视觉质量。论文用 FID、CLIP-Score、PSNR 等指标约束质量变化。强 purification 可以破坏水印，但在高强度下 PSNR 降到约 15–19 dB，图像质量也明显下降。

f) 防御者能力与安全目标：防御者可以在生成阶段部署 SynTag decoder，并在检测阶段运行 $P_{Syn}$、几何校正、dither compensation、DDIM inversion 和原水印解码器。安全目标是在固定 FPR 下提高几何攻击后的 TPR 和 bit accuracy，同时不明显牺牲非几何鲁棒性与生成图像质量。

g) 现实性评估：这个 threat model 适合模型拥有者主动部署水印的场景，例如受控 Stable Diffusion 服务或生成模型发行方。它不适合不能控制生成过程的纯 post-hoc 图像水印。另一个边界是几何攻击分布主要由训练时的旋转、缩放、平移、shear 覆盖，复杂局部形变、局部重绘、强裁剪后重构等不一定被充分覆盖。

h) 与后文关系：SynTag 的三个组件分别对应 threat model 中的关键问题：$D_{Syn}$ 负责在生成端主动注入同步特征，$P_{Syn}$ 负责估计攻击造成的几何变换，$C_p/C_l$ 负责吸收残余校正误差。实验中的几何攻击、组合攻击和自适应攻击分别验证这些设计是否覆盖主要威胁。

## 3. 方法设计与复现级理解

### 3.1 全局流程图式说明

SynTag 的执行链路可以分成训练、生成和检测三段。

训练阶段：输入普通训练图像，先用原 Stable Diffusion VAE encoder 得到 clean latent，再分别用原 VAE decoder 和待训练 SynTag decoder 解码。SynTag decoder 的输出会经过随机几何失真和非几何噪声层，形成 distorted image。训练目标一方面要求 SynTag decoder 的输出接近原 decoder 重建图像，另一方面要求 SynTag predictor 从 distorted image 中预测出几何校正用的四个角点。

生成阶段：原 inversion-based 水印方法先把 message $s$ 编入扩散初始 latent $z_T^s$，扩散采样得到 $z_0^s$，再由 $D_{Syn}$ 解码成图像 $I_s$。这张图像同时携带原水印 message 和 SynTag 同步特征。

检测阶段：输入可疑图像 $I_D$，先由 $P_{Syn}$ 预测几何校正参数，用 homography 得到校正图像 $I_C$。之后执行像素级和 latent 级 dither compensation，生成多个候选图像/latent，再分别做 VAE encoding、DDIM inversion 和原水印解码。只要某个候选 message 与目标水印相似度超过阈值，就判定水印存在。

### 3.2 DDIM inversion 与几何错位问题

论文沿用 latent diffusion 的 DDIM 采样与反演。生成阶段从噪声 latent 逐步去噪得到最终 latent；检测阶段则从图像编码回 latent 后，近似反推扩散过程，恢复初始 latent 中的水印结构。

DDIM 反向采样可写为：

$$x_{t-1}=\sqrt{\frac{\alpha_{t-1}}{\alpha_t}}x_t-\left(\sqrt{\frac{\alpha_{t-1}(1-\alpha_t)}{\alpha_t}}+\sqrt{1-\alpha_{t-1}}\right)\epsilon_\theta(x_t,t)$$

对应的 inversion 近似为：

$$x_t=\frac{x_{t-1}-b_t\epsilon_\theta(x_t,t)}{a_t}\approx\frac{x_{t-1}-b_t\epsilon_\theta(x_{t-1},t)}{a_t}$$

这个模块解决的问题是从图像侧恢复水印 latent，但它隐含假设输入图像仍与生成坐标系对齐。旋转、缩放、平移后，VAE encoder 得到的 latent 空间位置已经改变，因此 inversion 会在错误位置寻找水印证据。SynTag 后续所有设计都围绕“先恢复坐标同步，再做 inversion”展开。

### 3.3 SynTag initialization：训练同步特征注入器和几何预测器

这一阶段解决的问题是：如何让生成图像中出现一种不可见但可被检测端用于几何校正的信号。论文不直接训练一个传统 bit encoder，而是把原 VAE decoder 复制成可训练版本 $D_{Syn}$，并训练一个预测网络 $P_{Syn}$。

输入训练图像 $I_O$ 后，固定 VAE encoder 先得到 latent：

$$z_O=E_\Delta(I_O)$$

原 decoder 给出重建参考，SynTag decoder 给出带同步特征的图像：

$$I_R=D_\Delta(z_O),\quad I_{Syn}=D_{Syn}(z_O)$$

然后对 $I_{Syn}$ 施加随机几何变换和非几何噪声，得到 distorted image $I_D$。由于训练时知道施加的几何变换，可以计算 ground-truth 四角点 $P_{gt}$。预测网络输出：

$$P_p=P_{Syn}(I_D)$$

这里预测四个角点而不是直接预测角度、缩放率或平移量，是因为 homography 可以统一表示多种几何变换。论文使用的 homography 矩阵为：

$$H=\begin{bmatrix}h_{11}&h_{12}&h_{13}\\h_{21}&h_{22}&h_{23}\\h_{31}&h_{32}&h_{33}\end{bmatrix}$$

点坐标变换为：

$$\begin{bmatrix}u'\\v'\\1\end{bmatrix}=H\begin{bmatrix}u\\v\\1\end{bmatrix}$$

$h_{33}$ 归一化后剩余 8 个自由度，刚好可以由四个角点约束。复现时需要固定原图角点，例如 $(1,1),(1,A),(B,1),(B,A)$，再让网络预测变换后的四点 $P=\{P_1,P_2,P_3,P_4\}$。

训练损失有两类。第一类是图像重建/感知质量约束，防止 $D_{Syn}$ 为了注入同步特征破坏视觉质量：

$$L_R(I_{Syn},I_R)=\lambda_1L_{MSE}(I_{Syn},I_R)+\lambda_2L_{VGG}(I_{Syn},I_R)+\lambda_3L_{SSIM}(I_{Syn},I_R)$$

第二类是几何预测损失：

$$L_P=L_{MSE}(P_p,P_{gt})=L_{MSE}(P_{Syn}(I_D),P_{gt})$$

一次训练迭代的真实数据流是：采样训练图像 -> VAE encode -> $D_\Delta$ 和 $D_{Syn}$ 分别 decode -> 对 $D_{Syn}$ 输出加几何/非几何扰动 -> $P_{Syn}$ 预测角点 -> 计算重建损失和预测损失 -> 反传更新 $D_{Syn}$ 与 $P_{Syn}$。训练完成后，$D_{Syn}$ 会替换部署中的 VAE decoder。

### 3.4 Watermark injection：把 SynTag 接到已有 inversion-based 方法上

这一阶段的输入是待嵌入的 watermark sequence $s\in\{0,1\}^l$。原 inversion-based 方法 $F$ 先把水印写入扩散起始 latent：

$$z_T^s=F(s)$$

扩散模型从 $z_T^s$ 经过标准采样得到最终 latent $z_0^s$。与 GauShad 或 TreeRings 原流程的区别只在最后解码器：SynTag 使用训练好的 $D_{Syn}$ 输出图像：

$$I_s=D_{Syn}(z_0^s)$$

这一步的接口设计比较干净：$F$ 仍负责 message-level watermark，$D_{Syn}$ 负责 synchronization tag。也就是说，SynTag 不需要重写 GauShad 的消息编码逻辑，也不需要改变 TreeRings 的基本检测形式。论文实验中 GauShad-SynTag 使用 64-bit 水印；TreeRings-SynTag 由于 TreeRings 本身偏 1-bit 检测，因此主要报告 TPR。

复现时要注意，$D_{Syn}$ 在 injection 阶段应固定，不再继续训练；否则生成图像的同步特征分布和检测端 $P_{Syn}$ 训练时看到的分布可能不一致。

### 3.5 Watermark extraction：先校正，再反演，再解码

检测阶段输入是可能被攻击后的图像 $I_D$。第一步用 $P_{Syn}$ 预测角点 $P_p$，再由 homography correction 得到校正图像：

$$I_C=T_{P_p}(I_D)$$

如果 $P_{Syn}$ 预测完全准确，后续可以直接把 $I_C$ 编码回 latent 并执行 DDIM inversion。但实际预测会有残余误差，且 VAE encoder 和 DDIM inversion 对小错位比较敏感。因此论文加入两级 dither compensation。

像素级补偿 $C_p$ 在 $I_C$ 附近做小范围几何扰动，例如旋转 $\pm3^\circ$、缩放 0.9 和 1.1，生成一组候选校正图像 $I_p=\{I_p^1,\ldots,I_p^n\}$。每个候选图像都进入 VAE encoder 和 DDIM inversion，得到候选 latent。

latent 级补偿 $C_l$ 进一步处理 inversion 后的 latent。若 latent 尺寸为 $C\times H\times W$，先在空间维度 zero padding 到 $C\times(H+2r)\times(W+2r)$，再用步长 1 的滑动窗口裁出 $(2r+1)^2$ 个 latent 变体。每个 latent 变体都输入原水印提取函数 $F^{-1}$，得到候选消息集合：

$$S_{ex}=\{\bar{s}_1,\bar{s}_2,\ldots\}$$

最终检测规则是，只要某个候选消息与目标水印相似度超过阈值，就认为检测成功：

$$S(\bar{s}_i,s)\ge\tau$$

这套提取流程的关键不是单次预测，而是“粗校正 + 局部搜索”。$P_{Syn}$ 把大范围几何错位拉回可处理范围，$C_p$ 和 $C_l$ 再吸收角点预测、VAE 编码和 inversion 引入的小偏差。

### 3.6 训练、构造与推理分离

训练阶段只需要普通图像数据和随机失真层，目标是得到 $D_{Syn}$ 与 $P_{Syn}$。这一阶段不需要真实 watermark message，也不需要运行完整 GauShad/TreeRings。

部署生成阶段需要运行原 inversion-based watermark encoder $F$、扩散采样器和 $D_{Syn}$。此时 $P_{Syn}$ 不参与生成。

检测阶段需要运行 $P_{Syn}$、homography correction、dither compensation、VAE encoder、DDIM inversion 和 $F^{-1}$。此时 $D_{Syn}$ 不参与检测，但检测效果依赖生成图像中由 $D_{Syn}$ 注入的同步特征。

这个分离很重要：SynTag 不是每一步扩散都注入 watermark，而是在最终 VAE decode 阶段加入同步特征；message watermark 仍由原 inversion-based 方法在初始 latent 层负责。

### 3.7 复现配置表

| 项目 | 论文设置 |
|---|---|
| 基座模型 | Stable Diffusion v1.4 和 Stable Diffusion v2.1 |
| 图像尺寸 | $512\times512\times3$ |
| latent 尺寸 | $4\times64\times64$ |
| SynTag 训练数据 | MS COCO |
| prompt 数据 | Stable-Diffusion Prompt dataset |
| 采样设置 | 50 steps，guidance scale 7.5 |
| 提取设置 | empty-condition DDIM inversion，20 steps |
| 训练几何扰动 | rotation $-45^\circ$ 到 $45^\circ$，translation 10–75 pixels，scale 0.5–2，shear mapping 5–10 pixels |
| 训练非几何扰动 | Gaussian noise，median filtering，JPEG compression |
| pixel compensation | rotation $\pm3^\circ$，scaling 0.9/1.1 |
| latent compensation | padding radius $r=2$ |
| 阈值 | FPR $10^{-6}$ 下 $\tau=0.78125$ |
| 硬件/框架 | PyTorch 1.12.1，单张 NVIDIA A40 GPU |
| 代码 | 论文内容中未提供官方代码链接 |

### 3.8 复现风险与信息缺口

论文给出了主流程、攻击类型、阈值和部分训练配置，但仍有几个复现风险。第一，$D_{Syn}$ 和 $P_{Syn}$ 的具体网络细节、训练 epoch、batch size、优化器超参数和损失权重没有在主文中完全展开。第二，随机失真层的组合顺序和采样概率会影响 $P_{Syn}$ 泛化。第三，dither compensation 的候选枚举数量会影响检测开销和误报控制。第四，GauShad/TreeRings 与 SynTag 组合时的工程接口没有官方代码支持，复现者需要自己处理 latent、VAE、inversion 和 watermark decoder 的一致性。

## 4. 与其他方法对比

| 方法类别 | 代表方法 | 核心思想 | 优点 | 缺点 | SynTag 的改进点 |
|---|---|---|---|---|---|
| post-hoc image watermark | DwtDctSvd, RivaGAN, MBRS, StegaStamp, RoSteALS | 在生成后图像上嵌入水印 | 可脱离生成模型使用 | 对强编辑或生成式重构未必稳定 | SynTag 面向生成链路，利用模型内部 VAE decoder 注入同步特征 |
| fine-tune-based watermark | Stable Signature, LaWa | 微调生成模型或 decoder，让输出携带水印 | 可与模型分发结合 | 可能影响生成质量，且几何鲁棒性有限 | SynTag 专门为几何校正设计同步特征 |
| inversion-based watermark | TreeRings, Gaussian Shading | 修改扩散初始 latent/noise，检测时反演提取 | 非几何鲁棒性强，适合生成式水印 | 几何攻击后 latent 对齐失败 | SynTag 在提取前先恢复坐标同步 |
| SynTag | GauShad-SynTag, TreeRings-SynTag | 微调 VAE decoder 注入同步特征，并用 predictor 估计几何校正 | 显著增强几何攻击鲁棒性，兼容已有 inversion-based 方法 | 需要控制生成模型 decoder，检测开销更高 | 把“几何同步”从水印 message 中解耦出来 |

本质区别在于，SynTag 不把几何鲁棒性完全寄托在 watermark bits 本身，而是显式引入同步通道。它更像是给 inversion-based 水印加了一个 alignment layer。

## 5. 实验表现与优势

a) 主实验验证 SynTag 是否提升几何鲁棒性。论文在 Stable Diffusion v1.4/v2.1 上测试 9 类基线，指标包括固定 FPR 下的 TPR、bit accuracy、FID 和 CLIP-Score。几何攻击包括 rotation 30°、translation 50 pixels、scale 0.75 with padding、scale 1.25 with cropping、shear mapping 5 pixels；非几何攻击包括 JPEG QF=15、Gaussian noise $\sigma=0.05$、median filtering $k=11$、dropout、cropout 和 brightness adjustment。

b) 主结果显示，原 GauShad 在几何攻击下 TPR 只有 0.016/0.020，bit accuracy 为 0.633/0.635；加入 SynTag 后，GauShad-SynTag 的几何攻击 TPR 达到 0.980/0.988，bit accuracy 达到 0.938/0.940。TreeRings 的几何攻击 TPR 从 0.548/0.552 提升到 0.928/0.932。非几何攻击下，GauShad-SynTag 的 TPR 为 0.967/0.970，基本保持 GauShad 原有优势。

c) 图像质量方面，原 Stable Diffusion 的 FID 约为 25.28，GauShad-SynTag 的 FID 约为 25.21，CLIP-Score 从约 0.3628 降到约 0.3615。这个结果说明 SynTag decoder 没有明显破坏图像分布质量和文本图像一致性。

d) 自适应攻击中，autoencoder reconstruction 下 GauShad-SynTag 仍保持 0.920–0.980 的 TPR 和约 0.966–0.992 的 bit accuracy。purification attack 更强，随着攻击强度 $f$ 从 0.1 增加到 0.7，TPR 从 0.980 降到 0.000；但高强度 purification 的 PSNR 也下降到约 15–19 dB，说明攻击代价明显。

e) 消融实验验证了两个核心模块。只做被动 restoration predictor 的 $P^-$ 平均 bit accuracy 约为 0.730，而完整 $D_{Syn}+P_{Syn}$ 约为 0.938，说明主动注入同步特征是必要的。校正组件消融中，无 $P_{Syn}$ 时 TPR 仅 0.016；加入 $P_{Syn}$ 后到 0.763；加入 $P_{Syn}+C_l$ 后到 0.930；完整 $P_{Syn}+C_p+C_l$ 达到 0.990，说明 latent-level compensation 对残余错位修正贡献最大。

f) threat model 覆盖情况较完整：论文覆盖了常见几何攻击、非几何攻击、组合攻击、不同采样器/采样步数/guidance scale/prompt set，以及部分自适应重构攻击。缺口主要是没有充分评估白盒攻击者针对 $D_{Syn}$ 或 $P_{Syn}$ 反向优化去除同步特征的情况。

## 6. 学习与应用

a) 论文未提供可确认的官方开源代码。若要复现，最低可行路线是先训练 SynTag initialization：固定 Stable Diffusion VAE encoder/decoder，复制 decoder 得到 $D_{Syn}$，训练 $P_{Syn}$ 预测几何变换后的四角点。只有角点预测稳定后，再接入 GauShad 或 TreeRings。

b) 实现时最需要注意的是三类超参数：随机几何扰动分布、重建损失与预测损失权重、dither compensation 候选数量。扰动太弱会导致几何泛化不足；扰动太强会让 $D_{Syn}$ 难以保持不可见性；候选数量太少会漏检，太多会增加检测延迟并可能影响 FPR。

c) 迁移到其他任务时，SynTag 的思想比具体 homography 公式更重要。对视频可以考虑时空同步特征；对音频生成水印可以考虑时间轴同步特征，用于处理裁剪、时间伸缩、重采样或局部错位。不过音频中的 VC/TTS 重合成会改变声学细节，不能直接套用图像 homography。

## 7. 总结

a) 一句话概括：用隐形同步特征修正几何错位。

SynTag 的贡献在于把 inversion-based 生成式水印的几何脆弱性重新表述为同步问题，并给出一个可插拔的同步增强模块。它不替代 GauShad/TreeRings 的 message embedding，而是在 VAE decoder 端注入几何敏感特征，让检测端先恢复坐标再提取水印。

## 8. 图表精读与证据链

Table 1 是主证据链，证明 SynTag 对几何鲁棒性的提升不是牺牲非几何鲁棒性换来的。GauShad 几何攻击下几乎失效，而 GauShad-SynTag 达到约 0.98 TPR；同时非几何攻击 TPR 仍约 0.97，FID/CLIP-Score 也接近原 Stable Diffusion。

Figure 4 测试更连续的几何攻击强度，包括旋转、缩放、平移和 shear。它支持的 claim 是 SynTag 不是只对单一固定参数有效，而是在一段几何攻击范围内保持稳定。

Table 2 对应自适应攻击。autoencoder reconstruction 下结果较强，purification attack 在高强度下可以破坏水印，但会带来明显图像质量下降。这个表说明 SynTag 对温和重构有一定抵抗力，但不是不可移除水印。

Table 3 评估不同 sampling method、sampling steps、guidance scale 和 prompt set。它支持 SynTag 对生成配置变化有一定适配性，但证据仍局限在 Stable Diffusion v1.4/v2.1 的体系内。

Table 4 是组合攻击证据，说明几何校正后，原 inversion-based 水印的非几何鲁棒性还能继续发挥作用。

Table 5 和 Table 6 是最重要的消融证据。Table 5 说明主动 SynTag feature injection 比被动几何恢复更有效；Table 6 说明 $P_{Syn}$ 是主增益来源，而 $C_l$ 比 $C_p$ 更能补偿残余错位。

## 9. 复现难度与适合人群

复现难度：高。原因是该方法需要同时掌握 Stable Diffusion VAE、DDIM inversion、inversion-based watermark、homography correction、随机失真训练和多候选检测。缺少官方代码时，工程细节对结果影响较大。

主要依赖：Stable Diffusion v1.4/v2.1、MS COCO、VAE encoder/decoder 权重、GauShad 或 TreeRings 的实现、PyTorch、较高显存 GPU，以及稳定的几何攻击评估脚本。

最小可复现版本：先不接 GauShad，单独训练 $D_{Syn}$ 和 $P_{Syn}$，测试不同几何攻击下四角点预测误差和校正后图像对齐程度。第二步再接入一个已有 inversion-based watermark，比较无校正、只加 $P_{Syn}$、完整 $P_{Syn}+C_p+C_l$ 三种版本。

适合人群：适合研究生成式水印、扩散模型安全、图像版权保护和鲁棒检测的研究者；也适合想把同步机制迁移到视频/音频水印的人阅读。不适合作为初学者第一篇扩散水印复现论文。

## 10. 简短全面总结

SynTag 研究的是 latent diffusion 生成式图像水印在几何攻击下的鲁棒性问题。已有 inversion-based 方法通过修改扩散初始 latent 嵌入水印，面对 JPEG、噪声、滤波等非几何攻击较强，但旋转、缩放、平移、shear 会破坏图像与 latent 的空间对齐，使 DDIM inversion 难以正确提取水印。SynTag 的核心方法是在 VAE decoder 中注入不可见的 synchronization tag，并训练 $P_{Syn}$ 从失真图像中预测 homography 校正参数；检测时先校正图像，再用像素级和 latent 级 dither compensation 枚举残余错位候选，最后执行原水印解码。实验显示，GauShad-SynTag 在几何攻击下将 TPR 从接近失效提升到约 0.98，同时基本保持非几何鲁棒性和图像质量。主要局限是需要控制生成模型 decoder，缺少官方代码，且对白盒自适应去除同步特征的攻击分析仍不充分。

## 11. 论文写作逻辑分析

a) Intro 的问题铺垫比较清晰：作者先承认 inversion-based generative watermarking 在非几何攻击下已经很强，再指出其被几何攻击击穿的具体原因是 spatial misalignment。这样避免了泛泛说“鲁棒性不足”，而是锁定到同步缺失。

b) 核心 insight 从失败机制自然推出：如果几何攻击导致提取位置错位，那么解决方案应该是恢复坐标对齐，而不是简单加大水印强度。SynTag 因此把 synchronization tag 作为独立模块引入。

c) threat model 与方法承接较紧：攻击者能做几何/非几何编辑，防御者能控制生成 decoder；所以方法选择在 VAE decoder 端注入同步特征，并在检测端先估计几何校正。这个设定解释了为什么 SynTag 不是 post-hoc 水印。

d) 方法叙事顺序合理：先训练同步特征，再生成带水印图像，最后检测和补偿。尤其是 dither compensation 的出现有明确动机，即 $P_{Syn}$ 不可能完全消除残余误差。

e) 实验呼应较完整：主实验验证几何鲁棒性，非几何攻击验证是否保持原 inversion-based 优势，组合攻击验证实际编辑链路，自适应攻击验证更强移除风险，消融实验验证同步注入和补偿模块的必要性。

f) 证据链中最强的是 Table 1、Table 5 和 Table 6，分别对应主性能、主动同步特征必要性和模块贡献。相对不足的是白盒自适应攻击与跨生成架构泛化，论文没有充分证明攻击者知道 SynTag 机制后无法专门去除同步特征。

g) 可借鉴写法是：先把已有方法的失败定位为一个具体机制问题，再提出一个与失败机制直接对应的模块，最后用主结果、泛化设置、组合攻击和消融实验逐层闭环。对于安全论文，这种“failure mechanism -> defense module -> threat-aligned evaluation”的结构很值得参考。

## Overview.md 条目

- [SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking](./Other_Security/2025-SynTag.md)  
  *IEEE/CVF International Conference on Computer Vision (ICCV), 2025*  
  Citation: Fang, H., Chen, K., Ma, Z., Deng, J., Li, Y., Zhang, W., & Chang, E.-C. “SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking.” *Proceedings of the IEEE/CVF International Conference on Computer Vision*, 2025.  
  Links: [Paper](https://openaccess.thecvf.com/content/ICCV2025/html/Fang_SynTag_Enhancing_the_Geometric_Robustness_of_Inversion-based_Generative_Image_Watermarking_ICCV_2025_paper.html)
