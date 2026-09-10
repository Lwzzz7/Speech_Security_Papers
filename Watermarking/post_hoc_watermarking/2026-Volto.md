# Volto：Unlearnable Examples 与 Watermarking 协同的双功能语音数据保护

## 0. 翻译摘要原文

语音数据是现代人工智能系统的重要资源，支撑着从说话人识别（SR）到 AI 生成内容的多种应用。对海量数据的依赖引发了严重的隐私和产权问题，因为这些数据可能被未经授权地滥用。遗憾的是，目前还没有有效方法能够在满足感知音频质量要求的同时，同时保护语音数据的隐私和版权。为弥合这一缺口，本文提出 Volto：一种统一的双功能框架，用于生成不可学习且可追踪的语音样本。Volto 联合整合 unlearnable perturbations 和 learnable watermarks，同时利用人类听觉感知和深度神经网络表示中的弱点。这种协同设计使 Volto 能够破坏关键的模型敏感特征，阻碍未经授权的模型学习，同时编码可恢复的水印信号用于所有权验证。生成的 unlearnable examples 能在保持感知质量的同时混淆 DNN，在黑盒场景下使未授权模型准确率降低超过 21.44%。Volto 还引入了可验证水印，为数据所有权提供鲁棒证据。本文工作有效平衡了保护效果和可用性，为语音隐私提供了一种有效防御。

## 1. 方法动机

Volto 处理的问题不是“生成的音频是否为假”，而是“用户公开或贡献的真实语音是否会被未经授权地拿去训练模型”。语音数据既包含隐私属性，也包含知识产权属性：攻击者或模型开发者可以收集个人语音，用于 speaker recognition、voice cloning、AIGC 或其他语音系统训练。单纯匿名化可能破坏可用性，访问控制无法阻止数据一旦泄露后的训练滥用，传统音频水印又只能事后证明某段音频归属，不能阻止模型从数据中学习。

论文把现有方案的问题拆成两部分。第一，unlearnable examples 可以让训练数据对模型“不可学习”，但音频比图像更敏感，简单把图像领域的扰动迁移到语音会产生明显失真；已有音频 unlearnable 方法还常依赖目标模型、训练数据或 MFCC 特征假设，在黑盒现代 SR 模型上不稳。第二，watermarking 能提供版权证据，但如果非合作模型开发者拒绝配合验证，传统“检测音频里有没有水印”的方式不能直接证明模型是否训练过这些数据。

Volto 的核心 insight 是：把 unlearnable perturbation 和 learnable watermark 放进同一个保护流程里。第一阶段先移除说话人相关频率组合，使数据难以训练出有效 speaker encoder；第二阶段再加入 speaker-irrelevant watermark，让非法模型更容易学习水印触发特征而不是真实声纹特征。这样，水印不仅用于事后追踪，也作为“诱导模型学习错误特征”的一部分增强不可学习性。

## 2. 威胁模型解读

### 2.1 参与方与系统边界

参与方包括数据用户、未经授权的模型开发者和验证者。用户拥有语音数据，希望在公开分享、平台上传或数据贡献时保护隐私和产权。模型开发者未经同意收集这些语音，用来训练 speaker classification / speaker recognition 模型或进一步构建 speaker encoder。验证者通常是数据所有者或其代理，事后怀疑某模型使用了自己的数据时，通过水印触发查询验证。

系统边界内包括原始数据集 $D_c$、被保护数据集 $D_p$、测试集 $D_t$、辅助 i-vector speaker recognition model、频谱裁剪算法、ACO 频率选择、time-domain / frequency-domain watermark embedding，以及水印验证查询。系统边界外包括攻击者模型架构、参数、训练数据组成、数据清洗和压缩预处理。

### 2.2 攻击者目标

攻击者目标是从未经授权收集的语音中训练出有效模型，提取 speaker-specific signals，从而提升 speaker recognition、voice cloning 或其他语音任务性能。若攻击者发现数据可能被保护，还可能通过下采样、MP3、量化等处理尝试削弱 unlearnable/watermark 效果。

### 2.3 攻击者能力

论文假设攻击者可收集用户语音并训练自己的模型。用户对攻击者模型信息了解有限，不知道模型架构、参数和完整训练集组成，除自己的贡献数据外没有更多训练数据知识。因此 Volto 主要是黑盒数据级防御。实验还考虑攻击者/开发者可能对数据做 compression defenses，包括 downsampling、MP3 compression 和 quantization。

### 2.4 攻击者知识

攻击者可能知道数据保护方法的大致存在，但论文没有把攻击者设定为能访问 Volto 的所有内部参数、watermark generator、discriminator 或 ACO 选择结果。模型开发者是否知道每个用户对应的 time-domain watermark interval 或 frequency-domain watermark pattern，论文没有完整白盒化讨论。主要结论建立在用户不知道目标模型、攻击者不主动白盒优化去除 Volto 的设定上。

### 2.5 攻击约束

保护后的语音必须保持可用。论文用 PESQ 和 SNR 作为质量约束：PESQ 低于 1.99 视为极差、内容不可懂；SNR 低于 6 dB 通常被认为质量差。因此有效保护方法不能只追求降低模型准确率，还必须让语音仍可听、可传播、可用于合法用途。

### 2.6 防御者能力与安全目标

防御者能在发布或贡献语音前修改自己的数据，生成 protected examples。目标有两个：第一，降低未经授权模型在 SR 任务上的 ACC 或提高 EER，使数据对训练者不再有吸引力；第二，在怀疑某模型使用了保护数据后，通过水印查询观察模型是否对特定水印敏感，从而提供数据所有权证据。

### 2.7 现实性评估

Volto 的威胁模型贴近“用户只能控制自己数据、无法控制训练者”的现实场景，也比需要目标模型白盒信息的方法更实用。但它仍有边界：论文主要以 speaker classification / speaker recognition 作为代理任务，推断 speaker encoder 无法学习就会影响更广泛语音系统；对 voice cloning、TTS 或 SSL pretraining 的直接验证不足。水印验证依赖被训练模型对 trigger 的行为差异，若攻击者使用强数据清洗、对抗训练、去后门检测或只使用很少比例的保护数据，证据强度可能下降。

## 3. 方法设计与复现级解读

### 3.1 总体流程：先移除声纹相关信号，再加入说话人无关水印

Volto 是两阶段流程。第一阶段叫 Speaker-Specific Signal Removal，目标是削弱音频中对说话人识别最关键的频谱成分。它不是全频加噪，而是通过纵向裁剪和横向裁剪处理频谱：纵向裁剪降低某些频率能量，横向裁剪删除最影响 speaker similarity 的频率组合。第二阶段叫 Speaker-Irrelevant Signal Addition，目标是在剩余信号中加入水印特征，让训练模型优先学习这些结构化、可追踪但与真实身份无关的特征。

完整数据流是：输入干净语音 $x$，先做 Fourier transform 得到频谱 $x_{\mathrm{spec}}$；通过阈值搜索和 ACO 选择需要削弱或删除的频率，生成中间语音 $\hat{x}$；然后使用 Volto+ 或 Volto* 加入 watermark，输出最终 protected audio $\tilde{x}$。Volto+ 在时间域加短 Gaussian trigger，Volto* 训练一个 generator，把 time-domain watermark 映射为频域全局扰动并嵌入频谱。

### 3.2 问题形式化：让被保护数据训练出的模型表现更差

论文以 speaker classification 为代理问题。一个训练在数据集 $D_p$ 上的模型可写成 $f_{D_p}$。防御者希望构造 protected training dataset $D_p$，使模型在测试集 $D_t$ 上的自然表现变差：

$$\arg\max_{D_p}R_{\mathrm{nat}}(f_{D_p},D_t),\quad f_{D_p}=\arg\min_{\theta}R_{\mathrm{nat}}(f_{\theta},D_p).$$

这里原文把 $R_{\mathrm{nat}}$ 解释为测试准确率 natural risk；从优化目标看，若 $R_{\mathrm{nat}}$ 表示错误风险则取最大合理，若表示 accuracy 则方向应是降低 accuracy。论文叙述目标明确是降低模型性能，因此复现时应以 ACC 下降和 EER 上升作为实际目标。

水印验证的形式化是：被保护数据里嵌入 watermark $\delta$，若某模型训练过 $D_p$，它会对 $\delta$ 表现出不同于正常模型的敏感性：

$$f_{D_p}(\delta)\neq f_{D_c}(\delta).$$

语音质量约束不能直接用图像里常见的 $L_p$ norm，因为人耳对频率变化敏感。论文改用 PESQ 和 SNR 做感知约束，并设定 PESQ > 1.99、SNR > 6 dB 作为最低可用门槛。

### 3.3 Longitudinal cropping：用阈值削弱声纹相关低能频率

纵向裁剪的输入是语音 $x$，输出是经过部分频率能量削弱后的音频。Volto 使用预训练 i-vector model 作为辅助 speaker recognition network $f$。如果移除某个信号后，同一说话人的测试音频与注册音频的 embedding cosine similarity 低于阈值，论文认为该信号含有 speaker-specific information。

具体步骤是：先用 Fourier transform 将 $x$ 分解为频率 $w$ 和幅度 $m$，得到 $x_{\mathrm{spec}}$。然后设定幅度阈值 $t$，过滤或削弱幅度低于阈值的频谱分量，再 inverse transform 得到候选语音。候选语音送入 i-vector SR 模型，与同一人的 enrolled example 比较 similarity。若 cosine similarity 低于 0.6，则认为识别失败，说明被移除成分对说话人识别有效。

阈值 $t$ 通过二分自适应搜索确定。初始设为最大幅度 $m'$ 的 $1/64$。如果处理后模型仍正确识别，说话人相关信息没有削弱够，就提高阈值；如果发生误识别，就降低阈值并回退音频到前一状态。论文描述为：

$$t_0=\frac{m'}{64}.$$

若识别仍成功，则增大阈值：

$$t\leftarrow\frac{t+m'}{2}.$$

若误识别，则降低阈值：

$$t\leftarrow\frac{t}{2}.$$

这个模块解决的是“先大体削弱声纹信息，但不要造成明显音质损伤”。它提供的是粗粒度保护，不足之处是阈值过低保护弱，过高音质差，因此还需要 lateral cropping 精细选择关键频率。

### 3.4 Lateral cropping：用 ACO 搜索频率组合，而不是贪心删 top-n

横向裁剪针对的是频率组合。论文指出，简单逐个删除频率并按 similarity drop 排序不稳定，因为频率之间存在非线性相互依赖：删除 top $n-1$ 个频率可能比删除 top $n$ 个频率更有效。Fig. 2 展示了 direct criticality ranking 不稳定，而 ACO 曲线更平稳。

ACO 的候选池只包含幅度低于阈值 $t$ 的频率：

$$W_{\mathrm{cand}}=\{w_1,w_2,\ldots,w_n\}.$$

每个候选频率 $w_i$ 初始化 pheromone $phero_i=1$。初始权重由单独删除该频率后的 similarity score $s_i$ 决定：

$$wt_i=1-s_i.$$

每只蚂蚁从 top 10 most influential frequencies 中选起点，然后按概率选择后续频率。论文给出选择概率：

$$prob(w_i)=\alpha wt_i+\beta phero_i.$$

其中 $\alpha$ 和 $\beta$ 控制初始重要性与 pheromone 的权重。每只蚂蚁构造一个待删除频率集合 $W_{\mathrm{re}}$，生成修改音频后用 similarity score 和 PESQ 评价。效率分数为：

$$eff=\lambda(1-s)+\phi\frac{p}{4.5}.$$

$s$ 是模型相似度，越低越能破坏识别；$p$ 是 PESQ，越高越好；$\lambda$ 和 $\phi$ 平衡保护效果和音质。每轮只让 elite ants 更新 pheromone。先蒸发：

$$phero\leftarrow phero\times r.$$

再对 elite path 强化：

$$phero_i\leftarrow phero_i+eff,\quad i\in elites.$$

终止条件有三个：修改音频 similarity 低于预设阈值，例如 0.3；蚂蚁超过最大迭代数，例如 200；候选频率组合已探索完。这个模块是 Volto 的关键，因为它让频率删除从“单频重要性排名”变成“组合优化”，更符合语音频谱的相关性。

### 3.5 Speaker-Irrelevant Signal Addition：用水印把模型注意力引向错误特征

第二阶段不是普通版权水印，而是让水印成为 learnable but speaker-irrelevant feature。论文借鉴 backdoor attacks 的经验：DNN 倾向学习简单、显著、一致的特征。如果在每个 speaker 的 protected data 中嵌入 class-specific watermark，非法模型训练时会把这些 watermark 作为分类线索，从而进一步偏离真实 speaker embedding。之后用户可以用对应 watermark query 测试可疑模型是否学习了该触发特征。

Volto 提供两种嵌入方式。

Volto+ 是 time-domain 版本。对第 $i$ 个 speaker，在 waveform 的第 $i$ 个时间区间创建唯一 Gaussian-noise segment $tw_i$，然后叠加到第一阶段输出 $\hat{x}$：

$$\tilde{x}=\hat{x}+tw_i.$$

优点是简单、高效、容易被早期卷积层学习；缺点是局部时间扰动可能更容易被滤波、降噪或量化削弱，也可能带来轻微听感 artifact。

Volto* 是 frequency-domain 版本。先取 $\hat{x}$ 的频谱 $U(\hat{x})$，用 generator $G$ 将 time-domain watermark $tw$ 映射成频域全局表示 $G(tw)$，再 residual addition：

$$spec'=U(\hat{x})\oplus G(tw).$$

最后 inverse Fourier transform 得到输出：

$$\tilde{x}=Z(spec').$$

由于频域水印是全局分布式的，它比短时间段 trigger 更隐蔽，也更抗 restoration-based attack。

### 3.6 Volto* 的 generator-discriminator 训练

Volto* 使用类似 GAN 的 generator-discriminator 架构，但目的不是生成逼真语音，而是把水印稳健地嵌入频谱。$G$ 和 $D$ 都基于 U-Net。Generator 前面接 Fourier transform $U$，把 time-domain watermark 映射到 frequency domain；discriminator 末端接 inverse Fourier transform $Z$，用于恢复 watermark 信息。

Generator 有两个相似性目标。第一个是 time-domain similarity，要求生成语音和原语音接近：

$$\arg\min_G L(Z(U(x)\oplus G(tw)),x).$$

第二个是 frequency-domain similarity，要求修改后的频谱和原频谱接近：

$$\arg\min_G L(U(x)\oplus G(tw),U(x)).$$

Discriminator 的目标是从嵌入后的音频中恢复水印：

$$\arg\min_D L(D(Z(U(x)\oplus G(tw))),tw).$$

原文公式中第三个目标末尾写作 $t$，结合上下文“extracted and original watermark”可判断应对应 watermark $tw$，复现时应以原始水印为监督目标。这个模块的接口是：第一阶段输出的 $\hat{x}$ 和 speaker-specific watermark $tw$ 输入 generator，输出带频域水印的 $\tilde{x}$；discriminator 训练时从 $\tilde{x}$ 中恢复 watermark。

### 3.7 训练、构造与推理分离

Volto 的构造阶段发生在用户发布/贡献数据前。用户或平台对每条音频运行 speaker-specific removal 和 watermark addition，生成 protected audio。攻击者训练模型时只看到 protected audio，不知道干净版本和内部构造细节。验证阶段发生在事后：用户用 clean audio、noise-matched BGM 或 Volto watermark audio 去 probe 可疑模型，比较模型是否对对应 watermark 有异常高的接受率。

需要注意，Volto 不要求训练目标攻击模型，也不需要知道目标模型架构。它使用 i-vector 作为辅助模型来选频，但实验评估用 TDNN、ECAPA-TDNN、ResNetSE、Res2Net、ERes2Net、CAM++ 六个不同 SR 模型，强调黑盒迁移。

### 3.8 复现配置表

| 项目 | 论文配置 |
|---|---|
| 任务 | 语音数据 unlearnable protection + watermark-based ownership verification |
| 期刊 | IEEE Transactions on Dependable and Secure Computing, 2026 |
| DOI | 10.1109/TDSC.2025.3648349 |
| 数据集 | VoxCeleb1、CMU Arctic、Fluent Speech Commands |
| VoxCeleb1 子集 | 50 speakers，5808 examples |
| CMU Arctic 子集 | 9752 examples |
| FSC 子集 | 30 speakers，13419 examples |
| 辅助模型 | 预训练 i-vector speaker recognition model |
| 评估模型 | TDNN、ECAPA-TDNN、ResNetSE、Res2Net、ERes2Net、CAM++ |
| 特征 | Fbank，16 kHz，80 mel bins |
| 训练优化器 | Adam |
| 学习率 | $5\times10^{-4}$ |
| weight decay | $1\times10^{-5}$ |
| scheduler | $5\times10^{-6}$ 到 $5\times10^{-4}$，5 warmup epochs |
| 硬件 | Ubuntu 16.04，单 NVIDIA GeForce RTX 2080 Ti |
| ACO ants | 5 ants |
| elite 策略 | 每轮选择最优 ant 更新 pheromone |
| SR 相似度拒绝阈值 | cosine similarity 0.6 |
| ACO 目标阈值 | similarity 低于 0.3 或最大 200 iterations |
| 质量阈值 | PESQ > 1.99，SNR > 6 dB |
| 压缩防御 | downsampling 到 8000/10000/12000 Hz；MP3 64/128/192 kbps；8/9/10-bit quantization |
| baselines | CUDA、SN、EMN、RUE、TUE、WaveFuzz |
| 代码 | 当前材料和公开检索未发现官方代码 |

### 3.9 最小可复现伪代码

```python
for audio, speaker_id in dataset:
    spectrum = fourier_transform(audio)
    threshold = bisection_search_threshold(
        spectrum=spectrum,
        auxiliary_sr_model=ivector_model,
        enrolled_audio=enroll[speaker_id],
        reject_similarity=0.6,
    )

    candidate_freqs = select_freqs_below_threshold(spectrum, threshold)
    removed_freqs = ant_colony_search(
        candidate_freqs=candidate_freqs,
        objective_similarity=0.3,
        max_iterations=200,
        ants=5,
        quality_metric=PESQ,
    )

    x_hat = remove_frequency_set_and_inverse_transform(audio, removed_freqs)

    if variant == "Volto+":
        trigger = gaussian_segment_at_speaker_interval(speaker_id)
        x_tilde = x_hat + trigger
    else:
        tw = speaker_specific_watermark(speaker_id)
        wm_freq = generator(tw)
        spec_prime = fourier_transform(x_hat) + wm_freq
        x_tilde = inverse_fourier_transform(spec_prime)

    save_protected_audio(x_tilde)

train_suspect_model_on_protected_audio()
probe_suspect_model_with_corresponding_watermark_queries()
compare_verification_rate_against_clean_and_noise_matched_controls()
```

### 3.10 复现风险与信息缺口

主要复现风险有五个。第一，Table II/III/VI/VII/VIII 的完整数值在 PDF 文本层中提取不完整，复现核对需要直接看版面或源码。第二，Volto* 的 generator/discriminator 架构只说是 U-Net，缺少层数、通道数、kernel、训练 epoch、batch size 和 loss 具体实现。第三，ACO 中 $\alpha,\beta,\lambda,\phi,r$ 的具体取值在当前可读正文中未完整给出。第四，time-domain watermark 的长度、幅度、speaker-to-interval 映射细节需要代码或附录补充。第五，水印验证方式依赖可疑模型的接口和输出类型；如果模型只返回 embedding、不返回分类结果，或者攻击者做后门清洗，验证流程需要重新设计。

## 4. 与其他方法对比

Volto 的本质差异是双功能协同。传统 unlearnable examples 只关心让模型学不好，但无法事后证明模型是否使用了数据；传统 audio watermarking 只关心证明内容归属，但不能阻止模型训练；Volto 把两者连接起来，先削弱真实声纹特征，再注入可学习但说话人无关的水印特征，使非法训练模型既性能下降，又留下可验证行为痕迹。

| 方法类别 | 代表方法 | 核心思想 | 优点 | 缺点 | Volto 的改进 |
|---|---|---|---|---|---|
| 数据匿名化 | voice anonymization | 修改或隐藏说话人身份 | 保护直接身份信息 | 可能损害语音可用性，不能证明训练滥用 | Volto 保留可听质量并提供水印验证 |
| 访问控制 | policy / permission | 限制数据访问 | 部署简单 | 数据泄露后无效 | Volto 保护数据本身 |
| 图像 unlearnable examples | EMN、RUE、TUE | 加扰使训练失败 | 对图像任务有效 | 迁移到语音会损害音质或黑盒不稳 | Volto 使用语音频谱和听觉约束 |
| 音频 unlearnable | WaveFuzz 等 | 针对音频特征扰动 | 开始关注语音数据保护 | 常依赖 MFCC 或目标模型假设 | Volto 面向现代 SR 模型黑盒泛化 |
| 音频水印 | spread spectrum、QIM、DNN watermark | 嵌入可检测版权信号 | 可做所有权证明 | 不阻止模型学习 | Volto 让水印也参与误导训练 |
| Volto | 本文 | 频率裁剪 + speaker-irrelevant watermark | 隐私防护和版权验证同时实现 | 依赖复杂构造和验证设定 | 本文方法 |

Volto 更适合个人语音数据发布、平台语音数据贡献、训练集版权保护和 speaker recognition 数据防护。不适合纯实时通信加密，也不等价于防止所有 voice cloning，因为论文主要通过 SR/speaker encoder 代理任务评估。

## 5. 实验表现与优势

### 5.1 实验设计

论文使用三类语音数据集：VoxCeleb1、CMU Arctic、Fluent Speech Commands。评估模型包括 TDNN、ECAPA-TDNN、ResNetSE、Res2Net、ERes2Net 和 CAM++，并且这些模型与第一阶段使用的 i-vector 辅助模型完全不同，用于验证黑盒泛化。

指标分三类。训练破坏效果用 ACC 下降和 EER 上升衡量；感知质量用 PESQ、SNR、WV-MOS、UT-MOS 衡量；水印验证用可疑模型对 clean audio、noise-matched BGM 和 Volto watermark audio 的验证/接受率差异衡量。

### 5.2 unlearnable 保护效果

Table II 汇总三数据集、六模型的训练结果。论文明确给出的总体结论是：Volto 最大能使模型 ACC 降低 21.44%，平均 EER 增加 13.17%。在 Res2Net + CMU 这类 baseline 方法几乎无效甚至提升模型性能的场景，Volto 仍能使 ACC 降低 8.31%。这说明 Volto 的频谱删除和水印诱导不只依赖某个特定模型结构。

论文也强调，部分 baseline 在某些数据集上看似造成更强 ACC 下降，但通常伴随不可接受的语音质量。例如 TUE 在 VoxCeleb 上可造成 8.86%-21.65% accuracy reduction 和 9.03%-17.35% EER increase，但在 CMU 上不稳定，甚至可能提升模型表现。CUDA 在 FSC 上较强，但在 VoxCeleb 上弱，且音质指标不满足要求。

### 5.3 水印验证效果

Table III 评估模型是否学习了 Volto watermark。实验训练三类模型：clean audio、Volto+ protected audio、Volto* protected audio。测试查询有三种：clean audio、SNR 匹配的 MUSAN BGM audio、Volto watermark audio。BGM 作为噪声控制，用来区分“模型只是对噪声敏感”还是“模型真的学习了结构化 watermark”。

论文给出的例子是 TDNN on Vox：clean query 约 3%，BGM query 约 9%，Volto+ watermark query 升到 26.76%。总体上，Volto 相比 normal models 的平均 watermark verification 高于 14.06%。这说明被 Volto 数据训练的模型对对应水印触发更敏感，可用于事后所有权验证。

### 5.4 生成效率

Table IV 比较平均生成时间。EMN、RUE、TUE 等需要模型训练或迭代优化的方法 iteration time 经常超过 300 秒，不适合社交媒体或高吞吐发布。Volto 的 generator 可预训练，在线构造时只需 feature removal 和 watermark embedding，不需要迭代训练，约 10 秒生成一个 protected example。它比纯静态噪声方法慢，但比迭代式 unlearnable 方法实用。

### 5.5 消融实验

消融只保留第一阶段 Speaker-Specific Signal Removal，去掉第二阶段 Speaker-Irrelevant Signal Addition。相比完整 Volto+ 和 Volto*，单阶段版本 EER 分别下降 4.03% 和 3.75%，ACC 分别上升 9.96% 和 4.95%。这证明第二阶段不只是额外提供水印，还会进一步增强 unlearnability：模型被引导去学习结构化 watermark，而不是残余 speaker identity features。

### 5.6 分布与感知质量

Fig. 4 使用 MFCC + PCA 可视化 clean 和 protected data。SN、EMN、RUE、TUE 等方法和 clean distribution 差异明显，甚至容易形成异常聚类；Volto 和 CUDA 更接近 clean distribution，但 Volto 又比 CUDA 更能减少正常语音特征。Fig. 5 的 waveform residual 显示 Volto+ 和 Volto* 保留原始时域结构且 residual 较低。Fig. 6 的 spectral residual 显示 baseline 常在 1500-2500 Hz 附近引入明显成分，而人耳对该频段较敏感；Volto 的频谱残差更平滑、更低。

Table V 使用 WV-MOS 和 UT-MOS 预测主观质量。论文结论是 Volto+ 和 Volto* 在三数据集上都保持接近 clean 的感知质量，并取得最高 MOS；而 EMN、TUE 等 baseline 在部分情况下 WV-MOS 甚至为负，表示感知上非常不自然。

### 5.7 对预训练 SR 模型与压缩防御的鲁棒性

Table VI 测试直接把 unlearnable examples 输入 clean VoxCeleb1 上训练好的 ResNetSE、ERes2Net、CAM++。在 CMU 上，很多方法仍有 60%-70% 以上平均识别 accuracy；Volto 则能让平均 accuracy 低于 20%，同时 PESQ/SNR 满足质量要求。论文摘要中还给出自动工具/预训练模型分析场景下，平均 accuracy 从 95.29% 降至 10.25%。

Table VII 测试 downsampling、MP3 和 quantization。8000 Hz 下采样反而可能增强 Volto，因为更多信息被丢失。10000/12000 Hz、MP3 64/128/192 kbps 对 Volto 影响有限。量化对 Volto+ 稍有影响，因为短时 trigger 可能被部分去除；Volto* 的频域全局水印更稳。10-bit quantization 下，Volto 仍能使 ACC 降低 8.14%，EER 增加 5.48%。

### 5.8 不同保护比例

Table VIII 评估只保护部分训练数据的情况。总体趋势是保护比例越高，模型性能越差。论文举例：保护率 0.2 时，Volto 使 ACC 下降 1.44%、EER 增加 0.1%；保护率 0.4 时，ACC 下降 4.64%、EER 增加 0.93%。有趣的是 0.1 比 0.2 下降更明显，论文推测小量 protected examples 可能使模型过拟合这些异常样本，引起决策边界突变。

### 5.9 实验证据边界

实验覆盖了三数据集、六个 SR 模型、压缩/下采样/量化、防护比例、感知质量和水印验证，证据较系统。边界在于：完整数值表依赖版面读取；方法主要围绕 SR/speaker classification，不能直接等同于所有语音生成模型、voice cloning 或 SSL pretraining 场景；没有评估强白盒去水印、数据增强训练、防后门清洗、混合大规模未保护数据时的极端稀释效应。

## 6. 学习与应用

论文当前材料和公开检索未发现官方开源代码。若要复现，应先实现 Volto 的最小版本：i-vector 辅助模型 + Fourier threshold cropping + ACO lateral cropping + Volto+ time-domain watermark，再扩展到 Volto* frequency-domain generator。

实现建议：

- 先验证第一阶段能让 i-vector similarity 降到 0.6 以下，同时 PESQ > 1.99、SNR > 6 dB。
- ACO 不要只按单频重要性贪心删除，频率之间存在组合效应。
- Volto+ 适合作为快速 baseline，但要重点测试量化和滤波是否破坏短时 trigger。
- Volto* 需要完整训练 generator/discriminator，必须补齐 U-Net 架构和 loss 细节。
- 水印验证必须设计 clean query 和 noise-matched BGM query，否则无法排除“普通噪声也能触发”的解释。

对语音安全研究的启发是：水印可以不只作为检测信号，也可以作为训练数据保护机制的一部分。对于你的语音水印方向，这篇可以作为“watermark + data poisoning/unlearnable”交叉方向，放在 post-hoc watermarking 里并在 Overview 中标注为 dual voice data protection。

## 7. 总结

一句话概括：删声纹，再嵌水印。

## 8. 图表精读与证据链

- Table I 对比现有 voice protection methods，强调它们要么假设不现实，要么只关注模型性能破坏，要么没有版权验证。
- Fig. 1 是 Volto 框架图：第一阶段 frequency-domain cropping 移除 speaker-related signals；第二阶段添加 speaker-independent watermarks，可在时间域或频域嵌入。
- Algorithm 1 是 speaker-specific signal removal，包含 longitudinal cropping、阈值二分和 lateral cropping/ACO 频率选择，是复现第一阶段的核心。
- Fig. 2 比较 ACO 和直接按 criticality 删除频率，说明频率组合有非线性，ACO 更稳定。
- Algorithm 2 是 speaker-irrelevant signal addition，给出 Volto+ 和 Volto* 两条嵌入路径。
- Fig. 3 展示 Volto* frequency-domain watermark generator 的训练流程，支撑“水印频域全局嵌入更隐蔽”的设计。
- Table II 是主保护效果表，支撑 Volto 在三数据集六模型上的 ACC/EER 退化效果。
- Table III 是水印验证表，支撑 Volto 可提供事后 IP verification，而不只是让模型训练变差。
- Table IV 是效率表，说明 Volto 在线生成约 10 秒/样本，比迭代式 unlearnable methods 更实用。
- Fig. 4、Fig. 5、Fig. 6 分别从特征分布、波形 residual、频谱 residual 解释 Volto 为什么兼顾保护和音质。
- Table V 用 WV-MOS/UT-MOS 支撑感知质量。
- Table VI 验证 Volto 对 clean-data 预训练 SR 模型的直接欺骗能力。
- Table VII 验证压缩、下采样、量化后仍有保护效果，尤其 Volto* 对量化更稳。
- Table VIII 验证部分数据保护比例下的效果，贴近用户只能控制部分数据的现实。

证据链整体是：未经授权训练需要主动数据保护 -> 单独 unlearnable 或 watermark 都不够 -> 频谱裁剪削弱真实声纹 -> 水印诱导模型学习 speaker-irrelevant trigger -> ACC/EER、watermark verification、MOS/SNR/PESQ、compression robustness 共同验证。

## 9. 复现难度与适合人群

复现难度：高。

高难度来自三个方面：频谱操作和 ACO 需要精细实现；Volto* 的 generator/discriminator 细节不足；完整评估需要三数据集、六个 SR 模型、多个 baseline、感知质量指标和水印验证协议。若只复现 Volto+，难度可降为中等。

主要依赖包括 VoxCeleb1、CMU Arctic、Fluent Speech Commands、i-vector model、TDNN/ECAPA-TDNN/ResNetSE/Res2Net/ERes2Net/CAM++、Fbank 特征、PESQ、SNR、WV-MOS、UT-MOS、MUSAN BGM、MP3/downsampling/quantization 处理工具，以及 GPU 训练环境。

最小可复现版本：

1. 选 VoxCeleb1 小子集和一个 TDNN 模型。
2. 用 i-vector 辅助模型实现 longitudinal cropping。
3. 加入简化 ACO 选择关键频率。
4. 实现 Volto+ time-domain watermark。
5. 训练 TDNN 比较 clean vs protected 的 ACC/EER。
6. 设计 watermark query 和 BGM query 做验证。

适合人群：语音安全研究者、unlearnable examples 研究者、音频水印研究者、语音数据版权保护方向工程/研究人员。初学者可以先读它的方法动机和 threat model，不建议直接完整复现。

## 10. 简短全面总结

Volto 面向用户语音被未经授权收集并用于训练 SR 或语音模型的问题，提出“不可学习 + 可追踪”的双功能保护框架。论文指出，现有 unlearnable audio 往往依赖目标模型或牺牲音质，传统 audio watermarking 又只能做内容归属，不能阻止模型学习。Volto 第一阶段通过 Fourier 频谱处理移除 speaker-specific signals：先用纵向裁剪削弱频率能量，再用 ACO 横向选择关键频率组合，在保持 PESQ/SNR 可接受的同时降低 speaker similarity。第二阶段加入 speaker-irrelevant watermark，Volto+ 在时间域叠加 speaker-specific Gaussian segment，Volto* 用 U-Net generator 将 watermark 嵌入频域全局表示，并用 discriminator 学习提取。实验覆盖 VoxCeleb1、CMU Arctic、FSC 和六种 SR 模型，显示 Volto 最高降低 ACC 21.44%、平均提高 EER 13.17%，并提供高于正常模型 14.06% 的水印验证信号。主要局限是代码未公开、Volto* 架构细节不足，且强白盒去水印和非 SR 语音生成任务验证仍有限。

## 11. 论文写作逻辑分析

### 11.1 Intro 的问题铺垫

论文先从语音数据作为 AI 资源切入，再把风险分为隐私和 IP 两条线。随后分别讨论 unlearnable examples 和 watermarking 的局限：前者阻止训练但不能验证侵权，后者能验证但不能阻止训练。这个双缺口自然引出“dual-function framework”。

### 11.2 Insight 与 motivation

核心 insight 有两个：声纹可看作特定频率组合，因此可通过频谱裁剪破坏；DNN 容易学习显著简单特征，因此可用 speaker-irrelevant watermark 把学习注意力引走。这两个 insight 分别支撑第一阶段和第二阶段，且能解释为什么二者协同而不是简单拼接。

### 11.3 Threat model 的承接作用

Threat model 设定用户不知道目标模型架构、参数和训练集组成，因此方法必须 model-agnostic。后续用 i-vector 做辅助、六个不同 SR 模型做评估，正是为了证明黑盒迁移。用户只能控制自己数据，因此 Table VIII 的不同保护比例实验也和 threat model 对应。

### 11.4 方法叙事

方法叙事按“移除真实特征 -> 加入可追踪替代特征”展开，顺序合理。longitudinal cropping 解决粗粒度声纹削弱，lateral cropping 解决关键频率组合，watermark addition 解决追踪和学习诱导。Volto+ / Volto* 的并列设计也清楚展示了效率和隐蔽性的取舍。

### 11.5 实验呼应

实验覆盖主张的多个方面：Table II 测训练破坏，Table III 测水印验证，Table IV 测效率，Fig. 4-6 和 Table V 测质量，Table VII 测数据清洗鲁棒性，Table VIII 测部分保护率。整体组织能回应 introduction 的四个原则：合理假设、性能退化、感知质量、版权识别。

### 11.6 证据链完整性

强证据是三数据集六模型和消融实验，证明第二阶段水印确实增强 unlearnability。较弱处是对更强攻击者的讨论不足，例如攻击者若知道水印机制并做 trigger sanitization、adversarial training 或大规模混合未保护数据，验证强度如何变化尚不清楚。另一个弱点是许多表格数值在 PDF 中较密集，方法细节和完整超参数不够复现友好。

### 11.7 可借鉴写法

这篇的写作可借鉴点是把“隐私保护”和“版权追踪”合并成一个统一问题，而不是把水印作为附加模块。它的 argument chain 很适合安全论文：现实风险 -> 单一技术不足 -> 双机制协同 -> 每个模块对应一个失败模式 -> 实验分别验证保护、追踪、质量、效率和鲁棒性。

## Overview 条目

```md
- [When Unlearnable Examples Cooperate With Watermarking: A Dual Voice Data Protection Against Unauthorized Exploitation](./Watermarking/post_hoc_watermarking/2026-Volto.md)  
  *IEEE Transactions on Dependable and Secure Computing, 2026*  
  Citation: Ge, Y., Gu, R., Liu, Y., Zhao, L., Du, B., & Wang, Q. “When Unlearnable Examples Cooperate With Watermarking: A Dual Voice Data Protection Against Unauthorized Exploitation.” *IEEE Transactions on Dependable and Secure Computing*, vol. 23, no. 3, pp. 4652–4669, 2026.  
  Links: [DOI](https://doi.org/10.1109/TDSC.2025.3648349) | Code: Not found
```
