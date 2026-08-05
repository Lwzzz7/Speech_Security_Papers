# SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking

## 0. 论文基本信息

| 项目 | 信息 |
|---|---|
| 论文 | SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking |
| 作者 | Han Fang, Kejiang Chen, Zehua Ma, Jiajun Deng, Yicong Li, Weiming Zhang, Ee-Chien Chang |
| 机构 | National University of Singapore; University of Science and Technology of China |
| 会议 | IEEE/CVF International Conference on Computer Vision (ICCV), 2025 |
| 方向 | 生成式图像水印；inversion-based generative watermarking；几何鲁棒性 |
| 代码 | 未在论文/CVF页面或快速检索中发现公开官方代码链接 |

## 1. 论文要解决什么问题

这篇论文针对的是扩散图像生成模型中的 inversion-based 水印。已有方法通常把水印嵌入到扩散采样的初始噪声或潜变量中，再通过 DDIM inversion 等方式从可疑图像反推出潜变量并解码水印。这类方法对 JPEG、噪声、滤波、亮度变化等非几何失真通常比较强，但对旋转、平移、缩放、裁剪、shear 等几何失真很脆弱。

核心原因是：几何变换会改变图像坐标系，导致 inversion 反推到的潜变量与原始水印潜变量不再对齐。水印本身可能仍在图像中，但提取器看到的是错位后的证据，因此 bit accuracy 和 detection TPR 会显著下降。

SynTag 的目标不是重新设计一个完整水印编码器，而是在已有 inversion-based 水印框架外层加入一个可学习的同步标记机制：在生成图像中注入对几何变换敏感的隐形同步特征，让检测阶段先估计几何变换并校正图像，再执行原来的 inversion-based 水印提取。

## 2. 威胁模型解读

论文假设水印方控制生成模型的图像生成流程，可以在部署前替换或微调 VAE decoder，使生成图像同时携带原始 inversion-based 水印和 SynTag 同步特征。攻击者拿到生成图像后，可以执行常见传播与编辑操作，包括几何攻击和非几何攻击。

几何攻击包括旋转、平移、缩放、裁剪、padding、shear；非几何攻击包括 JPEG 压缩、高斯噪声、中值滤波、dropout、cropout、亮度变化等。论文还评估了自适应攻击，如自动编码器重构和 diffusion purification。

检测者已知水印密钥或目标 watermark sequence，需要判断可疑图像是否来自受保护模型；在 tracing 场景中，检测者面对多个源模型水印，需要从候选水印序列中找出最匹配的来源。论文使用固定 FPR 下的 TPR 和 bit accuracy 衡量鲁棒性。

需要注意的是，SynTag 保护的是生成式图像水印链路，而不是音频水印。它应归入 Other Security 或跨模态生成式水印部分，而不是语音水印主线。

## 3. 方法设计与复现级理解

### 3.1 总体思路：把“同步模板”接到 inversion-based 水印前面

SynTag 的基本判断是：inversion-based 水印不是没有信息冗余，而是几何攻击后缺少可靠的坐标同步。因此论文不直接修改 GauShad、TreeRings 这类水印机制，而是在图像生成端额外注入一种几何敏感的同步特征。检测端先用同步特征预测几何校正参数，把图像尽量拉回原始坐标系，然后再执行原水印方法的 inversion 和 message extraction。

完整流程分为三个阶段：

1. SynTag initialization：训练一个带同步特征的 VAE decoder 和一个几何参数预测网络。
2. Watermark injection：仍由原 inversion-based 方法生成带水印初始 latent，再通过扩散模型采样，最后用 SynTag decoder 解码成图像。
3. Watermark extraction：先预测并校正几何变换，再通过 dither compensation 生成多个候选输入，最后执行 DDIM inversion 和水印解码。

### 3.2 预备知识：DDIM inversion 为什么怕几何错位

论文沿用 latent diffusion 的 DDIM 采样形式。若用 $x_t$ 表示第 $t$ 步 latent，DDIM 反向生成可写为：

$$x_{t-1}=\sqrt{\frac{\alpha_{t-1}}{\alpha_t}}x_t-\left(\sqrt{\frac{\alpha_{t-1}(1-\alpha_t)}{\alpha_t}}+\sqrt{1-\alpha_{t-1}}\right)\epsilon_\theta(x_t,t)$$

为了做水印提取，检测端要近似反推生成过程：

$$x_t=\frac{x_{t-1}-b_t\epsilon_\theta(x_t,t)}{a_t}\approx\frac{x_{t-1}-b_t\epsilon_\theta(x_{t-1},t)}{a_t}$$

这里的关键不是公式本身，而是 inversion 假设输入图像在空间坐标上仍与生成图像对齐。旋转或缩放后，VAE encoder 得到的 latent 已经发生空间错位，后续 inversion 即使数值上能运行，也会把水印证据映射到错误位置。

### 3.3 几何校正建模：预测四个角点而不是直接预测攻击类型

SynTag 用 homography 描述几何变换：

$$H=\begin{bmatrix}h_{11}&h_{12}&h_{13}\\h_{21}&h_{22}&h_{23}\\h_{31}&h_{32}&h_{33}\end{bmatrix}$$

图像点变换写作：

$$\begin{bmatrix}u'\\v'\\1\end{bmatrix}=H\begin{bmatrix}u\\v\\1\end{bmatrix}$$

其中 $h_{33}$ 归一化为 1，剩余 8 个自由度可以覆盖 affine、translation 和 perspective 相关变换。论文不是让网络分类“这是旋转还是缩放”，而是固定原图四个角点 $(1,1),(1,A),(B,1),(B,A)$，让预测网络输出变换后的四个角点 $P=\{P_1,P_2,P_3,P_4\}$。有了这四个点，就可以构造对应 homography 并对图像做反向校正。

这种设计的优点是复现时不必为不同几何攻击写不同分支，统一用四点参数表示；缺点是预测误差会直接影响后续 watermark extraction，因此论文后面还加入了 dither compensation。

### 3.4 第一阶段：SynTag initialization

第一阶段只训练两个组件：SynTag decoder $D_{Syn}$ 和 SynTag predictor $P_{Syn}$。原始 VAE encoder $E_\Delta$ 和预训练 VAE decoder $D_\Delta$ 用来提供 clean latent 和重建参考。

给定训练图像 $I_O$，先由 VAE encoder 得到 latent：

$$z_O=E_\Delta(I_O)$$

原始 decoder 重建参考图像，SynTag decoder 输出带同步特征图像：

$$I_R=D_\Delta(z_O),\quad I_{Syn}=D_{Syn}(z_O)$$

然后对 $I_{Syn}$ 施加随机几何失真和非几何扰动，得到 $I_D$。训练时已知施加的几何变换，因此可以计算 ground-truth 四角点 $P_{gt}$。预测网络从失真图像中预测校正点：

$$P_p=P_{Syn}(I_D)$$

训练目标由两部分组成。第一部分约束 SynTag decoder 不破坏图像质量：

$$L_R(I_{Syn},I_R)=\lambda_1L_{MSE}(I_{Syn},I_R)+\lambda_2L_{VGG}(I_{Syn},I_R)+\lambda_3L_{SSIM}(I_{Syn},I_R)$$

第二部分约束预测网络正确恢复几何参数：

$$L_P=L_{MSE}(P_p,P_{gt})=L_{MSE}(P_{Syn}(I_D),P_{gt})$$

这一阶段训练完成后，$D_{Syn}$ 替换原始 VAE decoder。直观上，$D_{Syn}$ 学到的不是传统 bit message，而是一种对几何扰动敏感、但视觉上不可见的同步特征；$P_{Syn}$ 则学习如何从被攻击图像中读出这种同步特征并估计校正参数。

### 3.5 第二阶段：watermark injection

第二阶段接入已有 inversion-based 水印方法。给定 $l$ bit 水印序列 $s\in\{0,1\}^l$，原水印方法 $F$ 生成带水印的初始 latent：

$$z_T^s=F(s)$$

扩散模型从 $z_T^s$ 采样得到最终 latent $z_0^s$。与原方法不同的是，最后一步不用普通 VAE decoder，而是用 $D_{Syn}$ 生成图像：

$$I_s=D_{Syn}(z_0^s)$$

因此生成图像里同时存在两类信息：一类是原 inversion-based 水印方法负责的 message-level watermark，另一类是 $D_{Syn}$ 注入的 synchronization tag。前者用于检测和溯源，后者用于几何攻击后的校正。

论文主要把 SynTag 组合到 GauShad 和 TreeRings 上，得到 GauShad-SynTag 和 TreeRings-SynTag。GauShad-SynTag 使用 64-bit 水印；TreeRings 本身是 1-bit 检测方法，因此主要报告 TPR。

### 3.6 第三阶段：watermark extraction 与 dither compensation

检测端输入可能已经被攻击的图像 $I_D$。第一步由 $P_{Syn}$ 预测校正参数，得到 $P_p$，再通过 homography correction 得到校正图像：

$$I_C=T_{P_p}(I_D)$$

只做一次校正仍不够，因为预测网络输出的角点可能有小误差，VAE 编码和 DDIM inversion 也会放大这种误差。论文因此设计了两级 dither compensation。

像素级补偿 $C_p$ 在 $I_C$ 附近生成多个轻微几何扰动版本，例如旋转 $\pm3^\circ$、缩放 0.9 和 1.1。每个候选图像都进入 VAE encoder 和 DDIM inversion，形成多个候选 latent。

latent 级补偿 $C_l$ 对 inversion 后的 latent 做 zero padding 和滑窗裁剪。若 latent 尺寸为 $C\times H\times W$，padding 半径为 $r$，则得到尺寸 $C\times(H+2r)\times(W+2r)$ 的 padded latent，并通过步长为 1 的滑窗产生 $(2r+1)^2$ 个 latent 变体。每个 latent 都交给原水印解码器 $F^{-1}$ 得到候选消息。

最终候选集合记为 $S_{ex}=\{\bar{s}_1,\bar{s}_2,\ldots\}$。只要存在候选消息与目标水印相似度超过阈值，就判定水印存在：

$$S(\bar{s}_i,s)\ge\tau$$

这部分是 SynTag 鲁棒性提升的关键。$P_{Syn}$ 负责把大尺度几何变换校正回来，$C_p$ 和 $C_l$ 负责吸收小尺度残余误差。消融实验显示 latent 级补偿比像素级补偿贡献更大。

### 3.7 复现配置

论文在 Stable Diffusion v1.4 和 Stable Diffusion v2.1 上实验，生成图像分辨率为 $512\times512\times3$，latent 尺寸为 $4\times64\times64$。SynTag initialization 使用 MS COCO 训练图像。生成阶段使用 Stable-Diffusion Prompt dataset 的 prompt，采样步数为 50，guidance scale 为 7.5。

训练 SynTag 时的几何扰动包括：旋转 $-45^\circ$ 到 $45^\circ$，平移 10 到 75 pixels，缩放 0.5 到 2 并配合 padding/cropping，shear mapping 5 到 10 pixels。几何扰动后还会叠加非几何扰动，包括 Gaussian noise、median filtering 和 JPEG compression。

提取阶段使用 empty-condition DDIM inversion，步数为 20。像素级 dither compensation 包括旋转 $\pm3^\circ$ 和缩放 0.9/1.1；latent 级 compensation 的 padding 半径为 $r=2$。检测阈值按固定 FPR 选择，主实验设置 FPR 为 $10^{-6}$，对应阈值 $\tau=0.78125$。实现环境为 PyTorch 1.12.1 和单张 NVIDIA A40 GPU。

## 4. 实验设计与结果

### 4.1 对比方法和指标

论文比较了 9 类基线：post-hoc 水印包括 DwtDctSvd、RivaGAN、MBRS、StegaStamp、RoSteALS；fine-tune-based 方法包括 Stable Signature 和 LaWa；inversion-based 方法包括 TreeRings 和 Gaussian Shading。SynTag 主要作为插件式增强模块，与 GauShad 和 TreeRings 组合。

检测指标是固定 FPR 下的 TPR；对多 bit 水印还报告 bit accuracy；图像质量用 FID 和 CLIP-Score。主结果从随机 prompt 生成 50 张水印图像进行统计。

### 4.2 主结果：几何鲁棒性显著增强，非几何鲁棒性基本保持

在 Stable Diffusion v1.4/v2.1 上，原 GauShad 面对几何攻击时 TPR 只有 0.016/0.020，bit accuracy 只有 0.633/0.635；加入 SynTag 后，GauShad-SynTag 的几何攻击 TPR 提升到 0.980/0.988，bit accuracy 提升到 0.938/0.940。

TreeRings 也有类似提升。原 TreeRings 的几何攻击 TPR 为 0.548/0.552，TreeRings-SynTag 提升到 0.928/0.932。非几何攻击下，GauShad-SynTag 的 TPR 为 0.967/0.970，基本保持了 GauShad 原本的非几何鲁棒性。

图像质量方面，Stable Diffusion 原始 FID 约为 25.28，GauShad-SynTag 为 25.21，CLIP-Score 从 0.3628 附近变为 0.3615，说明 SynTag decoder 对视觉质量和文本一致性的影响较小。

### 4.3 更大范围几何攻击

论文进一步测试旋转 $-30^\circ$ 到 $30^\circ$、缩放 0.7 到 1.75、平移 20 到 75 pixels、shear 5 到 10 等连续强度范围。结果显示，随着几何攻击增强，原 inversion-based 方法迅速失效，而 SynTag 版本仍保持较高 TPR 和 bit accuracy。

这说明 SynTag 的收益不是针对某个固定攻击参数调出来的，而是来自显式同步校正机制。不过它仍依赖训练时见过的几何分布，若部署环境出现更极端的透视变换、非刚性变形或复杂局部编辑，仍需要重新评估。

### 4.4 自适应攻击

论文评估了两类自适应攻击。第一类是 autoencoder reconstruction，包括 Cheng、Bmshj、VQ-VAE、KL-VAE。GauShad-SynTag 在这些重构攻击下仍保持 0.920 到 0.980 的 TPR，bit accuracy 约为 0.966 到 0.992。

第二类是 purification attack。攻击强度 $f$ 从 0.1 增至 0.7 时，TPR 从 0.980 逐步下降到 0.000；当 $f=0.5$ 时 TPR 已降到 0.580，PSNR 约为 19.17 dB；当 $f=0.7$ 时 PSNR 约为 15.44 dB，图像质量已经明显受损。这个结果说明 SynTag 对温和重构和弱 purification 有鲁棒性，但强 purification 仍可能清除水印，代价是明显破坏图像。

### 4.5 dither compensation 的误报分析

dither compensation 会扩大候选集合，直觉上可能增加 false positive。论文用 1000 张水印图像和 1000 张非水印图像测试固定 watermark 下的 bit accuracy 分布。攻击包括旋转、缩放、JPEG 和高斯噪声。

结果显示，非水印图像即使用 dither compensation，最高 bit accuracy 也低于 0.8；水印图像几乎都高于 0.8。因此选择 $\tau=0.8$ 左右可以区分两类样本。这个实验很关键，因为 SynTag 的提取策略本质上是“多候选尝试”，必须证明它不会靠 brute-force 候选数把误报率抬高。

### 4.6 适配不同采样设置

论文测试了不同 sampling method、sampling steps、guidance scale 和 prompt set。DDIM、UniPC、PNDM 下 TPR 分别约为 0.990、0.960、0.970；采样步数 20、50、100 下 TPR 都在 0.980 到 0.990；guidance scale 3、7.5、11 下 TPR 为 1.000、0.990、0.980；不同 prompt set 下 TPR 也均高于 0.96。

这说明 SynTag 不只绑定某一个采样器或 prompt 分布。不过这些实验仍是在同一类 Stable Diffusion 生成链路中完成，跨模型架构和跨 VAE 的泛化还不是论文主要结论。

### 4.7 组合攻击

论文还测试了 10 组组合攻击，包括多个几何攻击组合，以及几何攻击叠加 JPEG、高斯噪声、中值滤波等非几何攻击。GauShad-SynTag 在这些设置下 TPR 全部不低于 0.92，bit accuracy 不低于 0.90。

这个结果支撑了论文的核心定位：SynTag 不是只对单一旋转或缩放有效，而是在几何校正后仍能把原 inversion-based 水印的非几何鲁棒性保留下来。

## 5. 消融实验解读

第一组消融验证主动注入 SynTag feature 是否必要。论文比较了只训练被动 restoration predictor 的方案 $P^-$，以及同时使用 $D_{Syn}$ 和 $P_{Syn}$ 的方案。面对旋转、缩放、平移、shear 等攻击，$P^-$ 平均 bit accuracy 约为 0.730，而完整 SynTag 约为 0.938。

这说明仅靠图像内容本身预测几何校正不够稳定。SynTag 的关键是生成端主动写入同步特征，使检测端可以从这些特征中恢复几何轨迹。

第二组消融验证校正组件贡献。无 $P_{Syn}$ 时，GauShad 在几何攻击下 TPR 只有 0.016，bit accuracy 0.633；加入 $P_{Syn}$ 后 TPR 到 0.763，bit accuracy 到 0.819；加入像素级补偿后 TPR 到 0.795，bit accuracy 到 0.865；加入 latent 级补偿后 TPR 到 0.930，bit accuracy 到 0.915；完整 $P_{Syn}+C_p+C_l$ 达到 TPR 0.990，bit accuracy 0.935。

这里可以看出三点：第一，几何参数预测是主增益来源；第二，latent 级补偿比像素级补偿更重要；第三，完整方法的高 TPR 来自“预测校正 + 多候选误差吸收”的组合，而不是某个单独模块。

## 6. 论文贡献总结

第一，论文明确指出 inversion-based generative image watermarking 的主要短板是几何同步，而不是水印容量或非几何鲁棒性本身。

第二，论文提出 SynTag，把同步模板思想引入 latent diffusion 生成水印，通过微调 VAE decoder 主动注入几何敏感特征，并训练 predictor 估计 homography correction。

第三，SynTag 能作为插件增强 GauShad、TreeRings 等已有 inversion-based 方法，在几何攻击下显著提高 TPR 和 bit accuracy，同时基本保持非几何鲁棒性和图像质量。

第四，论文用 dither compensation 解决预测误差残留问题，并通过误报分析说明多候选提取没有明显破坏固定 FPR 约束。

## 7. 局限性与风险

第一，SynTag 需要修改生成链路中的 VAE decoder，因此它适合模型拥有者主动部署，不适合无法控制生成模型的 post-hoc 图像水印场景。

第二，几何校正能力依赖训练时的攻击分布。论文覆盖了常见旋转、缩放、平移、shear，但复杂局部形变、强透视变换、局部重绘、裁剪后重新构图等场景仍可能超出能力边界。

第三，强 purification attack 可以在牺牲图像质量的前提下显著削弱水印。论文结果显示，当 purification 强度足够大时，水印检测会失败，只是攻击后的视觉质量也会明显下降。

第四，SynTag 引入了额外训练、替换 decoder 和多候选提取开销。尤其是 dither compensation 会增加检测阶段计算量，部署时需要在检测延迟和鲁棒性之间权衡。

第五，论文主要在 Stable Diffusion v1.4/v2.1 上实验。对于 SDXL、DiT、flow matching 或其他生成架构，能否直接复用同一 SynTag decoder 设计仍需实验验证。

## 8. 与语音安全/水印研究的关系

虽然这篇是图像生成水印论文，但它对语音生成水印有直接启发：很多音频水印方法在重采样、裁剪、时间伸缩、VC/TTS 重合成后失败，本质上也可能是同步丢失而不只是 bit embedding 不够强。

SynTag 的思想可以类比到音频中：生成端注入一种对时间轴扰动敏感但感知不可见的同步特征，检测端先估计时间偏移、速度变化或局部裁剪位置，再进行水印提取。对应到语音，homography correction 不能直接复用，但“主动同步特征 + 校正网络 + 多候选补偿”的范式值得参考。

不过音频的同步扰动更复杂，尤其是语音内容可被 ASR-TTS 或 VC 重合成后重新生成，时间轴和声学细节都会变化。因此 SynTag 更适合作为生成式水印鲁棒性设计的参考，而不是可以直接迁移的方案。

## 9. 精读结论

SynTag 是一篇目标非常明确的生成式图像水印论文：它不追求提出全新的 bit embedding 机制，而是补齐 inversion-based 水印在几何攻击下的同步短板。方法上的关键是用微调后的 VAE decoder 主动注入 invisible synchronization tag，再用 predictor 和 dither compensation 在检测前恢复坐标对齐。

从结果看，SynTag 对 GauShad 和 TreeRings 的增强非常明显，尤其是 GauShad-SynTag 在几何攻击下把 TPR 从接近失效提升到约 0.98，同时保持非几何鲁棒性和图像质量。论文最值得关注的不是某个具体网络结构，而是它把“水印提取前的几何同步”作为独立问题系统处理，这一点对图像、视频、音频生成水印都有借鉴意义。

## 10. 可放入 Overview.md 的条目

- [SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking](./Other_Security/2025-SynTag.md)  
  *IEEE/CVF International Conference on Computer Vision (ICCV), 2025*  
  Citation: Fang, H., Chen, K., Ma, Z., Deng, J., Li, Y., Zhang, W., & Chang, E.-C. “SynTag: Enhancing the Geometric Robustness of Inversion-based Generative Image Watermarking.” *Proceedings of the IEEE/CVF International Conference on Computer Vision*, 2025.  
  Links: [Paper](https://openaccess.thecvf.com/content/ICCV2025/html/Fang_SynTag_Enhancing_the_Geometric_Robustness_of_Inversion-based_Generative_Image_Watermarking_ICCV_2025_paper.html)

## 11. 后续复现建议

如果后续要复现 SynTag，建议先不要从完整 GauShad-SynTag 开始，而是先单独复现 SynTag initialization：固定 Stable Diffusion VAE encoder/decoder，训练 $D_{Syn}$ 和 $P_{Syn}$，验证 $P_{Syn}$ 对旋转、缩放、平移、shear 的角点预测误差。只有几何校正稳定后，再接入 GauShad 或其他 inversion-based watermark。

评估时应单独报告三组结果：无校正的原方法、只加 $P_{Syn}$ 的校正版本、完整 $P_{Syn}+C_p+C_l$ 版本。这样才能判断收益来自同步特征、校正网络还是候选补偿。若迁移到音频或视频任务，也应保留这种分解评估方式。
