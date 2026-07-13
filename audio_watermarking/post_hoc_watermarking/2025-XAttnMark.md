# XAttnMark：用交叉注意力统一鲁棒检测与身份归因

## 0. 翻译摘要原文

生成式音频合成与编辑快速普及，带来版权侵权、来源不明和深伪误导风险。水印可嵌入不可感知、可识别且可追踪的信号，但 WavMark、AudioSeal 等神经方案难以同时优化鲁棒检测和准确归因。XAttnMark 通过生成器与检测器的部分参数共享、用于高效消息恢复的交叉注意力、改善消息时间分布的条件模块，弥合两项任务的差距；同时提出符合心理声学时频掩蔽的损失，精细利用听觉掩蔽提升透明度。实验在常规处理和不同强度生成式编辑下取得较强检测与归因表现。

## 1. 方法动机

水印有两种不同任务：检测只需判断“有/无”，归因还要从用户池中恢复准确消息。AudioSeal 的生成器与检测器完全分离，检测很快收敛，但消息恢复学习慢且无攻击时也不理想；WavMark 完全共享可快速互逆，却在多种失真下出现检测鲁棒性退化。

论文提出中间解：只共享消息 embedding table，让生成端知道如何组合 bit，检测端又以同一表作为检索参照；消息不再均值池化后在时间上重复，而以 temporal modulation 分散到不同时间位置。音质方面，固定时频块 loudness 差不足以描述前/后掩蔽及向高频扩散的不对称性，因此构建逐 tile 心理声学权重。

## 2. 威胁模型解读

- 内容发布者给音频写入与用户 ID 对应的 16 bit 消息；验证端先检测，再在 $N=100$、1,000、10,000 等候选池中以最小 Hamming distance 归因。
- 信道包括 16 种常规编辑：滤波、增益、回声、噪声、平滑、变速、重采样、AAC/MP3、EnCodec、裁剪；附录还评估生成式编辑和 HopSkipJump 黑盒查询攻击。
- 检测阈值在平衡验证集上用 Youden’s Index 选择，因此表中 TPR/FPR 不是预先固定低 FPR 运营点；跨分布部署需重新校准。
- HSJA 假设可黑盒查询检测结果/分数并保持音质，覆盖一定自适应攻击；没有密钥泄露后的消息伪造、完整白盒梯度攻击或物理重录。
- attribution accuracy 定义为“已成功检测样本中正确归因比例”，与 AudioSeal 把全部样本作为分母的定义不同；必须同时看 Det.+Att. 或 FAR，避免指标口径优势。
- 速度变化下检测仍强但归因几乎失败，说明“存在性水印”与“可追踪身份”威胁目标不能互换。

## 3. 方法设计与复现级解读

### 3.1 全局流程

XAttnMark 继承 AudioSeal 波形生成器/检测器主干。生成端把 K-bit 消息映射到共享 embedding table 中每个 bit-position 的 0/1 向量，经时间调制形成消息 latent，与波形 latent 融合并输出加性水印。水印音频随机经 16 类编辑后进入检测器：一支输出逐帧水印存在概率，另一支将重建音频 latent 作为 query、共享 embedding table 作为 key/value，用 cross-attention 恢复各 bit。训练同时优化检测、消息、心理声学/谱/波形和对抗损失，并根据验证集薄弱攻击动态更新采样概率。

### 3.2 消息表与部分共享

对每个 bit 位置 $j$ 分别保存 0/1 两个 embedding，消息 $w_j$ 映射到索引 $I_j=2j+w_j$，因此只需 $2K$ 向量而非 $2^K$ 完整消息表。生成器取 $V(w)=[E_{I_1},\ldots,E_{I_K}]$。共享的不是全部声学参数，而是这个消息字典 $E$；它给检测端提供与嵌入端一致的组合基元，同时保留两端独立声学建模的鲁棒性。

### 3.3 交叉注意力消息恢复

将共享表相邻 0/1 项拼成 $E'\in\mathbb R^{K\times2H}$，线性投影为 key/value。检测器重建的时序 latent $\tilde h_x\in\mathbb R^{T\times H}$ 先通过时间解调 $W_{dem}$ 变成 K 个 position query，再计算

$$A=\operatorname{softmax}(QK^\top/\sqrt H),\qquad \tilde V_x=\operatorname{ELU}(AV).$$

最后每个位置经共享/独立线性头和 sigmoid 输出 bit。这里 attention 的作用不是对齐文字与音频，而是在已知每个 bit 的两个候选表示中，用音频证据检索最匹配的消息成分。

### 3.4 时间调制消息嵌入

AudioSeal 把 K 个消息向量平均成一个 $H$ 维向量再复制到所有时刻，形成信息瓶颈。XAttnMark 用 $W_M\in\mathbb R^{K\times t'}$ 对 K 个位置做线性时间调制：

$$h_w=W_M^\top V(w)\in\mathbb R^{t'\times H}.$$

不同 bit 在时间轴产生不同组合，使检测器可从声学序列中分离它们；代价是对时间伸缩更敏感，正好对应 speed 下归因失败。

### 3.5 心理声学时频掩蔽损失

原音 Mel tile 若超过该频带最大能量的 $\alpha_S=0.8$，被视为 masker。每个 masker 的影响范围按临界带宽随频率变化，时间上采用 20 ms backward、200 ms forward，频率和时间衰减方向不对称，以模拟高频上扩散和前掩蔽更强。每个 maskee 只取最强 masker 的阈值 $E^*_{mask}$。

若水印 tile 能量低于该阈值，就允许更大谱差；损失写成

$$L_{TF}=\sum_{m,t}\frac{\|S_W(m,t)-S_O(m,t)\|_2^2}{\lambda(m,t)},$$

其中被掩蔽 tile 的 $\lambda>1$。因此模型不是简单减少所有扰动，而是把更多水印能量迁移到感知不敏感区域。辅助项还包括波形 $L_1$、多尺度 Mel、feature matching 与多尺度复 STFT 判别器；对抗权重仅 1，低于 AudioSeal 的 4。

### 3.6 训练与推理

训练使用 1 秒、16 kHz 单声道；每步采样消息、生成水印、随机编辑、同时计算检测 BCE、消息 BCE 和感知/对抗损失。每 1000 steps 根据验证准确率调节 16 种编辑采样概率，薄弱攻击更常出现。推理对任意长度音频补齐并切成一秒块，逐块水印后拼接；检测/归因再聚合局部输出。

| 配置 | 论文设置 |
|---|---|
| 训练数据 | 约 4,100 h：VoxPopuli 3,016.43、LibriSpeech 100.59、FMA 879.29、MusicCaps 9、AudioSet 98.53 h |
| 训练输入 | 16 kHz、1 秒、mono；latent $H=32$ |
| 测试 | MusicCaps held-out 100 条，每条 100 消息、16 编辑，共 160k 实例；默认 5 秒 |
| 消息 | 16 bit，最多 65,536 IDs；池规模 100/1k/10k |
| 优化 | Adam lr $10^{-5}$，$\beta_1=0.4,\beta_2=0.9$；batch 16；73k steps；EMA 0.99 |
| 权重 | $\lambda_{TF}=1,\lambda_{adv}=1,\lambda_{L1}=0.1,\lambda_{msspec}=2,\lambda_{detect}=\lambda_{message}=10$ |
| 基线 | AudiowMark、WavMark、TimbreWM、AudioSeal |
| 主指标 | detection Acc/TPR/FPR/AUC；attribution Acc/FAR/Det.+Att.；PESQ、SI-SNR、ViSQOL、MUSHRA |

## 4. 与其他方法对比

| 架构 | 参数关系 | 优点 | 缺点 | XAttnMark 改进 |
|---|---|---|---|---|
| WavMark | 全共享 | 消息学习快 | 长训任务干扰、鲁棒性弱 | 只共享消息字典 |
| AudioSeal | 全分离 | 检测/定位强 | 消息恢复慢且弱 | cross-attention 查共享表 |
| TimbreWM | 音色式消息 | 部分编辑稳 | 检测/归因不均衡 | 动态攻击训练+时序消息 |
| XAttnMark | 部分共享 | 检测和归因兼顾 | speed 归因脆弱 | 本文方案 |

## 5. 实验表现与优势

### 5.1 16 类编辑下检测与归因

XAttnMark 平均 detection accuracy/TPR/FPR 为 0.9919/0.9856/0.0019，平均 attribution accuracy 0.93；AudioSeal detection 为 0.971，但 attribution 仅 0.39。MP3/AAC 下 XAttnMark attribution 为 1.00/0.88，EnCodec 为 0.99，crop 为 1.00，明显验证消息恢复设计。

速度变化是关键反例：detection 仍为 0.995，但 attribution 只有 0.03，附录跨用户池平均约 4%。这说明时间调制消息仍在但时序排列被破坏；论文提出黑盒 grid search 逆转速度作为后处理，尚非主模型内生能力。

### 5.2 用户池扩展与指标口径

MusicCaps 上 N=10/100/1k/10k 的 attribution 为 93.60/93.87/93.56/93.44%，FAR 约 6.1%–6.6%；Det.+Att. 为 87%–92%。VoxPopuli 上 N=10 时 AudioSeal attribution 96.29% 高于本文 94.97%，但其他池规模 XAttnMark 更稳定。把多种口径并列是必要的，因为“只在已检测样本中归因”会排除检测失败。

### 5.3 生成式编辑与查询攻击

论文评估不同强度生成式编辑，基线随编辑强度接近随机，XAttnMark 检测仍保持约 0.91–0.94 的代表性准确率。HSJA 黑盒攻击中，传统方法更易跌至随机，而本文在相近质量下保持更高检测；但这类结果样本量与查询预算需结合表 2/3 使用，不能外推为白盒安全。

### 5.4 音质与消融

MUSHRA 中原音约 95、XAttnMark 约 91，接近 AudioSeal，低于 AudiowMark 约 94、高于 TimbreWM 约 80。架构训练曲线显示本文约 2k steps 达 99% detection、约 19k steps 检测和消息均接近满分；AudioSeal 到 50k 时消息约 70%，WavMark 后期检测下降。去掉时间调制或 cross-attention 会显著降低消息学习，TF masking 与动态临界带宽提高音质—归因曲线。这里部分消融以曲线/附录呈现，精确表值应以原表为准。

## 6. 学习与应用

复现时必须同时实现 detection、bit decoding、candidate retrieval 三层评测，固定阈值选择与分母口径。工程上可把消息表与用户数据库对接，并为 speed 引入同步不变编码、纠错或显式时间对齐。心理声学损失可独立迁移到 AudioSeal、codec 水印或音乐水印；共享 embedding table 的思路也适用于其他“生成—检索”双端模型。

## 7. 总结

共享消息字典，用交叉注意力兼顾检测归因。

## 8. 图表精读与证据链

图 1 展示音质—归因 trade-off；图 2 是共享表、cross-attention 和 TF loss 主链；表 1 直接比较 16 类编辑；表 4/图 7 评价客观与主观音质；图 8 解释全共享、全分离、部分共享的训练差异；表 5 揭示 attribution 指标口径与跨域结果；附录表 6/7 暴露 speed 失败。证据覆盖较全，物理和白盒安全仍缺失。

## 9. 复现难度与适合人群

难度高，需 4,100 小时混合音频、AudioSeal 主干、多尺度判别器、16 种编辑与心理声学计算。最小版本可用 LibriSpeech+FMA 子集，先复现 shared table、cross-attention 和 identity/MP3/speed 三条件。适合研究生成音频溯源、身份归因、心理声学损失和多任务架构者。

## 10. 简短全面总结

XAttnMark 针对神经水印“能检测却难准确归因”的问题，在 AudioSeal 主干上只共享生成器与检测器的消息 embedding table：生成端用时间调制把各 bit 分散到声学序列，检测端以音频 latent 为 query、共享表为 key/value，通过交叉注意力恢复消息。同时，逐 Mel tile 的心理声学损失模拟临界带宽、前后掩蔽和频率不对称衰减，把扰动引导到不可感知区域。4,100 小时混合训练和 16 类动态采样攻击下，平均检测约 99.2%、归因约 93%，显著优于 AudioSeal 的消息恢复，并保持约 91 MUSHRA。关键局限是变速后检测仍强但归因仅约 3%–4%，物理与完整白盒安全也未覆盖。

## 11. 论文写作逻辑分析

论文先把水印效用拆成 detection 与 attribution，再用 AudioSeal/WavMark 的相反训练缺陷推出部分共享，问题—机制对应清楚。TF loss 从固定 loudness tile 的不足逐步引入心理声学非对称性，公式虽多但动机连贯。实验不仅给平均值，也公开 speed 这一强失败并讨论指标分母，可信度较高。最值得借鉴的是把架构学习动力学作为消融；可改进处是主文标题中的“robust attribution”容易掩盖 speed 与生成式编辑下的显著边界。
