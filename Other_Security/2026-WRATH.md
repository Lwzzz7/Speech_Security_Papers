# WRATH: Turning Watermark Robustness Against Itself via a Watermark-Agnostic Black-Box Invalidation Attack

## 0. 摘要翻译

水印正越来越多地被嵌入 AI 生成图像中，用于标识其机器生成来源。为了可靠检测，水印通常被设计为能够抵抗缩放、压缩等常见图像处理。本文揭示了一个此前被忽视的漏洞：水印方案的鲁棒性属性可能会意外暴露承载水印信号的图像特征。基于这一点，作者提出 WRATH，这是第一个水印无关的黑盒攻击，能够同时执行水印移除与水印伪造。该攻击具有实践性，只需要知道目标方案的鲁棒性信息，而这些信息通常是公开的，或可通过简单鲁棒性测试获得。在 Amazon 水印等最先进方案上的评估显示，WRATH 能成功攻击所有测试方案，同时保持较高感知图像质量。论文还讨论了缓解该漏洞的实际防御，并指出水印安全设计需要重新思考。

## 1. 方法动机

作者要解决的问题不是“再设计一个更强水印”，而是反过来追问：如果水印为了可检测而必须对某些变换保持鲁棒，这种鲁棒性会不会成为攻击者的侧信道？现有图像生成水印研究通常把鲁棒性当作核心优点，强调经过 JPEG、resize、crop、旋转、噪声等处理后仍能被检测；但论文指出，哪些操作保留水印、哪些操作破坏水印，本身就泄露了“水印信号依附在哪类图像特征上”。

现有攻击的不足主要有三类。第一，很多移除攻击只能做 removal，不能做 forgery；第二，能伪造水印的攻击往往依赖较强假设，例如知道水印结构、拥有非水印原图、访问模型权重或针对单一水印族定制；第三，同时覆盖语义水印和非语义水印、同时支持移除和伪造、且只需黑盒二值 detector 输出的攻击，在本文之前基本缺位。

WRATH 的核心 insight 是：鲁棒变换保留水印，脆弱变换破坏水印，因此二者的差异能够定位 watermark carrier。若某个图像特征在鲁棒变换下稳定、在脆弱变换下被扰动，它就很可能承载水印。更关键的是，许多现有方案在不同图像中反复使用相对一致的 carrier 和相似的信号模式，攻击者可以跨图像聚合这些信息，训练一个 carrier encoder，随后对任意新图像做移除或伪造。

## 2. 威胁模型解读

| 项目 | 论文设定 |
|---|---|
| 模型拥有者 | 运行专有图像生成模型，并在生成图像中嵌入水印；同时提供水印检测 API。 |
| 攻击者 | 普通 API 用户，可查询生成模型得到带水印图像，也可查询 detector 得到水印检测结果。 |
| 攻击目标 | 移除：让带水印图像被检测为非水印；伪造：让非水印图像被检测为带目标水印。 |
| 攻击知识 | 不知道水印类型、嵌入算法、检测器结构、模型权重、训练数据或密钥。 |
| 攻击反馈 | 最小反馈设定：detector 只返回二值标签 “watermarked/non-watermarked”。Amazon 实际有置信等级，但论文按二值反馈处理。 |
| 攻击预算 | 需要少量 detector query 构造 robustness profile；实验中每个方案少于 100 次查询。训练 carrier encoder 使用每个目标方案 1,000 张带水印图像。 |
| 质量约束 | 攻击输出应与原图感知相似；论文主要用 LPIPS 和 L2 约束视觉质量。 |
| 防御者目标 | 在保持水印鲁棒性和可检测性的同时，避免 carrier 与信号模式跨图像过度一致，降低可学习性。 |

这个 threat model 的强点在于足够接近真实 API 场景：攻击者不需要白盒信息，也不需要 detector 的 bit accuracy 或置信度，只需要公开检测接口的二值结果。弱点也很明确：它假设同一部署周期内使用单一水印方案，并且该方案在多张图像中存在可聚合的一致 carrier 或相似信号模式。论文的 meta-review 也指出，这种 carrier consistency 是方法成立的重要前提，若未来水印使用强内容自适应 carrier 或密钥多样化信号，WRATH 的当前实例会明显变弱。

## 3. 方法设计与复现级理解

### 3.1 全局流程

WRATH 的执行顺序可以拆成四步。

第一步是数据收集。攻击者调用目标图像生成 API，得到一批由同一水印方案处理过的图像。论文实验中每个方案用 1,000 张水印图像训练 carrier encoder，另用 100 张 unseen 图像评估。

第二步是 robustness profile construction。攻击者只选一张带水印图像，对它施加一组常见图像变换，如 rotate、crop、shear、Gaussian noise、brightness、contrast、gamma correction、JPEG、resize、median filter、Gaussian blur。每次变换后查询 detector 的二值输出。若输出仍为 watermarked，该变换被认为 robust；若输出为 non-watermarked，则认为 fragile。

第三步是训练 watermark carrier encoder。它不尝试复现原水印算法，而是学习一个特征提取器，把可能承载水印的图像区域或频率分量编码成 watermark embedding。训练信号来自 robustness profile：原图和鲁棒变换图应在 carrier 表征上接近，原图和脆弱变换图应远离。

第四步是 invalidation attack。对于水印移除，优化图像使其 watermark embedding 远离原带水印图像，同时尽量只改 carrier 且保持视觉相似；对于水印伪造，优化或再生成图像，使非水印图像的 watermark embedding 接近目标水印图像集合的平均 embedding。

### 3.2 鲁棒性画像构造：用二值检测结果找 fragile/robust 操作

该模块解决的问题是：变换空间无限大，攻击者不能枚举所有参数强度。论文将每个 manipulation 拆成 type 和 parameter，例如 JPEG 是 type，quality 是 parameter；rotation 是 type，角度是 parameter。type 数量有限，parameter 可以在给定范围内搜索。

输入是一张带水印图像 $I_w$、变换类型集合 $T=\{t_1,t_2,\ldots,t_n\}$，以及每种变换的参数范围 $[\theta^{low}_{t_i},\theta^{high}_{t_i}]$。输出是两个集合 $M_{robust}$ 和 $M_{fragile}$。实际步骤是：先在最低强度查询 detector，若已经变成 non-watermarked，则最低强度就是 fragile；否则在最高强度查询，若仍是 watermarked，则取中等强度作为 robust；如果最低强度保留水印而最高强度移除水印，则在区间内二分搜索刚好使 detector 输出 non-watermarked 的强度，并把该变换加入 fragile。

这里的设计理由是很具体的。对 fragile 变换，作者希望找到“刚刚移除水印”的最低强度，因为强度再大就会破坏更多非 carrier 特征，定位会变粗。对 highly robust 变换，作者不用最高强度，而用中等强度，因为最高强度虽然 detector 仍可能返回 watermarked，但真实水印 bit accuracy 可能已经接近阈值，说明 carrier 已经被部分破坏。由于攻击者只有二值输出，无法区分“完整保留”和“勉强过阈值”，所以中等强度是更稳妥的 profile 信号。

### 3.3 Watermark Carrier Encoder：用 triplet loss 学 carrier 表征

Carrier encoder 记为 $E_{carrier}(\cdot)$。输入图像可以先经过域变换 $T$，例如空间域、频率域或 latent 域。论文实验主要按 prior work 在频率域训练，但方法本身不限定频率域。形式上：

$$
\hat I = T(I), \qquad e_w = E_{carrier}(\hat I), \qquad e_w \in \mathbb{R}^d.
$$

训练数据是目标方案生成的水印图像集合 $\{\hat I_w^{(i)}\}_{i=1}^{N}$。每个训练 triplet 包含 anchor、positive、negative。anchor 是原始水印图像 $\hat I_w$；positive 是对同一图像施加 robust manipulation 后得到的版本；negative 是施加 fragile manipulation 后得到的版本。训练目标是让 anchor 与 positive 的 embedding 接近，让 anchor 与 negative 的 embedding 远离：

$$
L_{triplet}=
\max\left(
0,\ 
\left\|E_{carrier}(\hat I_w)-E_{carrier}(\hat I_{robust})\right\|_2^2
-
\left\|E_{carrier}(\hat I_w)-E_{carrier}(\hat I_{fragile})\right\|_2^2
+m
\right).
$$

论文正文没有把 triplet loss 完整写成这个标准形式，但明确说明采用 triplet loss，并用 anchor/positive/negative 的距离关系训练 encoder；上式是对论文训练目标的标准数学展开。它的含义是：只有那些在 robust 操作下稳定、在 fragile 操作下变化的特征，才会被 encoder 偏好。单个 triplet 暴露的是 candidate carrier set；跨大量图像和不同 robust-fragile 组合聚合后，encoder 才能收敛到跨样本一致的 carrier。

论文还加入 curriculum learning。原因是不同 triplet 的难度不同：candidate carrier set 大的 triplet 更容易优化，但也更可能混入非 carrier 特征；candidate carrier set 小的 triplet 更难，但更精确。作者先用较容易的 triplet 让模型找到大致位置，再逐步引入更难 triplet，迫使 encoder 聚焦更精确的 carrier。附录 B.2 用 candidate set 的 IoU proxy 与 triplet loss 的相关性支撑这一点，五个方案的相关系数为 -0.85 到 -0.99，说明候选 carrier 区域越小，训练损失越大。

### 3.4 移除攻击：把 watermark embedding 推离原图

给定一张新带水印图像 $I_w$ 和训练好的 $E_{carrier}$，移除攻击要生成 $I_{adv}$，使其视觉上接近 $I_w$，但 detector 判为 non-watermarked。目标由三部分组成：carrier disruption、location control、visual fidelity。

Carrier disruption 直接最大化原图和攻击图的 watermark embedding 距离：

$$
L_{carrier}
=
\left\|
E_{carrier}(T(I_w))-E_{carrier}(T(I_{adv}))
\right\|_2.
$$

因为最终优化是 minimization，论文在总目标中使用 $-L_{carrier}$，即距离越大越好。

仅拉远 embedding 可能破坏大量非 carrier 图像内容，因此作者引入 CAM 约束。先对 $E_{carrier}$ 在输入 $T(I_w)$ 上计算 Class Activation Map，得到二值 mask $X$，其中 $X_{i,j}=1$ 表示位置 $(i,j)$ 被视为 carrier。设：

$$
\delta = \left|T(I_w)-T(I_{adv})\right|.
$$

位置损失为：

$$
L_{location}
=
-\lambda_1
\frac{\sum_{i,j} X_{i,j}\left|\delta_{i,j}\right|}
{\sum_{i,j}\left|\delta_{i,j}\right|+\epsilon}
+
\lambda_2
\frac{\sum_{i,j}(1-X_{i,j})\left|\delta_{i,j}\right|}
{\sum_{i,j}\left|\delta_{i,j}\right|+\epsilon}.
$$

第一项带负号，鼓励修改集中在 carrier 区域；第二项为正，惩罚非 carrier 区域修改。$\epsilon$ 只用于避免分母为零。这个损失是 WRATH 区别于普通 adversarial perturbation 的关键：它不是盲目扰动整图，而是把扰动预算集中到由 robustness profile 学出来的 carrier 上。

视觉保真项使用 LPIPS 和 L2 的阈值惩罚：

$$
L_{perceptual}
=
\alpha_1 \cdot \mathrm{ReLU}
\left(L_{LPIPS}(I_w,I_{adv})-\tau_{LPIPS}\right)
+
\alpha_2 \cdot \mathrm{ReLU}
\left(\left\|I_w-I_{adv}\right\|_2-\tau_{L2}\right).
$$

ReLU 表示只有超过阈值才罚，因此它不是持续压低图像差异，而是把图像质量控制在可接受范围内。最终移除目标为：

$$
I_{adv}
=
\arg\min_{I'}
\left(
-L_{carrier}(I_w,I')
+L_{location}(I_w,I')
+L_{perceptual}(I_w,I')
\right).
$$

低扰动水印通常表现为噪声状像素或频率扰动，WRATH 直接优化像素级或频域修改即可。高扰动水印会改变结构性内容，例如 Tree-Ring 这类模式，单像素独立修改不足以移除，因此论文沿用 UnMarker 中的卷积式方法，学习一组内容自适应线性卷积核并顺序作用于图像，以捕获像素间结构关联。

### 3.5 伪造攻击：把非水印图像推向目标水印分布

给定非水印图像 $I_{nw}$ 和目标方案生成的一组带水印图像 $\{I_w^{(i)}\}_{i=1}^{N}$，伪造目标是生成 $I_{adv}$，使 detector 将其误判为 watermarked。

优化式伪造直接继承移除攻击框架，但方向相反：不是让 embedding 远离水印图像，而是让非水印图像的 embedding 接近目标水印集合的平均 embedding。论文选择平均 embedding，是因为某些水印可能随图像内容有一定变化，单张水印图像的 embedding 会混入内容因素；跨样本平均更能保留共同水印信号：

$$
\bar e_w
=
\frac{1}{N}\sum_{i=1}^{N}E_{carrier}(T(I_w^{(i)})).
$$

伪造中的 carrier 项可以写为：

$$
L_{carrier}^{forge}
=
\left\|
E_{carrier}(T(I_{adv}))-\bar e_w
\right\|_2.
$$

整体优化目标保留位置约束和感知约束：

$$
I_{adv}
=
\arg\min_{I'}
\left(
L_{carrier}^{forge}(I',\{I_w^{(i)}\}_{i=1}^{N})
+L_{location}(I_{nw},I')
+L_{perceptual}(I_{nw},I')
\right).
$$

对高扰动水印，论文进一步提出 regeneration-based forgery。训练阶段先构造水印图像与非水印图像对，其中非水印图像可由 WRATH removal 得到；用已训练的 $E_{carrier}$ 从带水印图像提取 $e_w$，同时训练一个 CNN encoder 从非水印内容中提取 content embedding $e_c$，再用 deconvolutional decoder 从 $[e_c,e_w]$ 重建原带水印图像。攻击阶段，对任意 clean non-watermarked image 提取 $e_c$，再拼接平均水印 embedding $\bar e_w$，由 decoder 生成 forged image。损失是重建图像与原带水印图像的 L2 距离。

论文最后把两类伪造方式统一成选择策略：低扰动方案用 optimization-based forgery，因为需要注入噪声状细粒度扰动；高扰动方案用 regeneration-based forgery，因为 CNN decoder 更擅长生成结构化水印模式。附录 C 在 StegaStamp 上显示 regeneration-based forgery 平均 SR 为 72.5%，明显高于 optimization-based 的 23%。

### 3.6 训练、部署与攻击阶段分离

离线训练阶段需要查询目标生成器收集水印图像、查询 detector 建 robustness profile、训练 carrier encoder。这个阶段是“一次性成本”，针对一个水印方案完成后，encoder 可以复用于任意数量的新图像。

在线攻击阶段不需要再训练 encoder。移除时输入一张带水印图像，运行优化或卷积核生成 $I_{adv}$，必要时可用 detector 验证是否已变成 non-watermarked。伪造时输入一张非水印图像，再使用目标方案水印图像集合的平均 embedding 生成 forged image。需要注意的是，论文没有公开完整攻击代码或 artifacts，且伦理声明明确会 withholding code and artifacts，因此复现需要自行实现 profile search、carrier encoder、CAM、优化循环和再生成模型。

### 3.7 复现配置表

| 复现项 | 论文给出的设置 |
|---|---|
| 目标方案 | Amazon Titan；DWT-DCT；HiDDeN；Stable Signature；StegaStamp；Tree-Ring；Gaussian Shading。 |
| 方案类型 | Amazon、DWT-DCT、HiDDeN、Stable Signature 为 low-perturbation；StegaStamp、Tree-Ring、Gaussian Shading 为 high-perturbation。 |
| 语义/非语义 | DWT-DCT、HiDDeN、Stable Signature 为非语义；StegaStamp、Tree-Ring、Gaussian Shading 为语义；Amazon 论文未公开其内部类型，只按 profile 归为 low-perturbation。 |
| 鲁棒性 profile | 12 类常见图像变换，覆盖 geometry-based 和 value-based manipulation。 |
| 查询预算 | 每个方案构造 profile 少于 100 次 detector 查询；表 6 中 Amazon 84、DWT-DCT 85、Stable Signature 83、HiDDeN 92、StegaStamp 59、Tree-Ring 61、Gaussian Shading 53。 |
| 训练数据 | 每个方案 1,000 张带水印图像训练 carrier encoder，100 张 unseen 图像评估。 |
| 训练域 | 频率域；论文说明框架可迁移至空间域或 latent 域，但主实验用频率域。 |
| 检测阈值 | 学术方案使用 EER 阈值；Amazon 使用其 API 输出，论文将置信等级映射为 0/1/2/3 报告均值。 |
| 质量指标 | LPIPS；小于 0.1 被认为 perceptually acceptable。 |
| 代码与 artifacts | 伦理部分明确不公开可复现攻击的代码和 artifacts。 |

## 4. 与其他方法对比

| 方法类别 | 核心思想 | 优点 | 缺点 | WRATH 的区别 |
|---|---|---|---|---|
| VAEAttack / DiffusionAttack | 用生成模型重建图像，破坏水印 | 黑盒、简单、适合部分非语义水印 | 对高扰动或语义水印不稳定，质量和成功率难兼顾 | WRATH 不盲目重建，而是先学习 carrier，再定向修改。 |
| UnMarker | crop 加频域扰动移除水印 | 对若干方案有效，实践性较强 | 主要面向移除；依赖 crop；不覆盖所有方案 | WRATH 可无 crop 工作，并统一支持 removal 与 forgery。 |
| Watermark-White-Noise | 生成带水印白噪声并叠加到 clean image | 黑盒且可用于伪造 | 对多数方案增益有限，信号迁移粗糙 | WRATH 学习目标方案的 carrier 与平均 watermark embedding。 |
| WMCopier | 训练扩散模型学习水印分布再注入 | 对部分非语义方案有效 | 需要适配/预训练，覆盖方案有限，难处理语义水印 | WRATH 以 robustness profile 为入口，覆盖语义和非语义方案。 |
| WRATH | 从 robust/fragile 变换推断 carrier，再做定向移除/伪造 | 黑盒、少量 query、统一 R+F、跨七个方案有效 | 依赖 carrier/signature 跨图像一致性；代码不公开；物理世界未充分验证 | 把“鲁棒性”本身变成攻击信号。 |

创新点主要在三处。第一，它把 robustness profile 当作侧信道，提出“水印越强调对某些变换鲁棒，越可能暴露 carrier”的安全视角。第二，它把 carrier localization 和 invalidation attack 分离，使训练好的 carrier encoder 同时服务于 removal 和 forgery。第三，它在最小反馈黑盒设定下覆盖 Amazon 和六个学术方案，证明该漏洞不是单一算法特例。

## 5. 实验表现与优势

### 5.1 RQ1：carrier encoder 是否真的学到水印 carrier

动机是验证 WRATH 的核心前提：robust/fragile profile 是否足以定位 carrier。设置上，作者对六个学术方案训练 carrier encoder，并与三类 baseline 比较：只用 fragile manipulation 构造的分类器、用 OOD 非水印图像训练的分类器、用 in-distribution 非水印图像训练的强 baseline。Amazon 因缺少 in-distribution 非水印 counterpart，被排除在该项评估之外。

指标一是分类准确率，即用学到的 carrier 特征区分 watermarked 与 non-watermarked counterpart 的能力。平均结果中 WRATH 达到 83.1%，高于 baseline 1 的 75.8%、baseline 2 的 80.5% 和 baseline 3 的 80.4%。分方案结果中 WRATH 在 DWT-DCT 为 88.0%、HiDDeN 为 81.0%、Stable Signature 为 91.0%、StegaStamp 为 74.5%、Tree-Ring 为 70.4%、Gaussian Shading 为 93.57%，所有方案都保持 70% 以上。

指标二是 CAM 与 ground-truth carrier 的 cosine similarity。ground truth 由 watermarked 与 original non-watermarked 图像在频域的 pixel-wise difference 估计，只能用于 DWT-DCT、HiDDeN、StegaStamp 这类有原图 counterpart 的方案。图 5 显示 WRATH 在这些方案上整体 CAM 对齐更好，说明它不是只学到数据源差异，而是真能靠 profile 聚焦 carrier。

### 5.2 RQ2：水印移除效果

移除实验的动机是验证 WRATH 是否比已有 practical removal attack 更稳定。比较对象包括 VAEAttack、DiffusionAttack、UnMarker w/o cropping、UnMarker w/ cropping，以及 WRATH w/ 和 w/o cropping。指标为 SR 和 LPIPS。SR 表示带水印图像被误判为 non-watermarked 的比例；LPIPS 衡量感知质量，论文以 SR > 50% 且 LPIPS < 0.1 作为有效且质量可接受的重要参考。

表 2 的关键结论是 WRATH 是唯一在所有七个方案上都达到有效移除的攻击。对于 Amazon，WRATH 的 confidence score 可降至 0.06，SR 达 95%，LPIPS 为 0.0185；对于 DWT-DCT，WRATH w/o cropping SR 为 84%，LPIPS 0.1717，w/ cropping SR 为 83%、LPIPS 0.1388；对于 HiDDeN，WRATH w/o cropping SR 为 77%、LPIPS 0.0037，w/ cropping SR 为 73%、LPIPS 0.0400；对于 Stable Signature，WRATH w/o cropping SR 为 84%、LPIPS 0.0247。对 Tree-Ring，WRATH w/o cropping SR 为 51%、LPIPS 0.0901，w/ cropping SR 为 70%、LPIPS 0.0987。

高扰动方案更难保持质量。StegaStamp 和 Gaussian Shading 上 WRATH 能有效移除，但部分设置 LPIPS 超过 0.1 或出现可见 artifacts。论文的解释是：这类水印涉及结构性 carrier，单纯低幅扰动不足以破坏，必须引入更强的结构性修改。即便如此，WRATH 仍是唯一无需专门知道方案内部即可覆盖这些方案的 practical attack。

### 5.3 RQ3：水印伪造效果

伪造实验测试 WRATH 能否把非水印图像改造成会被目标 detector 误判为 watermarked。非水印图像来自 Stable Diffusion v1.4、v2.0、v2.1 和 SD-XL。baseline 包括 WhiteNoiseAtk 和 WMCopier；WMCopier 只覆盖 DWT-DCT、Stable Signature、StegaStamp。

表 3 显示 WRATH 在七个方案上都稳定提升 SR。平均 $\Delta SR$ 分别为：Amazon 10.5%、DWT-DCT 68.5%、HiDDeN 28.75%、Stable Signature 62.75%、StegaStamp 69.75%、Tree-Ring 44.25%、Gaussian Shading 98.75%。作者汇总称，WRATH 相比 White-Noise Attack 平均提升 38.7%，相比 WMCopier 平均提升 36.75%，同时输出图像平均 LPIPS 低于 0.1。

不同水印类型对应不同伪造机制。低扰动水印适合 optimization-based forgery，因为目标是注入细粒度噪声状 pattern；高扰动水印适合 regeneration-based forgery，因为结构化 pattern 更像生成问题。附录 C 对 StegaStamp 的对照显示 regeneration-based SR 平均 72.5%，而 perturbation-based 只有 23%，直接支撑这个分流设计。

### 5.4 RQ4：防御分析

论文将 WRATH 成功归因于两个现象。第一，许多方案使用跨图像一致的 carrier。DWT-DCT、HiDDeN、StegaStamp 的 intra-scheme Jaccard 为 0.52，显著高于 inter-scheme 的 0.16，Mann-Whitney test 的 $p<0.001$。防御方向是避免固定 carrier，让 encoder 根据图像内容动态选择 carrier。RivaGAN 被作为例子：其 intra-scheme Jaccard 为 0.21，接近与 HiDDeN/StegaStamp 的 inter-scheme 值 0.17/0.22，WRATH 难以可靠识别其 carrier。

第二，许多方案跨图像使用相似 watermark signal pattern。DWT-DCT、HiDDeN、StegaStamp 的 intra-scheme carrier residual cosine similarity 为 0.91，高于 inter-scheme 的 0.55，且 $p<0.001$。改变 embedded binary string 也未显著改变这种相似性；在 10 到 50 个 distinct strings 条件下，多数方案仍可被 WRATH 学到信号。防御方向是引入 key-dependent pseudorandom watermark signal。Gaussian Shading 在使用更多 distinct keys 后，100 张图像、超过 20 个 distinct keys 时 WRATH 分类准确率降到约 50%。但论文也指出，当前密码学式 undetectable watermark 往往对几何错位极其脆弱，轻微平移就可能破坏水印。

## 6. 学习与应用

论文没有开源攻击代码。伦理部分明确说明作者会 withheld code and artifacts that could be used to replicate the attack。因此学习价值主要在方法思想、实验设计和防御启发，而不是直接复现跑通。

若要做最小复现，可以先避开 Amazon 这类专有接口，从开源学术水印方案入手：选 DWT-DCT 或 HiDDeN，生成 1,000 张带同一 bit string 的水印图像；实现 12 类常见变换的 profile search；在频域训练 triplet carrier encoder；用 CAM 生成 carrier mask；最后实现移除优化目标，先复现 RQ1 的 carrier 分类准确率和 RQ2 的 SR/LPIPS。伪造部分工程量更高，尤其 regeneration-based decoder 需要训练水印/非水印配对重建模型，适合作为第二阶段。

迁移到音频水印时，思想上可对应为：robust audio transforms 如压缩、重采样、加噪、滤波、裁剪、混响、重录是否泄露水印 carrier 所在的时频区域或 codec latent 位置。真正迁移的难点在于，音频 detector 可能输出连续置信度、音频质量约束更依赖 PESQ/STOI/MOS 或说话人相似度，而且时间同步攻击会让 carrier localization 比图像频域更复杂。

## 7. 总结

一句话：**用水印鲁棒性画像反推 carrier，并同时移除和伪造水印。**

## 8. 图表精读与证据链

- **图 1**：给出 Amazon 场景下 WRATH 的完整攻击链：收集水印图像、用一张图构造 robustness profile、训练 carrier encoder、对新图像做 removal/forgery。它直接支撑“黑盒、少量 query、一次训练多次攻击”的 claim。
- **图 2**：用 JPEG compression 展示三种鲁棒性行为：DWT-DCT highly fragile、Stable Signature highly robust、HiDDeN partially robust。它解释了为什么 profile search 需要最低强度、最高强度和二分搜索，而不是固定参数测试。
- **图 3**：统一 invalidation pipeline，按 low/high-perturbation 和 removal/forgery 分成四条路径。它说明 WRATH 不是一个单一扰动公式，而是根据 profile 推断水印类型后选择攻击机制。
- **图 4/5/8**：验证 carrier encoder 的有效性。图 4 是分类准确率，图 5 是 CAM 与 ground truth carrier 的余弦相似度，图 8 是可视化证据；三者共同支撑“WRATH 学到 carrier 而非偶然分类”的证据链。
- **表 2**：移除攻击主结果。它证明 WRATH 是唯一覆盖所有七个方案的有效 practical removal attack，但也暴露 StegaStamp/Gaussian Shading 上可见 artifacts 的质量边界。
- **表 3**：伪造攻击主结果。它证明 WRATH 在七个目标方案和四类非水印图像源上都能提升 SR，并且平均 LPIPS 低于 0.1。
- **图 6**：防御相关证据。改变 bit string 不足以阻止 WRATH，说明问题不只是固定 payload，而是 common signal pattern。
- **表 4/6**：复现和评估关键表。表 4 解释 EER 阈值选择，表 6 给出每个方案的 profile、阈值和 query 数，是复现 profile construction 的核心依据。
- **Meta-review**：接受理由承认该工作揭示 robustness/security tension，但 noteworthy concerns 明确指出 carrier consistency 可能不是永远成立，并且实验设置与真实部署仍有差距。这部分给论文边界提供了外部审稿证据。

## 9. 复现难度与适合人群

复现难度为高。原因不是单个公式复杂，而是需要同时搭建多个水印方案、生成训练图像、实现图像 manipulation 搜索、训练 carrier encoder、做 CAM 定位、再分别实现 removal 和 forgery 优化。Amazon 相关实验还依赖真实服务/API 行为和费用，论文虽给出约 1,000 张图像成本约 US$14.40，但没有公开攻击代码和 artifacts。

最小可复现版本建议从 DWT-DCT 或 HiDDeN 开始，只复现 robustness profile、triplet carrier encoder、carrier classification accuracy 和 removal SR/LPIPS。完整复现七个方案、尤其包含 Tree-Ring/Gaussian Shading 的高扰动伪造，需要更多工程实现和算力。

适合阅读人群包括图像/音频水印安全研究者、生成式内容溯源系统设计者、做 watermark attack/defense benchmark 的研究者，以及希望理解“鲁棒性如何变成信息泄露”的安全评估人员。对纯应用读者而言，最重要的 takeaway 是：不能只报告经过常见变换后的检测率，还要评估鲁棒性 profile 是否泄露固定 carrier 或共同信号。

## 10. 简短全面总结

WRATH 研究的是生成图像水印中一个反直觉风险：水印为了抵抗 JPEG、resize、crop、噪声等变换而表现出的鲁棒性，会泄露承载水印的图像特征。攻击者只需黑盒访问生成 API 和二值 detector，用一张水印图像测试常见变换，区分 robust 与 fragile manipulation，再用 1,000 张水印图像训练 carrier encoder。该 encoder 学到在 robust 变换下稳定、在 fragile 变换下被破坏的 watermark carrier，并将其编码为 watermark embedding。移除时，WRATH 优化图像使 embedding 远离原水印图，同时用 CAM mask 和 LPIPS/L2 控制扰动位置与视觉质量；伪造时，则把非水印图像的 embedding 推向目标水印集合的平均 embedding，或训练 encoder-decoder 再生成结构性水印。实验覆盖 Amazon Titan 和六个学术方案，显示 WRATH 能移除和伪造全部测试水印，且多数场景保持较高视觉质量。论文的主要贡献是把 robustness profile 识别为水印安全侧信道；关键边界是攻击依赖跨图像一致 carrier 或 common signal pattern，且作者不公开可复现攻击代码。

## 11. 论文写作逻辑分析

论文的写作主线很清楚：先从监管和工业部署说明水印检测的重要性，再指出社区过度强调 robustness，却较少把 robustness 当作可观测攻击面。Introduction 中最有效的一步是引入三组概念：watermark carrier、fragile manipulation、robust manipulation。这样读者可以自然接受后续“由 profile 反推 carrier”的方法，而不是把它看成凭空训练一个分类器。

Threat model 放在方法前面是必要的，因为 WRATH 的贡献很大程度上来自 practical black-box assumption。作者明确限定攻击者只有二值 detector 输出、没有水印算法知识、没有权重和训练数据，并强调 query 数少于 100。这使方法设计中的 profile search、triplet encoder 和平均 embedding 都能回扣 threat model。

方法叙事按真实攻击流程展开：先构造 profile，再训练 carrier encoder，最后做 removal/forgery。每个模块都回答一个前置问题：如何少 query 找 robust/fragile 操作；如何把这些操作转化为可学习 carrier；如何把 carrier 表征转化为失效攻击。尤其 4.4 把 low/high-perturbation 区分放在攻击选择里，避免用同一个公式解释所有现象。

实验组织与 claim 基本闭环。RQ1 验证 carrier encoder 是否有效，RQ2/RQ3 分别验证 removal 和 forgery，RQ4 回到 root cause 和 defense。最强证据是表 2/表 3 的跨七方案结果，以及图 4/5/8 对 carrier localization 的独立验证。较弱之处是防御只做初步经验分析，真实部署下动态 carrier、密钥轮换、限流策略和自适应 detector 的组合防御没有完整评估。论文最后把 meta-review 附上，也让边界更透明：WRATH 的实用性依赖一致 carrier 假设，而统一 removal/forgery pipeline 的“必然价值”仍可讨论。
