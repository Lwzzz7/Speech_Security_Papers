# An Efficient Watermarking Method for Latent Diffusion Models via Low-Rank Adaptation and Dynamic Loss Weighting

## 0. 翻译摘要原文

深度神经网络的快速扩散推动了模型水印技术的发展，因为训练好的模型本身构成了有价值的知识产权。当前水印方法主要关注修改模型参数或改变采样行为。然而，随着模型规模持续增长，提高水印嵌入效率对于管理不断增加的计算需求变得至关重要。优先考虑效率不仅能优化资源利用，使水印过程更适用于大模型，也能缓解模型性能可能下降的问题。本文提出一种面向 Latent Diffusion Models 的高效水印方法，基于 Low-Rank Adaptation。其核心思想是在冻结的 LDM 中引入可训练低秩参数来嵌入水印，从而保持原始模型权重的完整性。此外，作者设计了 Dynamic Loss Weight Scheduler，用于自适应平衡水印不可感知性和水印嵌入目标，使模型能够在尽量不影响输出图像质量的情况下实现有效水印嵌入。实验结果表明，该方法能够快速、准确地嵌入和提取水印，同时保持高质量图像生成，并且在不同 payload 设置下仍然有效。它在鲁棒性上与现有 SOTA 方法相当，在某些情况下甚至更优。此外，该方法在不同数据集和多个具有兼容 VAE 解码模块的 LDM backbone 上也保持有效。

## 1. 方法动机

a) 作者提出 EW-LoRA 的背景是，LDM 这类生成模型已经成为商业图像生成服务的核心资产。模型权重、服务接口或下游生成结果都可能被非法复制、转售或滥用，因此需要一种 box-free ownership verification 方法：验证者不访问模型参数，只通过模型生成图像来判断某个模型是否来自受保护源。

b) 现有 LDM 水印方法存在效率和不可感知性问题。Full Parameter Watermarking 需要训练或微调整个 U-Net/VAE decoder，随着 SD3.5-L 等大模型规模增长，成本越来越高。Additional Parameter Watermarking 虽然减少了训练量，但现有方法如 AquaLoRA 在 denoising module 中插入较大 rank 的 LoRA，参数量仍大，并可能改变生成语义；LaWa、WMAdapter 等 adapter 类方法也引入额外结构，效率和部署复杂度仍不理想。

c) 论文的核心 insight 是：既然 ownership verification 最终从生成图像中提取水印，那么低秩水印载体应尽量放在视觉输出更近的 VAE decoder，而不是 denoising backbone。通过只在 VAE decoder 的关键层插入低秩 LoRA，EW-LoRA 能用极少训练参数把 watermark signal 写入生成图像，同时保持原模型冻结。训练时再用 DLWS 动态调节 image loss 和 watermark loss，避免固定 loss 权重导致收敛慢或图像质量下降。

## 2. 威胁模型解读

a) 参与方与系统边界：模型所有者持有原始 LDM，并在发布前通过 EW-LoRA 嵌入目标 watermark bits。用户只能通过在线服务、模型 hub 或推理接口提交 prompt 并获得生成图像。验证者同样只能观察生成图像，使用预训练 watermark decoder 提取 bits 并做所有权假设检验。模型参数在 threat model 中不可访问，因此是 box-free 设定。

b) 攻击者目标：恶意用户希望移除、削弱或覆盖水印，使生成图像无法证明来自受保护模型；或者通过覆盖另一个水印来伪造所有权。论文覆盖的攻击包括常规图像后处理、watermark overwriting 和 learning-based compression/removal。局限部分还讨论继续微调模型的 model-level attack。

c) 攻击者能力：攻击者能对输出图像做 center crop、rotation、resize、brightness/contrast/sharpness adjustment、JPEG compression，也能使用 StegaStamp 或 SSL-Watermarking 给生成图像再次嵌入水印；还可以使用 Bmshj2018、Cheng2020 等学习式图像压缩模型重构图像以尝试移除水印。论文没有把攻击者设定为能读取 EW-LoRA 参数或直接白盒优化 LoRA 去水印。

d) 攻击者知识：主实验更接近黑盒/输出级攻击。攻击者知道图像可能带水印，但无法访问模型内部参数、LoRA 权重、目标 watermark bits 或训练过程。论文的 fine-tuning attack 属于更强模型级攻击模拟，但仍不是完整白盒自适应攻击。

e) 攻击约束：攻击后图像仍需保持可用质量。后处理和压缩攻击用 PSNR 与 BitACC 曲线衡量同等失真下的水印可恢复性；所有权验证在固定 FPR 下报告 TPR。水印本身需要满足 accuracy、unobtrusiveness、robustness、payload 和 efficiency 五项要求。

f) 防御者能力与安全目标：防御者能预训练或使用一个 watermark decoder $D_w$，冻结 LDM 原参数，只训练 VAE decoder 选定层中的 LoRA 参数。安全目标是在生成图像中稳定嵌入 $n$ bit watermark，使其能以高 BitACC 提取，同时尽量保持 PSNR/SSIM/LPIPS/SIFID 接近 clean model，并以极少参数和较短训练时间完成嵌入。

g) 现实性评估：box-free 设定贴近托管模型服务或用户只能拿到生成图像的场景。它的强假设是攻击者不能访问模型权重；一旦攻击者能继续微调模型，论文 Fig. 10 显示 EW-LoRA 的 BitACC 会快速下降，因此它不是强白盒模型水印。

h) 与后文关系：VAE decoder LoRA 对应效率和不可感知性；DLWS 对应收敛效率和 loss trade-off；后处理/覆盖/移除攻击对应 robustness；binomial hypothesis testing 对应 ownership verification；cross-dataset/cross-backbone 实验对应泛化。

## 3. 方法设计与复现级理解

### 3.1 全局流程图式说明

EW-LoRA 的流程分为三个阶段。

第一阶段预训练 watermark decoder $D_w$。它从图像中提取 $n$ bit watermark，训练时使用可微 noise layer 增强对裁剪、缩放、JPEG 等常见后处理的鲁棒性。这个 decoder 训练完成后被冻结。

第二阶段给 LDM 嵌入水印。原 LDM 的 VAE encoder、denoising module 和 VAE decoder 原始参数全部冻结，只在 VAE decoder 的选定层插入 LoRA 参数 $A,B$。训练时，同一 latent 通过 clean VAE decoder 得到 clean image $\hat{x}$，通过带 LoRA 的 VAE decoder 得到 watermarked image $\tilde{x}$。冻结的 $D_w$ 从 $\tilde{x}$ 中提取 watermark $\tilde{w}$，损失函数同时约束 $\tilde{x}$ 接近 $\hat{x}$，并让 $\tilde{w}$ 接近目标 watermark $w$。

第三阶段验证所有权。部署后，用户用 prompt 生成图像，验证者只对输出图像运行 $D_w$，得到 watermark bits，并通过 binomial hypothesis test 判断目标 watermark 是否存在。整个验证不需要访问模型权重，因此是 box-free。

### 3.2 LoRA 基础：用低秩残差承载水印

LoRA 的基本形式是对原参数矩阵 $W\in\mathbb{R}^{m\times n}$ 添加低秩更新：

$$\tilde{W}=W+\Delta W,\quad \Delta W=\frac{\alpha}{r}BA$$

其中 $B\in\mathbb{R}^{m\times r}$，$A\in\mathbb{R}^{r\times n}$，$r\ll\min(m,n)$，$\alpha$ 控制更新强度。LoRA 的关键优势是训练参数量从 $mn$ 降到 $r(m+n)$，训练后还能把低秩更新 merge 回原矩阵，推理时没有额外结构开销。

论文还评估了 LoRA 变体。LoHA 用两个低秩项的 Hadamard product 增强表达力：

$$\tilde{W}=W+\Delta W,\quad \Delta W=\frac{\alpha}{r}(B_1A_1\odot B_2A_2)$$

PiSSA 用 $W$ 的 top-$r$ SVD 初始化训练低秩项：

$$W=U_r\Sigma_rV_r^\top+R,\quad \tilde{W}=R+\Delta W,\quad \Delta W=\frac{\alpha}{r}BA$$

VeRA 共享冻结低秩基，只训练 per-layer scaling vectors：

$$\tilde{W}=W+\Delta W,\quad \Delta W=\mathrm{Diag}(b)B\mathrm{Diag}(d)A$$

这些公式在论文中用于说明：低秩 adaptation 与参数型模型水印有共同数学基础，都是通过轻量残差 $\Delta W$ 向预训练模型注入额外信息。

### 3.3 EW-LoRA 的水印参数化

一般模型水印可以写成对原参数 $\theta$ 添加扰动：

$$\tilde{\theta}=\theta+\Delta\theta$$

EW-LoRA 将这个扰动限制成低秩形式：

$$\Delta\theta=\frac{\alpha}{r}BA,\quad B\in\mathbb{R}^{d\times r},\quad A\in\mathbb{R}^{r\times d},\quad r\ll d$$

与 AquaLoRA 的关键区别是插入位置。AquaLoRA 把 LoRA 放在 denoising module，这会影响扩散去噪轨迹，可能造成语义偏移；EW-LoRA 把 LoRA 放在 VAE decoder 的 selected layers，因为最终 ownership verification 从生成图像中读取水印，越接近图像输出越能直接控制 image-level watermark signal。

具体地，给定 denoised latent $\tilde{z}_0$，带水印 VAE decoder 输出为：

$$\tilde{x}=D_I(\tilde{z}_0;\tilde{\theta}),\quad \tilde{\theta}_c=\theta_c+\Delta\theta_c,\quad \tilde{\theta}_l=\theta_l\ \mathrm{if}\ l\ne c$$

其中 $c$ 表示被选中的 VAE decoder 层，其他层参数不变。被选层的更新为：

$$\Delta\theta_c=\frac{\alpha}{r}BA$$

训练时只有 $\Delta\theta_c$ 可训练，原始 VAE decoder 参数不更新。这使水印可以作为小型增量参数保存、分发或合并。

### 3.4 训练目标：图像保真与水印可提取

论文用 clean VAE decoder 输出 $\hat{x}$ 作为图像质量参考：

$$\hat{x}=D_I(\tilde{z}_0;\theta_c)$$

带 LoRA 的输出是 $\tilde{x}$，冻结 watermark decoder 提取：

$$\tilde{w}=D_w(\tilde{x})$$

训练总损失为：

$$L=\lambda_iL_i(\tilde{x},\hat{x})+\lambda_wL_w(\tilde{w},w)$$

其中 $L_i$ 是 image loss，论文使用 Watson-VGG image loss；$L_w$ 是 watermark loss，使用 BCE。$\lambda_i$ 和 $\lambda_w$ 控制 generation fidelity 与 watermark detectability 的 trade-off。

复现时要注意，EW-LoRA 不重新训练 watermark decoder，也不训练 denoising backbone。训练数据流是：输入图像 -> VAE encoder 得 latent -> clean decoder 生成 $\hat{x}$ -> LoRA decoder 生成 $\tilde{x}$ -> 冻结 $D_w$ 提取 bits -> 计算 $L_i,L_w$ -> 只更新 LoRA 参数。

### 3.5 DLWS：按当前瓶颈动态调节 loss 权重

固定 loss 权重的问题是两个目标优化速度不同。预训练 LDM 已经能生成高质量图像，所以 image loss 初始较接近良好状态；watermark loss 则从零开始。如果 $\lambda_i$ 太大，水印学得慢；如果 $\lambda_w$ 太大，图像质量下降。DLWS 用当前 batch 的 PSNR 和 BitACC 判断哪个目标没达标，并动态增加对应权重。

固定权重下第 $t$ 次迭代的优化方向为：

$$\nabla_\theta L_t=\lambda_i^t\nabla_\theta L_i+\lambda_w^t\nabla_\theta L_w$$

DLWS 维护 PSNR target $\rho$ 和 BitACC target $\mu$。当前 batch 指标为 $\rho_t,\mu_t$，目标违反情况为：

$$v_i^t=\mathbb{I}[\rho_t<\rho],\quad v_w^t=\mathbb{I}[\mu_t<\mu]$$

如果 $\mu_t<\mu$，说明水印准确率不足，增加 $\lambda_w$；如果 $\mu_t\ge\mu$，再检查 $\rho_t$，若图像质量不足则增加 $\lambda_i$。当某个目标连续 $p$ 次满足后，DLWS 提高对应 target：$\mu=\mu+s_\mu$ 或 $\rho=\rho+s_\rho$，推动训练逐步追求更高指标。

论文给出的初始设置是 $\rho=30$，$\mu=0.9$，$p=5$，$\gamma=0.25$，初始 loss weights 为 $\lambda_i=1$，$\lambda_w=0$。算法中 watermark loss 权重更新更激进：

$$\lambda_w\leftarrow\lambda_w+2\gamma,\quad \lambda_i\leftarrow\lambda_i+\gamma$$

这个非对称设计是为了补偿初始化不平衡：图像生成已经强，而水印目标从头学习，所以需要先更快提高 watermark gradient 的贡献。

### 3.6 BitACC 与所有权验证

给定目标 watermark $w\in\{0,1\}^n$ 和提取 watermark $\tilde{w}$，BitACC 定义为：

$$BitACC=\frac{1}{n}\sum_{i=1}^{n}\mathbb{I}(\tilde{w}_i=w_i)$$

所有权验证被写成 hypothesis testing。令 $K$ 为 decoded watermark 与 target watermark 匹配的 bit 数。无水印假设 $H_0$ 下，匹配 bit 随机：

$$K\sim Bin(n,0.5)$$

有水印假设 $H_1$ 下，若 bit error probability 为 $p$，则：

$$K\sim Bin(n,1-p)$$

当 $K\ge T$ 时声明 ownership，阈值 $T$ 由固定 FPR $\alpha$ 确定：

$$Pr(K\ge T\mid H_0)\le\alpha$$

如果使用 $m$ 张生成图像联合验证，则把 $n$ 替换为 $nm$。论文指出在 confidence 0.95、$\alpha=0.005$ 时，$m\ge767.32$，因此使用 1000 张测试图像足够。

### 3.7 训练与推理分离

训练阶段需要 clean VAE decoder 和 LoRA VAE decoder 同时前向，以计算 image loss；还需要冻结的 watermark decoder 计算 BCE。只更新 LoRA 参数。

部署阶段可以把 LoRA 参数作为附加水印模块保留，也可以 merge 到 VAE decoder 参数中。用户正常用 text-to-image pipeline 生成图像，水印自然出现在输出中。

验证阶段只需要生成图像和 $D_w$，不需要访问 LDM 参数、LoRA 参数或 prompt。这个分离是 box-free watermarking 的核心。

### 3.8 复现配置表

| 项目 | 论文设置 |
|---|---|
| 期刊 | Expert Systems With Applications, 2026 |
| 训练/测试数据 | COCO、CelebA、Flickr、ArtELingo；每个数据集随机 3000 train / 1000 test |
| base models | SDv1.4、SDv1.5、SDv2.1、SDv3.5-L、DiT |
| 默认 backbone | SDv1.4 |
| 默认 payload | 48-bit，使用 HiDDeN decoder |
| 高 payload | 100-bit / 150-bit，使用 StegaStamp decoder |
| 训练参数 | AdamW，initial lr $5\times10^{-4}$，batch size 4，15000 iterations |
| 硬件/软件 | NVIDIA L40S，Python 3.8.18，PyTorch 2.1.0，CUDA 12.2 |
| noise layer | crop/resize 参数 0.3 或 0.7 等概率；JPEG 概率 0.5；QF 50 或 80 等概率 |
| DLWS 初始 | $\rho=30$，$\mu=0.9$，$p=5$，初始 $\lambda_i=1,\lambda_w=0$ |
| DLWS 超参选择 | $\gamma=0.30$，$(s_\rho,s_\mu)=(2.5,0.08)$ |
| 默认 LoRA 插入层 | VAE decoder 的 up.0.block.conv |
| 默认 LoRA 超参 | $(r,\alpha)=(4,32)$ |
| 主要指标 | Params, BitACC, AT-99, PSNR, SSIM, LPIPS, SIFID, TPR@0.5%FPR |
| 代码 | 论文未提供 EW-LoRA 官方代码链接；Data available on request |

### 3.9 复现风险与信息缺口

第一，论文给出了默认插入层 up.0.block.conv，但不同 diffusers 版本中 VAE decoder 层命名和结构可能不同，需要严格对齐模型实现。

第二，watermark decoder 的训练细节依赖 HiDDeN/StegaStamp 和 noise layer。若 decoder 鲁棒性不足，EW-LoRA 的后处理鲁棒性会被上游 decoder 限制。

第三，DLWS 的计数器在 Algorithm 1 中初始化位置需要工程实现时谨慎处理。实际训练中应让 $c_i,c_w$ 跨 iteration 保持状态，而不是每次函数调用都重置。

第四，EW-LoRA 对 continued fine-tuning 攻击较脆弱。若攻击者能访问模型并继续优化无水印任务，低秩水印 carrier 可能很快被削弱。

## 4. 与其他方法对比

| 方法类别 | 代表方法 | 嵌入位置 | 参数/效率 | 优点 | 缺点 | EW-LoRA 的差异 |
|---|---|---|---|---|---|---|
| FPW | Stable Signature | VAE decoder 全参数微调 | 49.49M trainable params | 成熟、BitACC 高 | 参数量大，训练成本高 | 只训 0.0353M LoRA 参数 |
| APW adapter | WMAdapter, LaWa | VAE decoder 中插入 adapter 或 latent perturbation | 1.1978M / 37.8943M | 输出级水印较直接 | 模块较重或 jointly train 复杂 | 用低秩参数替代大 adapter |
| denoising-side LoRA | AquaLoRA | denoising module | 517.5586M | 不覆盖原参数 | 参数量仍大，可能语义偏移 | 插入 VAE decoder 选层，靠近图像输出 |
| EW-LoRA | 本文 | VAE decoder up.0.block.conv | 0.0353M | 参数极少、收敛快、图像质量高 | 微调攻击下更脆弱 | 低秩水印 carrier + DLWS |

与 AquaLoRA 的关键差异是插入位置和 rank budget。AquaLoRA 在 denoising network 侧承载水印，影响生成语义路径；EW-LoRA 在 VAE decoder 选层承载水印，目标是更直接控制输出图像水印，同时显著减少可训练参数。

与 Stable Signature 的关键差异是是否 full fine-tuning。Stable Signature 全量微调 VAE decoder，参数多但水印稳定；EW-LoRA 保持原参数冻结，只学习低秩残差，效率更高但对后续模型级微调更敏感。

## 5. 实验表现与优势

a) 主实验比较 embedding efficiency、BitACC、视觉质量和 ownership verification。Table 2 中 EW-LoRA(48) 在 SDv1.4 上只需 0.0353M 参数，BitACC 0.997，AT-99 为 2.931 min，PSNR 32.543，SSIM 0.870，LPIPS 0.025，SIFID 0.093，TPR@0.5%FPR 为 1.000。相比之下，AquaLoRA 需要 517.5586M 参数且只能达到 AT-95；Stable Signature 需要 49.49M 参数；LaWa 需要 37.8943M 参数且 AT-99 为 3898 min。

b) LoRA 变体实验显示该框架可兼容 LoHA、PiSSA、VeRA。EW-VeRA 参数最少，仅 0.0015M；EW-PiSSA 图像质量较高，PSNR 33.370、LPIPS 0.015、SIFID 0.069。所有变体的 TPR@0.5%FPR 均为 1.000，说明低秩形式变化不破坏基本 ownership verification。

c) payload 实验显示 EW-LoRA 可扩展到更长 watermark。100-bit 设置 BitACC 0.994，PSNR 28.789；150-bit 设置 BitACC 0.992，PSNR 27.058。图像质量随 payload 增加下降，但 BitACC 和 TPR 仍高。论文也明确说明这里同时改变了 decoder 架构：48-bit 用 HiDDeN，100/150-bit 用 StegaStamp，因此不能把差异完全归因于 payload 长度。

d) 后处理鲁棒性实验 Table 3 覆盖 center crop 0.1/0.5、rotation 25°、resize 0.3/0.7、brightness/contrast/sharpness 1.5、JPEG QF80/QF50。EW-LoRA 在 P0–P10 中表现为 0.997、0.922、0.993、0.665、0.706、0.983、0.989、0.992、0.997、0.849、0.781，在多数攻击中处于第一或第二梯队。相对较弱的是旋转和强 resize，但仍优于或接近其他方法。

e) watermark overwriting 实验 Table 4 显示，使用 StegaStamp 覆盖时，EW-LoRA 的 overwriting/original BitACC 为 0.939/0.997；使用 SSL-Watermarking 覆盖时为 0.997/0.997。说明新水印难以擦除原模型身份，EW-LoRA 接近 WMAdapter 的表现。

f) watermark removal 实验使用 Bmshj2018 和 Cheng2020 learning-based compression。Fig. 4 显示在相同 PSNR 失真水平下，EW-LoRA 通常保持更高 BitACC，除 Bmshj2018 下 AquaLoRA 的部分情况外，整体优于 Stable Signature、WMAdapter、LaWa。

g) 跨数据集实验 Table 5 中，COCO、CelebA、Flickr、ArtELingo 交叉训练/测试的 BitACC 基本都超过 0.95。CelebA 测试图像通常 PSNR 更高，比如 COCO->CelebA 为 38.145 dB，ArtELingo->CelebA 为 37.390 dB。说明水印特征对数据分布变化不敏感。

h) 跨模型实验 Table 6 中，EW-LoRA 在 SDv1.4、SDv2.1、SDv3.5-L、DiT-512 上 BitACC 分别为 0.997、0.995、0.996、0.997，PSNR 在 32.011–33.173 范围。由于水印嵌入在 VAE decoder，这些结果应解释为对具有兼容 VAE decoding 的 LDM backbone 泛化，而不是对任意生成架构泛化。

i) ablation 表明插入层和超参数选择很关键。Fig. 6/7 显示 VAE decoder 的 up.0.block.conv 是最有效插入位置；attention 层 q/k/v/out 收敛不稳定或水印保留差。Table 7 中默认选择 $(r,\alpha)=(4,32)$，在保持 0.0353M 参数的同时获得较好 PSNR 和 AT-99。

j) 局限实验 Fig. 10 显示 fine-tuning attack 会让 BitACC 随攻击步数下降，EW-LoRA 在继续无水印任务优化 500 steps 后会快速下降。这是论文最重要的安全边界：低秩参数高效，但在模型级白盒/灰盒继续训练面前不如强嵌入稳定。

## 6. 学习与应用

a) 论文没有公开 EW-LoRA 官方代码。若要复现，建议从 diffusers 的 SDv1.4 VAE decoder 开始，在 up.0.block.conv 加 LoRA，冻结其余参数；先用 HiDDeN 训练 48-bit watermark decoder，再用 image-to-image reconstruction protocol 训练 EW-LoRA。

b) 关键实现点包括：保持 clean decoder 和 LoRA decoder 的输入 latent 一致；只更新 LoRA 参数；使用 Watson-VGG loss 约束 $\tilde{x}$ 与 $\hat{x}$；watermark BCE 通过冻结 $D_w$ 反传到 LoRA；DLWS 的 $\lambda_i,\lambda_w,\rho,\mu,c_i,c_w$ 需要跨 iteration 维护。

c) 迁移上，EW-LoRA 对“有 VAE decoder 的 LDM”最直接，包括 SD 系列和部分 DiT pipeline。对没有兼容 VAE decoder 的生成模型，必须重新寻找靠近输出端的低秩插入点。对音频生成水印，可借鉴“只在 decoder 末端低秩插入 + 动态 loss 调权”的思想，但不能直接使用图像 watermark decoder 或 VAE 层选择结论。

## 7. 总结

a) 一句话概括：用极少 LoRA 参数写模型水印。

EW-LoRA 的核心贡献是证明 LDM 模型水印不一定需要全量微调或大型 adapter。只在 VAE decoder 的关键层插入低秩更新，并用 DLWS 动态平衡图像质量和水印准确率，就能获得高效、box-free、可从生成图像验证的水印。

## 8. 图表精读与证据链

Fig. 1 对比 FPW 和 APW 流水线，说明 EW-LoRA 位于“additional parameter watermarking”但比 AquaLoRA 更靠近输出端、更轻量。

Fig. 2 是方法总览：先预训练并冻结 watermark decoder，再在 VAE decoder 中插入 LoRA，由 $D_w$ 监督水印提取，最后用 DLWS 调整 loss weights。它对应论文的两项核心设计。

Table 1 列出现有方法的固定 loss 权重，支撑 DLWS 的动机：不同方法依赖人工设置 $\lambda$，而固定权重难以适应训练阶段变化。

Table 2 是主证据链。它同时展示参数量、BitACC、AT-99、图像质量和 TPR，证明 EW-LoRA 的 Pareto 优势主要在极少参数和较高质量，而不只是单个指标最优。

Table 3 是后处理鲁棒性证据。EW-LoRA 在常见失真下多数位于前两名，但旋转和强 resize 仍是相对困难项。

Table 4 和 Fig. 4 分别验证覆盖攻击和移除攻击。覆盖攻击下原水印保持较好；学习式压缩移除下 EW-LoRA 在多数 PSNR 区间保持较高 BitACC。

Table 5/6 支撑泛化 claim。Table 5 是跨数据集，Table 6 是跨 LDM backbone。需要注意 Table 6 的结论依赖“compatible VAE decoding”，不能扩展成任意模型结构都适用。

Fig. 6/7 和 Table 7 是复现关键证据。它们说明为什么选 up.0.block.conv，为什么默认用 $(r,\alpha)=(4,32)$，也解释了 attention layers 为什么不适合作为默认水印载体。

Fig. 10 是限制证据。它直接显示 EW-LoRA 在 fine-tuning attack 下下降快，是安全边界而不是次要细节。

## 9. 复现难度与适合人群

复现难度：中高。LoRA 插入本身不复杂，但要复现论文结果，需要训练/准备 watermark decoder、对齐 Stable Signature 协议、实现 DLWS、选择 VAE decoder 插入层、实现 post-processing noise layer 和所有权假设检验。

主要依赖：diffusers/Stable Diffusion、HiDDeN 或 StegaStamp watermark decoder、COCO/CelebA/Flickr/ArtELingo 数据、NVIDIA L40S 或类似 GPU、Python 3.8/PyTorch 2.1/CUDA 12.2、图像质量指标和 BitACC/TPR 评估脚本。

最小可复现版本：使用 SDv1.4 和 48-bit HiDDeN decoder，只在 up.0.block.conv 插入 LoRA，采用 image-to-image reconstruction protocol，先不跑 text-to-image。目标是复现 0.99 左右 BitACC、30 dB 以上 PSNR 和少量参数训练。之后再加入 DLWS、后处理鲁棒性和跨数据集评估。

适合人群：适合研究生成式图像模型水印、LDM ownership verification、高效微调、LoRA 安全应用的人。对语音安全研究者，它的参考价值主要在“低秩增量水印”和“动态 loss 调权”，而不是图像后处理鲁棒性本身。

## 10. 简短全面总结

EW-LoRA 面向 Latent Diffusion Models 的 box-free 模型水印。论文指出，Stable Signature 等 full-parameter watermarking 训练成本高，AquaLoRA 等 additional-parameter 方法虽然较轻，但 denoising-side LoRA 参数量仍大且可能造成语义偏移。EW-LoRA 将低秩水印载体放到 VAE decoder 的选定层，只训练 LoRA 参数，原模型完全冻结；同时使用冻结的 watermark decoder 从生成图像中提取 bits，以 Watson-VGG image loss 和 BCE watermark loss 联合训练。DLWS 根据当前 PSNR 与 BitACC 动态调整 $\lambda_i,\lambda_w$，加速收敛并平衡图像质量与水印准确率。实验显示 EW-LoRA 用 0.0353M 参数达到 0.997 BitACC 和 1.000 TPR@0.5%FPR，并在多 payload、后处理、覆盖、压缩移除、跨数据集和跨 VAE-compatible LDM 上保持有效。主要限制是对模型级继续微调攻击较脆弱，且官方代码未公开。

## 11. 论文写作逻辑分析

a) Intro 的铺垫从生成模型商业化和模型知识产权切入，再把已有 LDM 水印分成 FPW/APW 两类，问题定位清楚：不是能不能水印，而是大模型时代如何高效、低损地水印。

b) insight 与方法连接紧密。作者先指出 AquaLoRA denoising-side 参数量大且可能语义偏移，再自然引出 VAE decoder 选层 LoRA：靠近输出端、参数少、图像级信号直接。

c) threat model 明确服务于 box-free 验证。因为验证者只能看输出图像，方法必须让 watermark 可从生成结果中提取；因为模型参数不可访问，低秩参数是否容易保存/merge 也成为部署优势。

d) 方法叙事按“低秩参数化 -> 训练目标 -> 动态 loss 权重 -> 验证假设检验”展开，顺序合理。DLWS 不是孤立技巧，而是从固定 loss 权重收敛慢的问题推出来的。

e) 实验呼应较完整：Table 2 验证效率和质量，Table 3/4/Fig. 4 验证鲁棒性，Table 5/6 验证泛化，Fig. 6/7/Table 7 验证插入层与超参数，Fig. 10 写出失败边界。

f) 证据最强的是效率对比，因为参数量差距非常大；较弱的是安全强度，因为白盒模型级攻击下 EW-LoRA 明显脆弱，且攻击感知训练目标仍是未来工作。

g) 可借鉴写法是：用系统分类框架先压缩相关工作，再从分类中抽出具体效率瓶颈，最后用一张主表同时报告性能、效率、质量和验证指标。这种写法适合强调 engineering trade-off 的模型水印论文。

## Overview.md 条目

- [An Efficient Watermarking Method for Latent Diffusion Models via Low-Rank Adaptation and Dynamic Loss Weighting](./Other_Security/2026-EW-LoRA.md)  
  *Expert Systems With Applications, 2026*  
  Citation: Lin, D., Li, Y., Tondi, B., Lin, K., Li, B., & Barni, M. “An Efficient Watermarking Method for Latent Diffusion Models via Low-Rank Adaptation and Dynamic Loss Weighting.” *Expert Systems With Applications*, vol. 331, Article 133172, 2026.  
  Links: [Paper](https://doi.org/10.1016/j.eswa.2026.133172) | [DOI](https://doi.org/10.1016/j.eswa.2026.133172)
