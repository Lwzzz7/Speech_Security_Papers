# AquaLoRA: Toward White-box Protection for Customized Stable Diffusion Models via Watermark LoRA

## 0. 摘要翻译

本文面向定制 Stable Diffusion 模型的白盒版权保护。已有图像后处理、初始噪声或 VAE 解码器水印，在攻击者能获取整个模型时可通过替换 VAE 或采样配置被绕开。AquaLoRA 将水印学习进 U-Net：先在潜空间训练可鲁棒解码的秘密水印模式，再用先验保持微调使 U-Net 学会该模式，并以携带消息的 Watermark LoRA 合并权重。方法兼顾图像保真、抗失真、采样配置变化和多用户部署，并在多种定制 SD checkpoint 上验证。

## 1. 论文基本信息

- 标题：AquaLoRA: Toward White-box Protection for Customized Stable Diffusion Models via Watermark LoRA
- 作者：Weitao Feng, Wenbo Zhou, Jiyan He, Jie Zhang, Tianyi Wei, Guanlin Li, Tianwei Zhang, Weiming Zhang, Nenghai Yu
- 发表：ICML 2024，PMLR 235
- 代码：<https://github.com/Georgefwt/AquaLoRA>
- 对象：定制的文本到图像 Stable Diffusion 模型（以 SD v1.5 为主）。

## 2. 研究背景与动机

开放社区会分发或售卖 DreamBooth/LoRA 等定制扩散模型。若水印只位于生成图像、初始噪声或 VAE decoder，拥有模型权重的攻击者可改采样器、替换 VAE 或用干净模型再去噪，从而绕过保护。真正的白盒保护应把归属信号绑定到 U-Net 的生成知识中：强行清除水印应同时损害生成质量。

## 3. 问题定义与威胁模型

模型所有者将 \(L\)-bit 身份 \(m\) 写入定制 SD 的 U-Net，之后任意人可用不同 prompt、采样器、步数、CFG、分辨率或兼容 VAE 生成图像；所有者通过私有解码器从图像恢复 \(m\)。攻击者拥有水印模型的白盒访问，能够替换 VAE、改变采样、追加其他 LoRA/ControlNet、做图像失真或再微调。目标是在这些操作下保持可检出，同时保持水印模型与原定制模型的视觉分布和语义布局接近。

该工作并不声称在攻击者愿意牺牲模型质量、从头蒸馏或重新训练时仍绝对不可移除；其安全直觉是水印与 U-Net 的核心能力耦合，移除存在效用代价。

## 4. 核心洞见

仅把随机 bit 映射到图像像素不适合扩散过程：去噪会放大潜变量扰动，且直接用常规扩散目标微调会遗忘原模型先验。AquaLoRA 因而先学习“U-Net 容易吸收、VAE 能显现”的潜空间水印方向 \(\Delta z_w\)，再用原模型的噪声预测作为教师来约束水印模型；最后把用户消息编码到 LoRA 的缩放矩阵，以较低的增量权重支持多用户。

## 5. 方法概述

1. **潜空间水印预训练**：秘密编码器把 \(m\) 映射为 \(\Delta z_w\)，与 VAE latent 相加；秘密解码器从经 VAE 解码及失真的图像恢复比特。
2. **鲁棒与保真损失**：训练含比特 BCE、感知相似度、分布约束及失真层；Peak Regional Variation Loss（PRVL）进一步抑制局部显著扰动。
3. **Watermark LoRA**：采用 \(\Delta W=AB\) 的 LoRA，并由每个 bit 的嵌入构造对角缩放矩阵 \(S\)，令消息进入 \(A S B\) 的低秩更新；合并后水印位于 U-Net 权重而非外接 decoder。
4. **PPFT**：对加水印 latent 的输入，要求水印 U-Net 的噪声预测匹配冻结原 U-Net 对干净 latent 的预测，以保留原有生成先验。
5. **粗类型适配**：为相近的定制模型类型训练通用 AquaLoRA，部署时按模型分布选择匹配类型，而非每个用户重新训练整个框架。

## 6. 关键技术细节

PPFT 的关键目标可写为让 \(\epsilon_\theta(\sqrt{\bar\alpha_t}(z_0+\Delta z_w)+\sqrt{1-\bar\alpha_t}\epsilon,t,c)\) 匹配冻结原模型 \(\epsilon_{\vartheta}(\sqrt{\bar\alpha_t}z_0+\sqrt{1-\bar\alpha_t}\epsilon,t,c)\)。这不是把水印样本强行拟合到噪声真值，而是保持原模型在对应内容上的去噪行为。

默认配置嵌入 48 bit。预训练使用 COCO2017 的 10,000 张训练图像；PPFT 使用这些 caption 与 Stable-Diffusion-Prompts 的 10,000 个 prompt。SD v1.5 上 LoRA rank 为 320，PPFT 训练 30 epoch；采样时通过 \(\alpha\) 在保真与提取率间调节，主实验取 \(\alpha=1.05\)。

## 7. 实验设置

- 定制模型：从 Civitai 下载 25 个 checkpoint，每个生成 100 张图像；prompt 来自 PartiPrompts（去除过短 basic 类）。
- 指标：FID（COCO val2017 5,000 图）、DreamSim；比特准确率和在 \(\mathrm{FPR}=10^{-6}\) 下的 TPR。
- 对比：DwtDctSvd、RivaGAN、Stable Signature、Tree-Ring watermark。
- 攻击/扰动：ColorJitter、Crop&Resize、Blur、Gaussian noise、JPEG、基于干净扩散模型的 denoising；另测采样器、步数、CFG、输出尺寸、替换 VAE、追加 LoRA/ControlNet 与微调攻击。

## 8. 实验结果与分析

- 多失真平均 bit accuracy 为 91.86%，高于表中的 DwtDctSvd（70.55）、RivaGAN（84.19）和 Stable Signature（77.01）；在 \(\mathrm{FPR}=10^{-6}\) 下平均 TPR 为 0.906。
- AquaLoRA 对 DDIM、DPM-S/DPM-M、Euler、Heun、UniPC 的提取率接近（约 95%），对不同 CFG、采样步数和三类兼容 VAE 也较稳定，验证了水印不依赖特定 decoder 或 sampler。
- 分辨率增大时提取率下降但仍可用；这说明潜空间图案并非完全尺度不变，需要训练期增广与 decoder-only 微调补偿。
- 与 Tree-Ring 相比，作者强调同 seed 下主体布局更接近原模型；PPFT 的消融表明直接扩散微调会令 bit accuracy 接近随机，而先验保持是有效嵌入的关键。

## 9. 优点与局限

优点：明确处理白盒模型而非只保护单张生成图；水印绑定 U-Net，能跨 VAE/采样配置；48-bit 载荷与缩放矩阵设计支持多用户；PPFT 对保真问题有清晰针对性。

局限：验证仍依赖保密的编码/解码网络；“移除即损伤质量”是经验结论而非加密保证；高 rank（320）LoRA 带来存储和训练成本；对模型蒸馏、恶意再训练、潜空间不兼容的全新架构，覆盖有限；只用生成图验证时，归属结论仍取决于检验阈值和样本数量。

## 10. 可复现性与使用建议

复现时应严格区分模型水印和后处理图像水印，固定 48-bit payload、COCO/PartiPrompts 划分和 FPR 阈值。白盒攻击至少应包含 VAE 替换、采样器迁移、再去噪、微调、LoRA merge/unmerge；同时报告 FID、语义相似度、bit accuracy、TPR/FPR 和模型尺寸。部署中应保存每个用户的消息—LoRA—密钥映射，并避免公开秘密解码器。

## 11. 论文写作逻辑分析

文章先以“开放权重令既有水印失效”建立白盒问题，再把水印位置锁定到 U-Net，继而逐一补齐三项工程矛盾：潜空间预训练解决可显现性与鲁棒性，PPFT 解决先验漂移，Watermark LoRA 解决可扩展的用户编码。实验也按相同逻辑展开：先保真，再失真、配置与白盒操作鲁棒性，最后做 PPFT/rank/类型适配消融。
