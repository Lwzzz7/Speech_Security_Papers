# Latent-Mark: An Audio Watermark Robust to Neural Codec Compression

> 精读依据：arXiv:2603.05310v3（2026-06-25）全文；会议信息依据 ISCA Archive；官方仓库用于核验实现入口。核验日期：2026-10-05。v1 的 Neural Resynthesis 标题与本文属于同一预印本，不应重复入库。

## 0. 摘要翻译

已有音频水印在传统数字信号处理攻击下具有较强鲁棒性，但仍容易被神经压缩破坏，因为现代神经音频 codec 会像噪声过滤器一样丢弃不可感知的波形变化。本文提出 Latent-Mark，一种为抵抗神经 codec 压缩设计的零比特音频水印。核心思路是在 codec 的不变潜空间中嵌入标记：优化音频波形，使编码后的潜表示发生可检测的方向性偏移，同时约束扰动以保持不可感知性。为避免过拟合单一 codec 的量化规则，作者提出跨 codec 联合优化，在多个代理 codec 上寻找共享的潜空间稳定特征。实验展示了对未见 codec 的迁移能力，并保持一定传统 DSP 鲁棒性与感知质量。摘要中的“共享不变量”应理解为设计动机，正文没有给出所有 codec 共享严格不变空间的数学证明。

## 1. 研究动机

神经 codec 不是 MP3 那样只做传统变换域压缩，它通过编码、量化和神经解码重建音频。若水印主要藏在原波形细微残差中，重建可能保留语音内容却删掉残差。作者将目标从“波形里留下可检测扰动”改成“让 codec 认为标记是潜表示的一部分”。

与库中 LAW 的生成初始噪声重排不同，Latent-Mark 接受任意现成音频，通过逐音频优化波形完成后处理嵌入。它是 zero-bit 来源存在性判断，未实现任意多比特消息恢复。也不同于 2025-Latent-Watermarking：相同 latent 关键词不能作为重复依据。

## 2. 威胁模型

| 项目 | 论文设定 |
|---|---|
| 防御者 | 持有代理 codec、嵌入方向与校准统计；可优化发布波形，并在检测时运行 codec 编码器 |
| 攻击者 | 对水印音频做神经 codec 编码—解码、传统滤波/重采样/缩放/加噪 |
| 攻击目标 | 内容保持可用但水印存在性判决消失 |
| 控制能力 | 控制后处理 codec/参数及 DSP 操作；主要实验没有优化攻击者的目标损失 |
| 知识与密钥 | 方法使用 secret axis；主体评测是未知/未见 codec 变换，未验证知晓方向的白盒自适应移除 |
| 防御约束 | 波形扰动幅度限制及质量评价；无需训练原生成器 |
| 系统边界 | 现成波形 → post-hoc 优化 → 发布 → 外部压缩 → 潜表示检测 |

现实性在于 codec 重建是合理的水印清洗通道；但“未见 codec”不等于具有明确查询预算的恶意黑盒攻击。方向由公开码本聚类得到时，保密机制本身仍需研究。本文没有证明方向具备密码学密钥熵、密钥轮换或抗伪造安全性。

## 3. 方法设计与复现级解读

### 3.1 全局流程与数据形状

离线：加载冻结 codec → 从量化码本导出方向 → 用干净音频校准分数均值/方差与阈值。嵌入：读波形 → 统一工作采样率/长度 → 各 codec 视图编码量化 → 沿方向计算投影 → 用梯度优化同一个波形扰动 → 截断预算 → 保存音频。检测：读待测波形 → 编码量化 → 计算标准化投影 → 单视图或多视图聚合 → 输出存在性分数。

对一个 codec，原文记 $R(s)=D(Q(E(s)))$；$s\in\mathbb{R}^{T}$，量化潜表示 $z_c=Q(E(s))\in\mathbb{R}^{d\times L}$。$d$ 为通道维，$L$ 为时间步，具体取决于 codec。检测器比较潜表示而非回归原波形。

### 3.2 单 codec 方向与嵌入（原文 §3.2）

把码本向量分成两个簇，获得质心 $\mu_A,\mu_B$，归一化质心连线：

$$v_c=(\mu_B-\mu_A)/\|\mu_B-\mu_A\|_2$$

该方向表示码本中两个区域之间的结构变化。作者认为沿此方向移动比把标记塞入会被量化丢弃的残差更稳定。码本有多层 RVQ 时使用哪一组权重、簇标签如何确定正负方向、随机种子和多密钥构造，正文没有全部展开。

对每个时间步做内积，再取平均：

$$\bar p_c(s)=L^{-1}\sum_{t=1}^{L}\langle z_{c,t}(s),v_c\rangle$$

嵌入不是直接修改接收者不可见的潜变量；它优化波形，使编码后的投影达到目标：

$$L_c(\delta)=\max(0,\gamma_c-\bar p_c(s+\delta))$$

单视图实验取 $\gamma_c=1.5$，Adam 优化 150 步，每步对波形扰动硬截断。预算按音频 RMS 与目标 SDR 调节：

$$\epsilon=\beta\,\mathrm{RMS}(s)\,10^{-SDR/20}$$

这里是 $\ell_\infty$ 幅度限制，由 RMS/SDR 公式生成预算，并不自动保证真实输出达到给定 SDR。codec 参数冻结，梯度经编码和量化路径流向波形；离散量化的梯度代理应以实际代码核对，正文不足以直接确定所有 codec 的 STE 处理。

### 3.3 校准与检测

使用未标记音频估计 $\mu_c,\sigma_c$，阈值设为 $\tau_c=\mu_c+k\sigma_c$；归一化判决：

$$m_c(s)=(\bar p_c(s)-\tau_c)/\sigma_c$$

$m_c>0$ 表示有水印。校准集应与测试集隔离；标准差接近零时的数值保护、各域校准方式、真实 FPR 是复现要点。$k=1.5$ 不代表实际测试误报被严格控制在某个百分比，仍需读实验 FPR。

### 3.4 跨 codec 联合优化（原文 §3.3）

不同 codec 的投影尺度不同，不能直接加未归一化损失。作者用干净样本上的平均正阈值缺口计算每个视图的归一化尺度：

$$\alpha_c=\mathbb{E}_{s\sim S_0}[\max(0,\tau_c-\bar p_c(s))]$$

$$L_{joint}(\delta)=|C|^{-1}\sum_{c\in C}\max(0,\tau_c-\bar p_c(s+\delta))/\alpha_c$$

每轮把同一个 $s+\delta$ 转换到各 codec 所需采样率和长度，求各视图损失平均，再更新一个共享扰动。工作波形为 44.1kHz，长度补齐为 4096 的倍数；各视图再重采样。联合设置 Adam 150 步，$\beta=2.5$，预算限制为 $[10^{-4},0.1]$。单 codec 目标 $\gamma_c$ 与联合阈值 $\tau_c$ 不应混用。

检测时各视图输出 $m_c$，排序后取下中位数，避免一个视图异常高分主导结果。多视图稳健聚合服务 codec 不确定性，但运行多个模型有额外开销。若攻击者知道聚合机制并联合优化所有视图，本文没有实证其安全性。

### 3.5 训练与推理分离、指标接口

无数据训练的水印网络；每条音频的优化是嵌入构造成本。码本方向和校准是离线工作。检测需要冻结编码器，不能把“无需重新训练 TTS”理解为“完全不依赖模型”。

原文还使用压缩前后相对分数差，例如比较水印与干净音频经过相同 codec 后的分数。差值为正只说明标记提高了分数，是否跨检测阈值仍需另看 Survivability。不要把相对提升当作检测成功。

### 3.6 复现配置与缺口

| 项目 | 配置 | 来源 |
|---|---|---|
| 方法类型 | zero-bit、逐音频波形优化；codec 冻结 | §3 |
| 主要单 codec | SNAC；文中不同位置出现 24/32kHz，复现需按具体实验确认 | §4、表 1/2 |
| 联合 C1 | SNAC32、DAC16、DAC44 | §5.2 |
| 联合 C2 / F1 | SNAC32+EnCodec24+EnCodec32 / SNAC24+DAC24+EnCodec24 | §5.2 |
| 跨架构 D1 / D2 | SNAC24+DAC44+FunCodec / SNAC24+FunCodec+APCodec | §5.2 |
| 优化 | Adam、150 步；正文学习率等需代码核对 | §3 |
| 数据量 | 每域 120 条，水印/干净各 60 | §4 |
| 数据集 | AIR、Clotho、DAPS、LibriSpeech、jaCappella、PCD、MAESTRO、GuitarSet；DSP 表另有 Freischuetz | §4、表 1/3 |
| 阈值 | $k=1.5$；$\gamma_c=1.5$（单视图） | §3 |
| 完整超参 | SDR 数值、初始化、码本选择、量化梯度、种子、成本需逐实验核对 | 原文未完整汇总 |

官方仓库提供 `watermark_research/src/latentmark.py`、`benchmark.py`、`summarize.py` 及质量评估入口，明确标注各表命令。它可补足论文细节，但仓库默认值不能无标注地写成正文事实。

## 4. 与其他方法对比

| 方法类别 | 核心思想 | 优点 | 局限 | 本文差异 |
|---|---|---|---|---|
| AudioSeal/WavMark/SilentCipher | 波形中嵌入神经水印 | 可直接处理现成音频 | 神经 codec 可能清除残差 | 让波形对应潜表示方向变化 |
| codec 内嵌水印 | 改 codec 或联合训练 | 编码过程可主动保留标记 | 发布端必须用指定 codec | 本文用冻结代理 codec 优化现成音频 |
| LAW 等生成式 latent 水印 | 在生成先验/轨迹嵌入 | 与生成流程结合 | 必须控制生成输入或模型 | Latent-Mark 是 post-hoc，非生成噪声重排 |
| Latent-Joint | 多代理潜投影联合优化 | 扩展 codec 迁移 | 成本上升，跨远架构仍困难 | 归一化目标与多视图聚合 |

## 5. 实验表现与优势

### 5.1 任务、口径与主表

表 1 的 Det. 是 Accuracy(TPR/FPR)，Sur. 是压缩后的水印检出率。Latent-Cluster 在 DAPS 上 Det.99.2%(100/1.7)、Sur.93.3%；LibriSpeech 上 Det.100%(100/0)、Sur.80.8%。相同压缩下 AudioSeal 的 Sur.分别只有 8.3%、5.0%，WavMark 与 SilentCipher 在这两域接近或等于零。该结果直接支持神经压缩下的优势。

优势并非每域达到接近满分：AIR 与 Clotho 的 Cluster Sur.为61.7%、58.3%。Joint 在 AIR/Clotho 为53.3%/58.3%，DAPS 为76.7%，并非联合训练在同一攻击下总是更好；DAPS Joint 的 FPR 达20%。不能只摘 Sur.而忽略检测阈值的代价。

### 5.2 未见 codec 迁移

表 2 先筛选能经 SNAC24 存活的水印，再评测 SNAC44、EnCodec48、DAC24，所以是条件迁移成功率。C1 在 LibriSpeech 的对应结果为100/85/100%，C2 为80/100/79.17%。这些数字不能当作所有输入音频的无条件整体成功率。

包含目标架构家族的代理组合通常优于远架构组合；例如 LibriSpeech D1 对三 codec 为65.67/70.80/73.33%，D2 为47.50/69.17/44.17%。结论是代理结构选择影响迁移，未证明普适不变潜空间。

### 5.3 保真度

图 3 用 $\Delta$SI-SNR 与 UTMOS。作者报告各方法 UTMOS 接近，波形指标有差异；这是预测器与客观波形比较，不是人工 MOS。图中缺少可直接准确抄录的汇总数字，不补造 PESQ/MOS。逐样本优化需要同时评估嵌入耗时与质量预算，正文没有完整成本表。

### 5.4 常规攻击与容量

表 3 四项 DSP：Gaussian SNR60dB、幅度0.5、低通4kHz、重采样16kHz。在 LibriSpeech 上 Latent-Mark 的通过率为100/100/75.81/100%；AudioSeal 均100%。在 Freischuetz 上分别69.86/71.23/56.16/64.38%，显示不同域仍有失效。60dB 噪声是轻量扰动，不能据此声称抗强噪声。

本文是零比特，没有消息容量/bit rate 对比，也无 VC、ASR-TTS、目标参考克隆和自适应白盒移除的完整测试。

### 5.5 消融

比较 Cluster、PCA、Random 方向，Cluster 在若干域更稳定，但不是每项最佳。例如 AIR 的 Random Sur.66.7% 高于 Cluster61.7%；DAPS 为65.0% 低于93.3%。这支持方向选择存在域依赖，而非聚类轴保证最优。联合组合消融较充分，但 codec 家族、采样率、预算和检测集的作用需要进一步控制。

## 6. 学习与应用

[官方代码](https://github.com/yenshan0530/Latent-Mark) 已核实为对应论文仓库。最小复现按单 SNAC 视图、固定校准集、同样的60/60测试分离做 clean 与一个 codec；然后加联合视图。先检查编码后的投影和检测阈值，再检查实际压缩后分数，避免只报告优化损失。

研究启发是把“攻击通道保留哪些表示”作为嵌入设计依据。迁移到其他语音生成器时不必改生成器，但要重新校准 codec 潜表示、预算和质量。在自己的鲁棒性比较中应保留神经 codec 的明确名称/码率、统一 FPR 与质量预算，并把条件迁移率另列。

## 7. 总结

优化波形以移动码本潜表示。

## 8. 图表精读与证据链

方法示意和投影分布说明潜空间偏移如何成为存在性信号；表 1 证明相对传统基线的神经压缩存活优势；表 2 支撑代理家族选择影响迁移；图 3 支撑质量较接近；表 3 界定 DSP 鲁棒性。最需注意表 2 的先筛选条件、表 1 部分高 FPR、图 3 的预测 MOS 与人工听评差异。码本几何解释是合理假说，但没有直接证明聚类轴就是跨 codec 的语义不变量。

## 9. 复现难度与适合人群

难度中高。依赖多 codec 权重、跨采样率可微流程、量化梯度、基线权重、多域音频和质量工具。官方脚本降低启动成本，全部 codec 联合优化会增加显存与时间。适合关注神经重建清洗、post-hoc 水印与跨 codec 泛化的研究者。

## 10. 简短全面总结

Latent-Mark 针对神经音频 codec 抹除微弱波形水印的问题，提出零比特后处理水印。它不重新训练生成器，而是冻结代理 codec，对每条发布音频优化波形扰动，让量化潜表示沿由码本聚类形成的方向移动，并用幅度预算限制质量损失。检测端通过干净样本校准投影均值、方差和阈值，以标准化分数判断标记存在。跨 codec 版本对多个代理的归一化缺口联合优化，并聚合多视图分数，减少单一量化规则过拟合。实验在语音、音乐和环境声音上比较 AudioSeal、WavMark、SilentCipher，DAPS 与 LibriSpeech 的单视图神经压缩存活率分别达到93.3%和80.8%，但环境声音和部分联合设置较弱。迁移表证明结构相近的 codec 家族更易共享有效方向，不过采用先筛选后测试的条件口径。质量采用 UTMOS 与波形指标，传统 DSP 仍有域依赖。贡献是将抗重建目标移入 codec 潜表示，关键限制是零比特容量、逐音频优化成本和缺少知晓检测器的自适应攻击验证。

## 11. 论文写作逻辑分析

引言先用传统 DSP 成功与神经重建失败建立新攻击面，再从信息瓶颈自然推出潜表示策略。这种从通道机制推设计的写法比只增加训练增强更有解释力。方法按单 codec → 多 codec → 聚合展开，便于看清每个新增模块的目的。

实验组织把主压缩、方向消融、代理选择、质量、DSP 分开，基本对应动机。论证最强的是同一压缩下对传统基线的大幅优势；较弱的是“共享不变量”与“普适框架”外推，以及条件迁移率容易被误读。论文内数据集数和某些采样率描述不完全统一，复现文档宜逐表列出确定配置。可借鉴其机制导向的问题定义，并在自己的摘要把无条件检出率、FPR、质量与成本同时交代。

## 12. 论文信息

- Authors: Yen-Shan Chen, Shih-Yu Lai, Ying-Jung Tsou, Yi-Cheng Lin, Bing-Yu Chen, Yun-Nung Chen, Hung-yi Lee, Shang-Tse Chen.
- Venue: Proc. Interspeech 2026, Long Track, pp. 3052–3061.
- DOI: [10.21437/Interspeech.2026-1979](https://doi.org/10.21437/Interspeech.2026-1979).
- [Paper](https://www.isca-archive.org/interspeech_2026/chen26u_interspeech.html) | [PDF](https://arxiv.org/pdf/2603.05310) | [arXiv](https://arxiv.org/abs/2603.05310) | [Code](https://github.com/yenshan0530/Latent-Mark).
- Version: 精读为 v3；v1 标题中的 Neural Resynthesis 不作为新论文。
- Citation: Chen, Y.-S., Lai, S.-Y., Tsou, Y.-J., Lin, Y.-C., Chen, B.-Y., Chen, Y.-N., Lee, H.-y., & Chen, S.-T. “Latent-Mark: An Audio Watermark Robust to Neural Codec Compression.” Proc. Interspeech 2026, 3052–3061. doi:10.21437/Interspeech.2026-1979.

## Overview 条目

```md
- [Latent-Mark: An Audio Watermark Robust to Neural Codec Compression](./Watermarking/post_hoc_watermarking/2026-Latent-Mark.md)  
  *Proc. Interspeech 2026, pp. 3052–3061, 2026*  
  Citation: Yen-Shan Chen, Shih-Yu Lai, Ying-Jung Tsou, Yi-Cheng Lin, Bing-Yu Chen, Yun-Nung Chen, Hung-yi Lee, Shang-Tse Chen. “Latent-Mark: An Audio Watermark Robust to Neural Codec Compression.” Proc. Interspeech 2026, pp. 3052–3061, 2026. doi:10.21437/Interspeech.2026-1979.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2026/chen26u_interspeech.html) | [PDF](https://arxiv.org/pdf/2603.05310) | [DOI](https://doi.org/10.21437/Interspeech.2026-1979) | [arXiv](https://arxiv.org/abs/2603.05310) | [Project/Code](https://github.com/yenshan0530/Latent-Mark)  
  Code: 已验证官方仓库；精读v3，v1旧标题不另入库
```
