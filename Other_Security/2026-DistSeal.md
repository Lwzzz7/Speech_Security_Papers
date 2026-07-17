# Learning to Watermark in the Latent Space of Generative Models (DISTSEAL)

## 0. 摘要翻译

现有 AI 生成图像水印多在像素空间进行后处理，因而会带来计算开销和视觉伪影。本文探索潜空间水印，提出统一方法 DISTSEAL，适用于扩散模型与自回归模型。其做法是在生成模型的潜空间训练后置水印器，随后可将该潜空间水印器蒸馏到生成模型本身或 latent decoder 中，从而实现模型内水印。结果表明，潜空间水印在保持相近不可感知性与竞争性鲁棒性的同时，相比像素空间基线可获得最高 20 倍加速。实验进一步发现：蒸馏潜空间水印优于蒸馏像素空间水印，因而为开源生成模型提供了更高效、更稳健的模型内水印方案。代码与模型已公开。

## 1. 方法动机

像素后处理水印的优点是可插拔、模型无关，却需要在高分辨率图像上额外运行 embedder；例如 512×512×3 的像素表示远大于压缩 latent。更严重的是，对开源模型而言，使用者只要跳过后处理调用即可绕过水印。已有生成期水印有的改初始噪声、有的只改 decoder，通常面临模型类型受限、无法关闭后处理与可蒸馏性不足之间的权衡。

DISTSEAL 的核心假设是：生成模型最终都要经由连续 latent 或离散 token latent 解码成图像。若将后置 watermarker 从像素空间迁移到这一较小的表示空间，再将它产生的水印 latent 蒸馏入生成器或 decoder，便能同时获得低延迟后置水印与更难被一行代码关闭的模型内水印。论文特别关注两个问题：潜空间信号是否足够鲁棒，以及这种信号是否比像素扰动更容易被模型权重学习。

## 2. 威胁模型解读

| 项目 | 论文设定与边界 |
|---|---|
| 保护者 | 生成模型/模型服务提供者；训练 latent watermarker，并可将其蒸馏进模型或 decoder。 |
| 验证者 | 持有 extractor，从生成图像（可经变换）恢复固定或动态消息。 |
| 攻击者 | 对图像实施 valuemetric、几何、压缩及组合编辑；对开源模型可跳过外置后处理。 |
| 防御目标 | 后置模式下快速、不可感知且可提取；模型内模式下不依赖每次推理后的外置调用。 |
| 主要能力限制 | 文中实验不是对拥有全部权重后进行强重训练、蒸馏删除或 extractor/key 泄露的完备安全证明。 |
| 关键区分 | “post-hoc latent”保护的是输出处理管线；“in-model”才针对开源发布时直接关闭后处理的问题。 |

论文的鲁棒攻击是图像层编辑，而蒸馏实验是模型层集成能力测试。两者不能互相替代：高 bit accuracy 不证明权重级不可删除；成功蒸馏也不证明可抵抗任意微调遗忘。

## 3. 方法设计与复现级理解

### 3.1 背景与统一表示

对图像 x，autoencoder 给出 latent z，再由 decoder D 重建图像。扩散模型在 latent 中从噪声逐步去噪；自回归模型则把连续 latent 量化为离散 token 序列。DISTSEAL 将两种模型统一为“在 decoder 前修改 latent”：连续 latent 可直接扰动，离散 latent 需要考虑量化前或 token embedding 两种位置。

### 3.2 阶段一：训练后置 latent watermarker

**目标。** 这一阶段的目标不是修改生成器，而是学习 embedder W 和 extractor E：W 读入生成模型产生的 latent 及 K-bit 消息 m，输出可解码但视觉不可感知的水印 latent；E 从经攻击的图像恢复 m。该阶段产物是可动态改变消息的 post-hoc watermarker，也将作为下一阶段蒸馏教师。

**连续 latent 分支。** 对连续 z，watermarked latent 和图像为：

$$
z_w=z+\epsilon W_\theta(z,m),\qquad x_w=D(z_w).
$$

其中 epsilon 是水印强度，控制提取鲁棒性与视觉扰动的权衡。与像素水印直接修改 x 不同，W 的空间尺寸与 latent 一致；论文为此移除了 VideoSeal embedder 的下采样/上采样层，仅保留中间 ResNet blocks，而 extractor 架构保持一致，避免把架构差异误当成 latent 优势。

**离散 latent 分支。** 对自回归模型，可在量化前修改连续表示再量化：

$$
z_w=Q\bigl(z+\epsilon W_\theta(z,m)\bigr),
$$

训练时通过 straight-through estimator 穿过 Q。也可在量化后修改 token embedding；后者鲁棒性较好，却不会产生新的离散 token 序列，故不能蒸馏进自回归生成器，只能蒸馏到 decoder。这是论文中两个“latent watermarker”变体的部署差异。

**攻击、提取与损失。** 水印图先接受 valuemetric、几何与压缩增强，再送至 E。提取项是消息预测与真值的 BCE，discriminator 项在像素空间使用 hinge loss，迫使 x_w 与真实图不可区分：

$$
L=\lambda_w L_w(x_w,m)+\lambda_{disc}L_{disc}(x,x_w).
$$

作者刻意不对 latent watermarker 加 reconstruction/perceptual loss：量化 latent 的微扰可能触发语义变化，强行逐像素对齐不一定合适；他们以 epsilon 与对抗 discriminator 约束水印可见性。这一选择是方法特征，也意味着视觉质量必须由 FID/IS 等独立实验检验。

### 3.3 阶段二：将教师 watermarker 蒸馏为模型内水印

第一阶段完成后，固定 W 和一条固定消息 m。对每个训练样本先由 W 产生 z_w，之后不再要求推理时运行 W，而是让生成模型或 decoder 学会自行输出对应水印结果。

**蒸馏进生成模型。** 对扩散或自回归生成器 G，训练目标由原来的干净 latent/token z 改为 z_w：

$$
L_{gen}=L_{recon}\bigl(G_\phi(x),z_w\bigr).
$$

扩散场景中的 x 是带噪 latent，自回归场景中的 x 是 token 序列；L_recon 分别可为 MSE 或 cross-entropy。完成后 G 在常规生成时直接产生带水印 latent，省去后置步骤。量化后 embedding 水印不能走此分支，因为它不定义新的 token 目标。

**蒸馏进 latent decoder。** 也可以保留生成器的干净 z，仅微调 D，使 D(z) 逼近原 decoder D_o 对水印 latent 的输出：

$$
L_{dist}=L_{rec}\bigl(D_o(z_w),D(z)\bigr)+\lambda_wL_w\bigl(E_\theta(D(z)),m\bigr),
$$

$$
L_{rec}=\ell_1+\lambda_pL_{LPIPS}.
$$

第一项把“后置教师生成的水印图”作为视觉目标，第二项直接确保 extractor 能读出消息。论文强调这不同于 Stable Signature：后者主要由 extractor 信号引导，而 DISTSEAL 显式要求 distilled decoder 重建教师的水印输出。该设计解释了为什么 latent 教师比 pixel 教师更可蒸馏——latent 扰动更符合 decoder 的可表示空间，高频像素噪声则难由 decoder 复现。

### 3.4 训练、部署与复现配置

| 项目 | 论文配置 |
|---|---|
| 扩散模型 | ImageNet class-conditional UViT-H + DCAE，latent 为 8×8×128，输出 512×512 |
| 自回归模型 | RAR-XL + MaskGIT-VQGAN，256×256 压缩为 256 token，codebook 1024 |
| 水印教师训练 | 64 bit，ImageNet，600k steps，batch 128；前 100k 高强度 epsilon，后 100k cosine 降至最终强度 |
| 生成器蒸馏 | DCAE 100k steps；RAR-XL 10k steps，均恢复原论文训练并改用水印目标 |
| decoder 蒸馏 | Adam，10k steps，batch 16，1k step warmup 至 1e-4 后 cosine decay |
| decoder LPIPS | DCAE 取 lambda_p=0；RAR-XL 取 lambda_p=1 |
| 评估规模 | 后置与模型内各生成 50,000 图；FID/IS 对 ImageNet val 计算 |

部署时，后置模式可为每次生成传入不同 m；模型内模式固定 m，推理只跑蒸馏后的 G 或 D。论文未给出将不同固定消息安全轮换到同一权重的完整运维方案；附录只进一步讨论 multi-watermarking 和 watermark forgetting。

## 4. 与其他方法对比

| 方法类别 | 水印位置 | 动态消息 | 开源模型能否跳过 | 关键限制 | DISTSEAL 的改进 |
|---|---|---:|---:|---|---|
| 像素后置（VideoSeal） | 高分辨率图像 | 是 | 是 | 延迟高，蒸馏到 decoder 困难 | latent 处理降低维度并更易蒸馏 |
| 初始噪声/采样引导 | 生成过程外部 | 通常受限 | 取决于实现 | 模型类型与 sampler 依赖 | 可蒸馏到权重，统一扩散/AR |
| Stable Signature | latent decoder | 固定为主 | 不易跳过 | 依赖 decoder 集成 | 以教师水印图重建加 extractor 联合蒸馏 |
| AquaLoRA | U-Net LoRA | 多用户设计 | 模型内 | 以扩散为主 | DISTSEAL 同时支持 AR，且以输入图条件的 latent watermark 为教师 |

## 5. 实验表现与优势

### 5.1 后置连续 latent：效率与常规鲁棒性

动机是检验 latent 操作会否牺牲质量和编辑鲁棒性。DCAE 上，pixel 后置与 latent 后置使用同一训练目标、日程、增强与超参；表 1 报攻击后 bit accuracy、PSNR、FID、IS。latent 平均 bit accuracy 为 95.18%，低于 pixel 的 97.78%，组合攻击下为 84.28% 对 97.29%；但 valuemetric 98.08%、compression 99.23% 仍强。PSNR 从 43.48 降到 31.06 dB，作者指出 latent 小差异经 decoder 放大，PSNR 不足以代表感知质量；相对无水印 FID 10.66，pixel/latent 仅增加 +0.18/+0.76，IS 仍为 99.53/98.11。

效率是该阶段最明确的收益：DCAE latent 8×8×128 相比像素 512×512×3，CPU 单图从 63 ms 降为 3 ms，约 20×。去掉输入依赖的 coverless latent 变体平均仅 91.81%，说明并非“在 latent 里放固定模式”就足够，内容条件化的 W 有贡献。

### 5.2 自回归离散 latent：量化位置的影响

RAR-XL 上，量化前水印平均为 90.60%，量化后为 93.96%，pixel 为 97.27%。量化会抹去细小连续扰动，因此量化后直接改 embedding 更稳健；但量化前才生成新 token 序列，能作为 G 的蒸馏目标。二者 FID 接近（3.56/3.44 对 pixel 3.09）。latent 16×16×256 相比 256×256×3 的压缩更弱，速度优势降至 2.4×（17 ms 对 41 ms），与表示尺寸解释一致。

### 5.3 decoder 蒸馏：为何 latent 教师更好学

表 3（DCAE）直接验证论文的中心主张。仅用重建项时，latent 教师蒸馏后平均 bit accuracy 94.41%，接近教师 95.18%，FID 11.34 对 11.42；加入 extractor 权重 0.1 后提升至 96.77%，FID 11.48。相反 pixel 教师仅重建时为 51.38%，即使加较强 extractor 权重到 1.0 也仅 92.95%，且 FID 恶化至 17.08。图 3 的差分图显示 decoder 可复现 latent 教师的结构化模式，却难复现 pixel 教师的高频扰动。

RAR decoder 的表 4 同样显示 extractor loss 有必要：量化后 latent 教师不加该项为 81.40%，加入后为 90.57%，FID 3.62 对教师 3.44。它证明联合损失不仅是形式添加，而是弥补单纯图像重建不能保证可解码水印的缺口。

### 5.4 证据边界

攻击覆盖了图像编辑和组合变换，模型内评估覆盖 decoder/生成器蒸馏；但表格没有给出对已知 watermark 算法的自适应擦除、长期微调遗忘或模型提取攻击的完整主结果。这些在论文附录被讨论，不能由常规 bit accuracy 外推为不可删除性。

## 6. 学习与应用

代码与模型：<https://github.com/facebookresearch/distseal>。最小复现建议先只做 DCAE 连续 latent 的阶段一：保持 VideoSeal extractor、缩小 embedder 分辨率、固定 64-bit 和同一攻击管线，复现表 1 的速度/bit accuracy。随后以教师生成的 D_o(z_w) 训练 decoder，并分别设置 extractor 权重为 0 与 0.1，验证表 3 所显示的蒸馏差异。实现时最易混淆的是 AR 的量化前/后两分支：前者适合蒸馏 G，后者适合 decoder；不可把两者的鲁棒指标直接当作相同部署能力。

## 7. 总结

一句话：**先在 latent 学后置水印，再把更易学习的 latent 信号蒸馏进生成模型权重。**

## 8. 图表精读与证据链

- **图 1**：两阶段总图。上半部是可动态消息的 latent teacher，下半部是固定消息的模型蒸馏；它解释为何后置与模型内不是相互排斥的两套方法。
- **图 2**：decoder 蒸馏的监督来自原 decoder 对 z_w 的输出，而非仅依赖 extractor；对应式 (3)。
- **表 1/2**：分别连续/离散 latent 的后置权衡。它们支撑效率和鲁棒主张，也暴露组合攻击与量化的困难。
- **图 3 与表 3**：最强证据链。相同 decoder 蒸馏框架下，latent 教师显著优于 pixel 教师，支持“latent 扰动更容易被 decoder 学习”的解释。
- **表 4**：将该结论扩展至 RAR，但绝对鲁棒性仍低于教师，显示 AR 离散表示的蒸馏损失。

## 9. 复现难度与适合人群

难度为高。它需要可训练的 ImageNet 级扩散/自回归生成器、50k 图评估、攻击增强、watermarker 对抗训练与多阶段蒸馏。主要依赖包括 DCAE/UViT-H、RAR-XL、autoencoder/tokenizer、GPU 计算和公开代码。最小可复现版本是只训练/评估 DCAE 的 post-hoc latent watermarker；完整 in-model 复现需恢复预训练生成器训练。适合研究生成内容溯源、扩散/AR watermark、开源模型版权保护的人群。

## 10. 简短全面总结

DISTSEAL 将生成图像水印从像素空间迁移到生成模型的 latent 空间，并把后置模式与模型内模式统一起来。第一阶段训练 content-dependent latent embedder 和 extractor，以 BCE 与像素空间 discriminator 在攻击增强下学习 64-bit 水印；连续 latent 直接扰动，离散模型则区分量化前 token 序列水印和量化后 embedding 水印。第二阶段把固定消息的 watermarked latent 作为教师目标，蒸馏进生成器，或让 decoder 重建原 decoder 对水印 latent 的输出并联合 extractor loss。DCAE 实验显示 latent 后置水印平均准确率 95.18%、CPU 推理约快 20 倍；decoder 蒸馏时 latent 教师可达 96.77%，明显优于像素教师蒸馏。论文的关键贡献是证明 latent 信号不仅更快，也更适合写入模型；边界是其鲁棒评估仍主要针对图像变换，而非完备的权重级自适应删除者。

## 11. 论文写作逻辑分析

论文先用工业常用的 pixel post-hoc 作为现实基线，指出效率与开源可关闭性两个缺口；随后将问题拆为“能否在 latent 后置水印”和“能否把它蒸馏成模型内水印”。方法模块顺序与该问题完全对应：4.1 先构造兼容连续/离散 latent 的教师，4.2 再给出 G 与 D 两条蒸馏路径。实验也沿着同一证据链：表 1/2 验证后置品质、鲁棒和速度，表 3/4 验证 latent 教师的可蒸馏性，图 3 给出机制层面解释。最值得学习的写法是把“教师水印容易蒸馏”这个抽象主张落实为 pixel/latent 教师在同一 decoder 训练条件下的受控比较；最需补强的是将开源对手的长期自适应移除实验纳入主文。
