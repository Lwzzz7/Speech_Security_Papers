# AquaLoRA: Toward White-box Protection for Customized Stable Diffusion Models via Watermark LoRA

## 0. 摘要翻译

本文研究定制 Stable Diffusion 模型的白盒版权保护。现有水印若放在初始噪声、生成图像或 VAE decoder，拥有模型的攻击者可替换 decoder 或采样策略绕过。AquaLoRA 先训练一个潜空间水印编码器/解码器，使秘密比特变成适于扩散 U-Net 学习的 latent 扰动；随后用先验保持微调将该模式写进 U-Net 的 Watermark LoRA。LoRA 的缩放矩阵携带不同用户的消息，因此可合并进定制模型权重。实验评估图像保真、失真、采样配置和白盒环境操作下的提取能力。

## 1. 方法动机

开放权重使定制 SD checkpoint 可被复制、重新分发。后处理水印只能保护图片副本；Tree-Ring 等采样期方案可被改 sampler 或去噪流程影响；Stable Signature 把标记留在 VAE decoder，白盒攻击者替换 VAE 即可移除。作者的主张是：保护信号必须与包含主要生成知识的 U-Net 耦合，才能让“移除水印”与“保持原模型质量”形成冲突。

## 2. 威胁模型解读

| 角色/能力 | 论文设定 |
|---|---|
| 所有者 | 持有秘密 encoder/decoder、为不同用户生成不同 Watermark LoRA，并将其 merge 到定制 U-Net |
| 攻击者 | 获得水印 SD 权重；可改 sampler、步数、CFG、分辨率、VAE，追加 LoRA/ControlNet，做图像失真或微调 |
| 验证 | 从水印模型生成图像，用私有 decoder 恢复 48-bit 秘密，以 bit accuracy 和固定 FPR 的 TPR 判定 |
| 安全边界 | 论文测试的是配置迁移、失真和有限微调；未给出面对从头蒸馏、重训练整个 U-Net、泄露秘密 decoder 的密码学不可移除保证 |

这比普通图像水印更强，因为攻击者白盒拥有生成器；但验证仍需要所有者私有密钥/解码器与足够多的生成样本。

## 3. 方法设计与复现级理解

### 3.1 两阶段总览：先得到 U-Net 可学习的 latent codebook，再将其写入模型权重

AquaLoRA 将问题拆开，而不是直接对定制 U-Net 训练一个“能出水印图”的 LoRA。第一阶段训练秘密 encoder/decoder，学习把用户 bit 串变为适合 VAE latent 空间的 watermark pattern；第二阶段冻结该 pattern 的产生/解码机制和原 U-Net，以 prior-preserving fine-tuning（PPFT）训练 Watermark LoRA。前一阶段交付私有 codebook \((E_s,D_s)\)，后一阶段交付可 merge 的 LoRA 权重；部署时水印存在于 U-Net，而非依赖外接 VAE decoder。

完整数据流为：消息 \(m\) → secret encoder \(E_s\) → watermark latent \(\Delta z_w\)；第一阶段用 VAE 验证其可见、可解码；第二阶段让加有 \(\Delta z_w\) 的扩散输入经 Watermark LoRA U-Net 去噪，得到图像后仍由同一 \(D_s\) 取回 \(m\)。因此 \(D_s\) 是私有验证器，LoRA 是面向模型分发的保护载体。

### 3.2 第一阶段：latent watermark pre-training（训练秘密 codebook）

**目标与输入。** 给定干净图像 \(I_o\)，VAE encoder 得到 \(z_o\)；随机 48-bit 消息进入秘密 encoder \(E_s\)，产生与 \(z_o\) 同尺度的水印扰动。将二者在 latent 空间结合后由 VAE decoder 得到水印图 \(I_w\)，再经随机失真层得到 \(I_r\)。秘密 decoder \(D_s\) 从 \(I_r\) 预测消息。此阶段不更新扩散 U-Net，避免还未确定水印语义时就破坏生成器。

**为什么需要 latent 而不是普通图像水印。** 传统像素水印进入 VAE encoder 后可能被压缩掉；相反，任意 latent 噪声虽然能进入 U-Net，却未必可在最终图像端稳定恢复。\(E_s\) 让消息主动适配 VAE/latent 通道，\(D_s\) 把“可恢复”定义为第二阶段固定要满足的接口。

**损失与训练逻辑。** bit 级 BCE 是主要恢复信号；LPIPS 约束水印图和原图的感知距离；SWD 约束潜在/图像统计分布；PRVL（Peak Regional Variation Loss）抑制局部突出的扰动峰；失真层让 decoder 对训练期攻击可用。它们的顺序关系是：先保证 decoder 有可学的消息，再用感知/分布项压低可见性，而不是将随机噪声直接当水印。论文使用 COCO2017 10,000 张训练图、AdamW（\(10^{-3}\)）、40 epoch、\(\lambda=5,\mu=0.5\)；失真层和损失调度的完整实现应以代码/附录为准。

### 3.3 第二阶段：Watermark LoRA 与 PPFT（将 codebook 集成进 U-Net）

**冻结关系。** 原 U-Net \(\epsilon_\vartheta\) 是教师且冻结；第一阶段 \(E_s,D_s\) 也冻结；仅优化 LoRA 的低秩矩阵。这样可将更新以 \(\Delta W=ASB\) 合并回定制 checkpoint，同时不必为每个用户全量训练 U-Net。

**消息化的 LoRA。** 普通 LoRA 为 \(AB\)。AquaLoRA 用各 bit 的嵌入组成缩放矩阵 \(S\)，使 \(ASB\) 随消息而变；不同用户 ID 对应不同的 \(S\)，而基础低秩因子可复用。这个设计解决的是多用户分发的存储/训练效率，不表示不同 ID 可在同一张图中同时被无歧义识别。

**PPFT 的前向与损失。** 对 \(z_0\)、caption \(c\)、噪声 \(\epsilon\) 和 timestep \(t\)，水印侧在含 \(\Delta z_w\) 的输入上预测噪声，教师在干净输入上预测噪声。最小化二者 MSE：
\[
\mathcal L_{PPFT}=\mathbb E\|\epsilon_\theta(\sqrt{\bar\alpha_t}(z_0+\Delta z_w)+\sqrt{1-\bar\alpha_t}\epsilon,t,c)-\epsilon_\vartheta(\sqrt{\bar\alpha_t}z_0+\sqrt{1-\bar\alpha_t}\epsilon,t,c)\|_2^2.
\]
实际含义是：水印 U-Net 面对被偏移的 latent 时，仍应执行与原 U-Net 面对干净内容近似的去噪，因此避免直接扩散微调导致的先验遗忘；同时由冻结 \(D_s\) 的比特恢复约束确保它没有把水印简单“抹平”。

### 3.4 粗类型适配与训练/部署边界

一个通用 LoRA 可能与 anime、人物、风格化等定制 checkpoint 的分布不匹配。作者在 PPFT 框架下按粗类型继续训练 AquaLoRA；实际部署为待保护模型挑选最接近的类型版本，而不是对每个 checkpoint 从零训练。PPFT 使用 SD v1.5、COCO captions 10,000 条与 Stable-Diffusion-Prompts 10,000 条，rank 320、AdamW \(10^{-4}\)、30 epoch；采样为 dpm-solver 30 步、CFG 7.5。

训练阶段使用图文对与教师 U-Net；部署/推理时用户只加载 merge 后的水印 U-Net，用普通 prompt、seed、sampler 生成图。验证阶段由所有者持有 \(D_s\) 对图像解码；\(\alpha\) 控制保真—提取权衡，论文默认 \(\alpha=1.05\)。因此“替换 VAE 仍可检出”来自水印在 U-Net，而不是验证方能够访问某个特定 VAE。

### 3.5 最小复现闭环与信息缺口

最小闭环应先固定 SD1.5、48-bit、单一粗类型：训练 \(E_s,D_s\) 至干净/攻击后可解码；冻结二者和 U-Net，仅更新 rank-320 LoRA；对同 prompt/seed 比较 clean 与 watermarked 的 FID/DreamSim、bit accuracy 和固定 FPR 的 TPR；最后更换 sampler/VAE/分辨率和做再去噪。论文未完整披露 secret encoder/decoder 的层级细节、PRVL 的全部实现以及所有 type-adaptation checkpoint 的选择规则，不能把标准图像水印网络结构当作论文事实补造。

### a) 两阶段流程

阶段一不训练扩散模型：从真实图像 \(I_o\) 经 VAE encoder 得 \(z_o\)，秘密 encoder \(E_s\) 将 bit 串映射为 \(\Delta z_w\)，得到 \(z_w=z_o+\Delta z_w\)，VAE decoder 生成 \(I_w\)。秘密 decoder \(D_s\) 从经过失真层的图像恢复 bits。这样学到的 \(\Delta z_w\) 是一个“latent codebook”，而不是随意像素噪声。

阶段二冻结原 U-Net \(\epsilon_\vartheta\)，只训练 LoRA。训练时将 \(z_0+\Delta z_w\) 的去噪预测约束为接近同一内容的干净预测：
\[
L_{PPFT}=\mathbb E\|\epsilon_\theta(\sqrt{\bar\alpha_t}(z_0+\Delta z_w)+\sqrt{1-\bar\alpha_t}\epsilon,t,c)-\epsilon_\vartheta(\sqrt{\bar\alpha_t}z_0+\sqrt{1-\bar\alpha_t}\epsilon,t,c)\|_2^2.
\]
它的作用是让“加了水印的输入”仍遵守原模型的去噪先验，而非把水印样本直接拟合成噪声真值。

### b) latent 水印预训练与损失

比特 BCE 迫使 \(D_s\) 解码，LPIPS 限制感知差异，分布相关约束限制 latent/图像统计偏移；训练中加入失真层，并使用 PRVL（Peak Regional Variation Loss）抑制局部高峰扰动。各项的分工是：BCE 保容量，失真层保传播鲁棒，LPIPS/PRVL 保可见质量。论文未把秘密 encoder 的完整网络层数写成可独立复现的规范，实际应以仓库为准。

### c) Watermark LoRA 如何携带用户 ID

普通 LoRA 是 \(\Delta W=AB\)。AquaLoRA 将 bit 的嵌入组合成对角缩放矩阵 \(S\)，再以 \(ASB\) 形成权重更新；不同 bit 串对应不同 \(S\)，不必为每位用户重新训练完整 U-Net。原权重保持冻结，合并为 \(W_{wm}=W+\alpha\Delta W\)。这降低了部署成本，但高 rank LoRA 仍有额外存储和训练开销。

### d) 粗类型适配与训练配置

为减少“一个通用水印 LoRA”与具体定制模型分布的距离，作者按粗粒度类型做额外 PPFT，再为相近 checkpoint 选用相应 AquaLoRA。阶段一：COCO2017 训练集随机 10,000 图、40 epoch、AdamW，学习率 \(10^{-3}\)，\(\lambda=5,\mu=0.5\)；阶段二：SD v1.5，COCO captions 10,000 条加 Stable-Diffusion-Prompts 10,000 条，rank 320、AdamW \(10^{-4}\)、30 epoch。嵌入 48 bit，训练采样 dpm-solver 30 步、CFG 7.5，默认 \(\alpha=1.05\)。

## 4. 与其他方法对比

| 类别 | 代表 | 水印位置 | 白盒绕过风险 | AquaLoRA 的改进 |
|---|---|---|---|---|
| 后处理 | DwtDctSvd、RivaGAN | 最终图像 | 重新生成/去噪后脱钩 | 信号进入 U-Net |
| 采样期 | Tree-Ring | 初始噪声/采样 | 改 sampler 或去噪 | 不依赖指定采样轨迹 |
| decoder 集成 | Stable Signature | VAE decoder | 替换 VAE | 写入 U-Net 权重 |
| AquaLoRA | 本文 | U-Net LoRA | 仍可能被重训练/蒸馏 | PPFT 保先验、\(S\) 支持多用户 |

## 5. 实验表现

作者以 COCO val2017 的 5,000 张结果计算 FID，并用 DreamSim 衡量水印前后布局/语义相似性；在 Civitai 下载 25 个定制 checkpoint，每个生成 100 张图评估。鲁棒实验包含 ColorJitter、Crop&Resize、Blur、Gaussian noise、JPEG，以及用干净扩散模型再去噪的擦除操作；还测 DDIM、DPM-S/DPM-M、Euler、Heun、UniPC、步数、CFG、三类 VAE、分辨率、LoRA/ControlNet。

表 2 的平均 bit accuracy：AquaLoRA 91.86%，DwtDctSvd 70.55%、RivaGAN 84.19%、Stable Signature 77.01%；在 \(\mathrm{FPR}=10^{-6}\) 下平均 TPR 为 0.906。表 3 中不同 sampler 的 bit accuracy 约 95%，说明信号不是某一个 sampler 的偶然产物；更大分辨率下下降，表明尺度泛化仍是边界。PPFT 消融中 naive diffusion 的 bit accuracy 接近随机（48.11%），而 PPFT 约 95%，是“先验保持必要”的直接证据。

## 6. 学习与应用

代码公开。复现最易出错处是 VAE latent 标准、\(\alpha\) 与 LoRA merge 方式；应同时保存用户 bit 串、secret encoder/decoder 版本与对应 coarse type。部署评估不能只报 bit accuracy，必须给出无水印 FPR 和多图聚合的归属阈值。

## 7. 总结

一句话：**将可解码 latent 模式经 PPFT 和消息化 LoRA 绑定到定制 SD 的 U-Net。**

## 8. 图表精读与证据链

图 2 对应两阶段接口：左侧训练 codebook，右侧只更新 LoRA；表 2 对应抗失真主张；表 3 对应不依赖特定推理配置；表 5 的 PPFT/rank/初始化消融对应“保真与可学性”的方法动机。证据缺口是极强白盒移除者（蒸馏、完全重训）的系统量化。

## 9. 复现难度与适合人群

难度为高：需要 SD 训练、25 类定制 checkpoint 或可替代域、私有水印网络和显存充足的 GPU。最小版本可只在 SD1.5、COCO 子集、单一 48-bit payload 上复现 codebook+PPFT。适合扩散模型版权、模型水印和 AIGC 溯源研究者。

## 10. 简短全面总结

AquaLoRA 将威胁模型从“图片经过失真”推进到“攻击者拥有定制扩散模型”。其两阶段策略先构造 VAE latent 中可见且可解码的秘密方向，再以冻结原 U-Net 的噪声预测为教师训练 Watermark LoRA；缩放矩阵将不同用户 ID 写入低秩更新。实验同时检查图像分布、布局相似性、常规失真、再去噪和推理配置，平均鲁棒指标优于比较方法。核心价值是把保护点放到 U-Net；主要局限是抗强白盒移除依然是经验性而非严格保证。

## 11. 论文写作逻辑分析

文章从开放 SD 的白盒缺口切入，按“水印图案如何可学—U-Net 如何不遗忘—多用户如何部署”展开 latent pretraining、PPFT 与 LoRA 缩放矩阵。实验与三项需求对应：FID/DreamSim 对 fidelity，攻击和配置对 robustness，类型适配/消息化 LoRA 对 flexibility。最强的论据是 PPFT 消融；最需补强的是更强模型再训练攻击。
