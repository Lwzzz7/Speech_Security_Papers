# ALIGNED-IS：自回归音频生成模型的鲁棒无失真水印

## 0. 摘要翻译

ALIGNED-IS 解决自回归音频统计水印中的 retokenization mismatch：生成 token 经 vocoder 再编码后常变成相邻 token，使文本水印检测失效。方法先按音频 token embedding 聚类，把声学相近 token 视为等价；再用 aligned inverse sampling 重排各 cluster 在 [0,1] 的概率区间，使密钥伪随机数与预期 cluster 对齐。它保持原 token 分布的期望不变，并在 SpiritLM、SpeechGPT、多数据集和13种攻击上显著优于其他无偏统计水印。

## 1. 方法动机

前作已经证明直接移植 KGW/DiP 会因 decode→encode 的 token 替换大幅掉检。新的观察是：错配通常不是任意 token，而是连续声学特征位于量化边界附近，被分到 embedding 相近的码本项。若检测从“必须恢复同一个 token”放宽为“恢复到同一声学 cluster”，就能吸收错配，同时仍在 cluster 内按原概率选 token，以保留生成分布。

## 2. 威胁模型解读

提供者控制自回归模型采样并持有密钥；检测者需密钥、音频 tokenizer 与离线 cluster 映射，但不需访问生成 logits。攻击者可对波形施加13种 no-box 单一攻击：回声、30 dB噪声、低通、平滑、0.9/1.1倍速、EnCodec、MP3、Opus、64级量化和去噪。未评估攻击叠加、白盒密钥推断、伪造/拷贝、上下文操控或专门破坏帧对齐。方法对变速明显脆弱，安全边界本质上是 tokenizer 帧同步仍大致保留。

## 3. 方法设计与理解

### 3.1 离线 token 聚类

从模型的 audio-token encoder 取得全部 token embedding，以 k-means 划分 h 个 cluster；每个模型只做一次并保存映射，推理无额外聚类成本。选择欧氏 k-means 是因为原 tokenizer 本身也按与码本向量的欧氏距离量化。SpiritLM 六个数据集上原 mismatch 约0.3587–0.4295，cluster 等价后降至0.2109–0.2300，减少40.47%–46.44%。

### 3.2 Aligned inverse sampling

普通 inverse sampling 把各 cluster 总概率依次铺在 [0,1]；生成时知道概率边界，检测时却不知道 logits，只看到 token，因而无法确认密钥随机数 r 是否落在该 cluster 的真实区间。ALIGNED-IS 把每个 cluster 对齐到固定宽度 1/h 的检测区间：大于1/h的概率质量切出多余部分，小于1/h的空隙用其他 cluster 的溢出填充。它不改变 cluster 总概率，cluster 内仍按原模型条件概率采 token，因此对随机水印码取期望等于原分布。

### 3.3 生成、检测与统计保证

每步由密钥和前一个 token（1-gram）生成水印码及 r；若该码已在历史中出现，为保持多次生成的无偏性，直接按原分布采样；否则用 aligned inverse sampling 选 cluster，再在 cluster 内采 token。检测将波形重新编码，对每步重建 r；若当前 token 所属 cluster 与 r 对应的固定区间一致，记1分。无水印时总分按成功率1/h的二项分布建模，用 Hoeffding 尾界计算给定 FPR 的阈值。

### 3.4 复现配置

| 项目 | 设置 |
|---|---|
| 模型 | SpiritLM、SpeechGPT |
| 数据 | MMW三任务、Dolly-CW、Longform/Finance QA、LibriSpeech |
| 每任务 | 500条 |
| 密钥上下文 | prefix 1-gram |
| 聚类 | k-means，h=20，另做线性分配拉开 centroid |
| 攻击 | 13种独立 no-box 攻击，不叠加 |
| 硬件 | NVIDIA A6000 |
| 指标 | TPR@FPR=1%/0.1%、median p-value、FAD、语音质量/ASR |

## 4. 与其他方法对比

KGW/Unigram 通过偏置换检测率，会改变分布；DiPmark、γ-reweight 和 ITS 无偏，但不利用音频 token 的声学几何。ALIGNED-IS 的创新是把 tokenizer 的量化结构纳入采样与检测，对 cluster 而非 token 对齐。相比 AudioSeal/WavMark，它不改波形、无需在线嵌入，但要求控制自回归采样并依赖特定 tokenizer 聚类。

## 5. 实验表现

### 5.1 可检测性

SpiritLM 上 ALIGNED-IS 在 MMW Book/Story/Fake News 的 TPR@1%FPR 为0.88/0.92/0.97，Dolly/Longform/LibriSpeech 为0.92/0.96/0.95；相比 ITS 通常提升约0.10以上。SpeechGPT 的 Dolly/Longform/Finance 为0.82/0.95/0.94，明显高于 DiPmark 和 γ-reweight，并接近或超过有失真的 KGW强配置。

### 5.2 攻击鲁棒性与边界

SpiritLM Longform 无攻击0.96；回声、噪声、低通、平滑为0.93/0.93/0.94/0.94；EnCodec、MP3 32/40、Opus 16/31、量化、去噪为0.92/0.80/0.84/0.91/0.92/0.88/0.92。0.9/1.1倍速仅0.31/0.28，因为帧错位破坏 n-gram 与 cluster 时序。WavMark 对部分攻击为1.0，但 EnCodec和量化仅0/0.06；说明两者失效面不同。

### 5.3 无失真质量与消融

Longform 的 FAD：无水印0.0337、ALIGNED-IS 0.0416、AudioSeal 0.3061、WavMark 1.7910。LibriSpeech 上 speaker similarity/CER/WER 为0.7548/0.1038/0.1472，几乎等同基线0.7550/0.1039/0.1475。cluster 数并非越多越好：SpiritLM h=20 TPR 0.922，h=150仅0.688；SpeechGPT h=20为0.816。检测随时长提升，超过约5秒才较可靠。帧内时间偏移从0到320 samples时 TPR从0.94降到0.43，随后因接近下一帧重新上升，直接揭示同步弱点。

## 6. 学习与应用

适合可控制 logits 的自回归音频服务，尤其希望零比特来源检测且不改变生成分布的场景。工程上应为每个 tokenizer 独立聚类，缓存 token→cluster 映射，并在检测时尝试少量帧偏移以缓解裁剪错位。对短音频、变速和未知 tokenizer 不应直接部署。

## 7. 总结

以声学等价簇修复音频统计水印错位。

## 8. 图表精读与证据链

图1定义问题；表1证明聚类确实减少错配；表2–4验证跨模型/数据检测；表5/6覆盖威胁模型；图2和表23支持无失真；表19/20给出cluster甜点；表21/22暴露时长与同步边界。证据链完整，但攻击均为单一 no-box，尚缺自适应攻击和真实重录。

## 9. 复现难度与适合人群

难度中等。作者公开代码，方法不训练新网络，主要依赖 SpiritLM/SpeechGPT 推理、embedding导出、k-means与 logits processor。最小版本可用 SpiritLM、h=20、Dolly-CW，复现错配率和 TPR。适合统计水印、音频tokenizer和生成内容检测研究者。

## 10. 简短全面总结

ALIGNED-IS 针对生成 token 经 vocoder 后无法精确重编码的问题，先按声学 embedding 聚类，再通过对齐逆采样把密钥随机数映射到固定 cluster 区间；cluster 内仍按原模型概率采样，因此在随机水印码下保持原分布。SpiritLM 与 SpeechGPT 上，它相对其他无失真水印的 TPR普遍提升至少约10个百分点，在压缩、噪声、滤波和去噪下保持较高检测，同时 FAD、说话人相似度和 ASR指标接近无水印基线。主要边界是变速、帧错位、短于约5秒以及每个 tokenizer 都需重新聚类。

## 11. 论文写作逻辑分析

这篇是前作的标准“问题—机制观察—专用修复”升级：先用图1说明错位，再以embedding几何推出cluster等价，随后设计既可检测又无偏的aligned sampling。实验顺序也紧扣主张：聚类有效性、检测、攻击、质量、cluster数、时长和同步。最值得借鉴的是主动报告倍速与帧偏移失败曲线；需要谨慎的是“鲁棒”仍限定于无盒单攻击，并不代表对自适应水印移除安全。
