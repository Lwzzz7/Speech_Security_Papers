# Adversarial Speech for Voice Privacy Protection from Personalized Speech Generation（AS-PSG）

## 0. 摘要翻译

个性化 TTS 与语音转换已使合成语音难以被人耳区分，因而需要防止他人利用公开语音克隆目标说话人。本文在尽量保持原语音不变的前提下，对参考语音加入对抗扰动，使下游语音生成模型不能正确提取目标说话人特征。作者以开源 YourTTS 为白盒目标，使用基于梯度的 I-FGSM 扰动其说话人编码器，并以生成语音的自动说话人验证评估保护效果。结果表明，该扰动可降低 YourTTS 生成目标音色的能力。

## 1. 方法动机

个性化生成把少量参考语音变成可用于 TTS、VC 的音色条件，公开录音因而可被滥用。普通加性噪声既可能明显损伤可听性，又未必改变模型使用的说话人表示。本文的直觉是：不必攻击整套生成器，只需使参考语音经过其 speaker encoder 后远离原说话人嵌入，便可使后续条件生成失配。

## 2. 威胁模型解读

防御者在发布自己的参考语音前本地加扰；攻击者收集这些已保护语音并将其作为 YourTTS 的 reference，用于零样本 TTS 或 VC。论文的扰动构造为白盒，已知 YourTTS 的 H/ASP speaker encoder 与参数；未证明对未知商业模型的迁移。攻击者可使用受保护音频生成，但不能拿到干净原声。安全目标是降低生成语音与受保护者的音色相似度，同时让原始受保护语音仍可听。

## 3. 方法设计与复现级解读

### 3.1 全局流程

原始波形先取 STFT，得到频谱 $x$；冻结 YourTTS 的 speaker encoder，迭代优化频谱扰动；最后复用原相位 iSTFT 回到波形。部署时，该波形仅作为 YourTTS 的 reference speech：文本 TTS 路径和源语音 VC 路径都由同一个被扰乱的 speaker embedding 条件化。

### 3.2 I-FGSM 说话人嵌入攻击

令原参考嵌入为 $e$，第 $i$ 次扰动频谱的嵌入为 $\tilde e_i$。作者用负余弦相似度

$$L(e,\tilde e_i)=-\frac{e^T\tilde e_i}{\|e\|_2\|\tilde e_i\|_2}$$

驱动梯度上升，使两嵌入远离。每步按 $\tilde x_{i+1}=\mathrm{Clip}_\epsilon(\tilde x_i+\alpha\,\mathrm{sign}(\nabla_{\tilde x_i}L))$ 更新；$I=1$ 即 FGSM。裁剪保证 $L_\infty$ 预算，原相位重建则是论文的波形输出方式。该目标直接对齐“reference embedding 不能代表本人”，但不直接优化生成端的文本、声码器或感知损失。

### 3.3 配置与复现边界

| 项目 | 论文设置 |
|---|---|
| 目标模型 | 开源 YourTTS，H/ASP speaker encoder |
| 数据 | LibriSpeech test-clean，16 kHz |
| 特征 | 512 bins、Hanning 窗、25 ms 窗长、10 ms 跳帧 |
| FGSM | $\epsilon=0.02$ |
| I-FGSM | 50 步，$\alpha=0.0004$ |
| 评价 | PESQ、SNR、嵌入余弦距离；ECAPA-TDNN ASV 的 EER |

论文未给出所有 STFT 幅度/归一化细节及随机种子。白盒 encoder 梯度是该方法的关键依赖；若模型/前处理未知，构造步骤将首先失效。

## 4. 与其他方法对比

| 方法 | 攻击位置 | 优点 | 局限 |
|---|---|---|---|
| 高斯噪声 | 波形 | 简单 | 未对齐音色表示 |
| FGSM | speaker encoder | 快 | 单步效果有限 |
| I-FGSM（本文） | speaker encoder | 直接拉开身份嵌入、可用于 TTS/VC | 白盒且只评 YourTTS |

## 5. 实验表现与优势

表 1 检查“保护语音是否仍可用”：I-FGSM 的 SNR 37.00 dB、PESQ 4.17，高于所列 Gaussian/FGSM，同时嵌入距离最大。随后以 TTS 与 VC 输出的 EER 衡量克隆失败程度；较高 EER 表示 ASV 更难确认生成语音属于原说话人。实验逻辑覆盖了感知质量与下游音色保护，但只在白盒 YourTTS 与 ASV 代理指标上成立。

## 6. 学习与应用

论文使用开源 YourTTS 与 ASV-subtools，适合复现数字域最小版本；当前材料未给出作者独立代码仓库。实现时应先复现 STFT 上的 embedding loss，再检验 iSTFT 后生成链路。可迁移至任何“参考语音→speaker encoder→生成器”结构，但必须重做模型迁移验证。

## 7. 总结

以 I-FGSM 破坏参考语音的说话人嵌入来阻断克隆。

## 8. 图表精读与证据链

图 1 定位了 YourTTS 中共同使用 reference embedding 的 TTS/VC 接口；图 2 给出频谱梯度攻击数据流；表 1 将听感与嵌入偏移并列；表 2 将偏移落到生成语音 ASV。缺少跨架构、去噪和物理传播控制，故证据是特定白盒模型的保护证据。

## 9. 复现难度与适合人群

难度为**中等**：需要可微 YourTTS、GPU 和 ASV 模型，但无需重新训练生成器。适合语音隐私、speaker encoder 鲁棒性和生成式语音安全研究。

## 10. 简短全面总结

AS-PSG 把个性化语音生成的隐私风险归结为参考语音中的 speaker embedding 可被稳定提取。方法不攻击文本或声码器，而在 STFT 域对白盒 YourTTS 的 H/ASP encoder 运行 FGSM/I-FGSM，最大化干净与保护语音嵌入的余弦距离，并以原相位重建保护音频。LibriSpeech 上，I-FGSM 在 SNR、PESQ 与嵌入偏移间取得优于高斯噪声和单步 FGSM 的结果；后续 TTS/VC 的 ASV EER 上升，说明生成音色更难被认成原说话人。其贡献是以生成模型内部身份表示作为保护靶点；边界在于白盒、单一生成器和未覆盖自适应净化。

## 11. 论文写作逻辑分析

文章以“参考音频可驱动 TTS 和 VC”建立风险，继而把 speaker encoder 指为最短攻击路径。方法从 FGSM 背景到 I-FGSM 目标再到生成评估，叙事简洁。实验把保护语音质量、表示偏移和下游生成依次连接，但没有把白盒结果延伸到真实平台，因此结论边界应更明确。
