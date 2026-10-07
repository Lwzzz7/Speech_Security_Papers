# SSTMark：无需训练的鲁棒语义级语音水印

## 0. 翻译摘要原文

随着语音生成模型变得越来越逼真且易于获取，人们对合成语音滥用、来源归属和治理的担忧持续增加。水印为合成语音提供了一种实用的可追踪与可验证机制。现有大多数语音水印方法把水印信息嵌入波形、频谱等信号级表示中；当失真足够强时，嵌入的水印可能被削弱或破坏，导致检测能力下降。本文提出 SSTMark，一种通过文本水印在语义层运行、无需训练的语音水印框架。不同于传统信号级水印，SSTMark 将水印信息编码进生成语音所表达的语言内容，并从恢复出的语言内容中检测水印。AudioMarkBench 上的实验表明，SSTMark 取得了最强的平均鲁棒性。在固定 1% 假阳性率下，与最先进基线相比，SSTMark 在信号处理编辑和压缩编辑上的平均检测率分别提高 4.6% 和 16.9%。

## 1. 方法动机

### 1.1 为什么从信号级转向语义级

多数语音水印把标记写入波形、频谱或神经潜表示，因此验证依赖某种低层声学结构在传播后仍然存在。噪声、滤波、变速、重采样、传统 codec 和神经 codec 会改变这些结构，却常常不影响听众理解语音内容。恶意使用者为了让伪造语音继续传递欺骗性消息，也必须保留可懂度和主要语义。

SSTMark 利用这一不对称性：低层声学细节可能剧烈变化，但语言内容往往比波形稳定。因此，作者不再把“水印载体”限定为音频样本，而是把语音先转写为文本，通过受语义约束的同义词替换在文本中制造带密钥的统计偏置，再把改写文本重新合成为语音。只要攻击后的音频仍可被 ASR 恢复出足够文本，检测器就有机会找回水印证据。

### 1.2 现有方法的痛点

1. **强信号失真会抹去声学水印。** 特别是神经 codec 会重新编码和重建波形，使许多波形水印完全失效。
2. **攻击类型不可预知。** 依靠训练期 noise layer 模拟已知攻击，可能对未见分布或更强失真泛化不足。
3. **水印模型与生成器耦合。** 一些生成式水印需要访问内部 token、latent 或模型参数，难以包装任意黑盒语音生成器。
4. **只用波形相似度衡量质量不适合重合成。** SSTMark 会修改措辞并重新生成整段语音，即使语义和说话人保持，样本级波形也不可能对齐。

### 1.3 核心 insight

SSTMark 的核心可以概括为：**把语音水印改写为“经过语音通道的带密钥文本水印”。**

它将常见音频攻击视为一个只要仍保留可懂度，就会保留大部分文本 token 的噪声通道。水印不需要从精细声学残差中恢复，而是依靠同一密钥定义的 token 二进制分组，在 ASR 转写结果中检测 bit-1 比例是否显著高于随机文本的 0.5。

## 2. 威胁模型解读

### 2.1 参与方与系统边界

- **语音生成方/水印嵌入方**：能调用任意语音生成模型、STT、文本水印重写器和 TTS，并持有共享水印密钥 $k$。
- **验证方**：持有同一密钥 $k$、STT 和文本检测器，从可疑语音的转写文本计算水印分数。
- **普通用户或传播平台**：可能对语音做压缩、重采样、滤波、变速、量化或神经 codec 重建。
- **攻击者**：目标是让带水印语音无法被检测，同时尽量保留可懂语义和音质；论文评估 no-box 信号处理以及查询式决策边界攻击。
- **系统外部**：密钥分发、身份归属、密钥吊销、说话人授权、文本内容签名和多用户追踪不在论文方法中。

### 2.2 攻击者目标

主要目标是**水印移除**：攻击者希望使检测 z-score 低于阈值，同时让语音仍能使用。论文没有系统评估**水印伪造**，即攻击者把无水印语音处理成能在目标密钥下通过检测；也没有评估将水印文本统计偏置复制到另一段语义内容、密钥猜测或多密钥合谋。

### 2.3 攻击者能力与知识

AudioMarkBench 的 no-box 设置允许攻击者处理音频，但不查询检测器，覆盖 Gaussian noise、低通、高通、变速、平滑、回声、MP3、量化、SoundStream、Opus 和 EnCodec。HSJA 实验允许攻击者在波形空间进行 100、300 或 500 次决策查询，并寻找能翻转检测结果的对抗样本。

论文没有明确把文本水印算法或密钥暴露给攻击者。若攻击者知道 SSTMark 使用 ASR 与同义词统计偏置，即使不知道密钥，也可以直接执行“ASR → 独立释义 → TTS”，有较大概率重写 token 关系。若攻击者知道密钥，则可以定向把 bit-1 token 替换成 bit-0 token。此类语义级自适应攻击并未评估。

### 2.4 攻击约束

信号攻击需要保持语音可懂。作者使用不同强度的信号与 codec 变换，并在 HSJA 成功样本上使用 PESQ、SI-SNR、ViSQOL 测量攻击代价。水印本身还受语义约束：同义词候选必须同时通过句子级和词级相似度阈值；最终语音应保持语义、说话人特征和整体音质。

### 2.5 防御者能力与安全目标

防御者可以完全重写生成语音的转写文本，并用 TTS 重新合成，因此其能力明显强于只允许微小波形扰动的传统 post-hoc 水印。安全目标是：在固定 FPR 下获得稳定 TPR，并在重写与重合成后维持语义和说话人一致性。

该方法检测的是“文本中存在与密钥匹配的统计偏置”，不是恢复多 bit 身份消息，也不是密码学签名。因此它更接近 keyed presence detection，而不是用户级 attribution 或不可否认认证。

### 2.6 现实性与关键边界

SSTMark 的现实优势是能作为任意语音生成器的外部 wrapper，不需要内部 hidden states 或训练数据。代价是它实际上重新生成整段语音：时长、韵律、情感、口音、音色细节和说话人特征都可能变化。默认实验平均语音约 40 秒，这为文本统计检验提供了大量 token；短命令、短客服回复、验证码和单句播报未得到充分验证。

此外，密钥只是哈希输入，没有公开密码学安全分析。水印强度来自 token bit-1 比例偏置；当文本过短、语言不是英语、ASR 错误高或攻击者主动释义时，统计功效会明显下降。

### 2.7 方法和实验的对应关系

STT/TTS wrapper 对应黑盒生成器适配；keyed token hashing 对应共享密钥检测；语义相似度过滤对应内容保真；AudioMarkBench 验证低层信号变化下的语义稳定；F5-TTS/Canary 替换验证模块泛化；HSJA 衡量波形空间查询攻击代价。缺少的是语义释义、翻译、ASR-TTS laundering、文本编辑、短音频、多语言和伪造攻击。

## 3. 方法设计与复现级理解

### 3.1 全局流程

SSTMark 是三阶段 post-hoc 流程。

**阶段一：预语音生成。** 给定文本提示 $p$，任意语音生成器输出原始语音 $s_{pre}$。该语音只作为中间结果，并不是最终发布版本。

**阶段二：语义水印注入。** Whisper 等 STT 将 $s_{pre}$ 转为文本 $x_{pre}$。文本水印模块在密钥 $k$ 控制下，为每个 token 根据自身、前一个 token 和密钥分配 0/1；对满足词性要求且被分到 bit-0 的词生成同义词候选，经过句子级和词级语义过滤后，优先替换为 bit-1 候选，使整段文本产生 bit-1 偏置。CosyVoice 等 TTS 使用原语音作为参考音色，将改写文本 $x_{wm}$ 重新合成为最终水印语音 $s_{wm}$。

**阶段三：水印检测。** 对可疑语音 $s_{sus}$ 再做 ASR 得到 $x_{sus}$，使用同一密钥重新计算 token bit，统计 bit-1 比例并转换成 z-score。z-score 超过在干净负样本上校准的阈值时判为有水印。

### 3.2 预语音生成

语音生成器记为 $G_{sp}$：

$$s_{pre}=G_{sp}(p).$$

该阶段不要求访问生成器内部文本、logits、codec token 或模型权重。只要输出能被外部 ASR 转写，就可接入 SSTMark。实验使用 SpeechGPT，并把 LibriSpeech train-clean-100 文本放入固定提示模板，要求模型朗读句子。

这一设计提升了架构通用性，但也引入一次不必要的信息回路：如果上游本来就有准确文本，仍先合成再 ASR 会增加识别错误。论文坚持从语音恢复文本，是为了模拟无法访问内部响应文本的端到端 speech model 场景。

### 3.3 基础文本水印

#### 3.3.1 上下文相关的 token bit

原始文本水印方法为第 $i$ 个 token 分配一个伪随机 bit。为避免同一 token 永远属于固定组，bit 同时取决于当前 token 和前一个 token：

$$b_i=RB(h(x_i)\oplus h(x_{i-1})),\qquad 2\le i\le n.$$

$h$ 是字符串哈希，$RB$ 把哈希结果映射到 0 或 1。对无水印自然文本，hash 随机性使 bit-0/bit-1 近似各占一半。

#### 3.3.2 同义词候选与语义过滤

若 token 通过词性过滤且当前 bit 为 0，系统使用 BERT 上下文过程生成候选同义词集合 $C$。候选 $s$ 只有同时满足句子级与词级相似度才被保留：

$$C'=\{s\in C\mid S_{sent}(s,x_i)\ge\tau_{sent},\ S_{word}(s,x_i)\ge\tau_{word}\}.$$

对 $C'$ 中每个候选重新计算 bit，只考虑能把当前位置变成 bit-1 的候选，再选择词级语义相似度最高者。顺序遍历并重复这一过程后，文本中 bit-1 的比例被推高，但句子语义尽量保持。

### 3.4 SSTMark 的密钥化改造

SSTMark 把共享水印密钥 $k$ 加进 token bit 的哈希输入：

$$b_i=RB(h(k)\oplus h(x_i)\oplus h(x_{i-1})).$$

候选同义词 $s_j$ 的 bit 使用相同前文和密钥：

$$b_j=RB(h(k)\oplus h(s_j)\oplus h(x_{i-1})).$$

因此，同一文本在不同密钥下会产生不同 bit 分组。嵌入器和检测器只有使用相同 $k$ 才能观察到一致的 bit-1 偏置。密钥化文本水印算子写为：

$$x_{wm}=W(x_{pre};k).$$

论文固定句子级阈值 $\tau_{sent}=0.83$，用 $\tau_{word}$ 控制水印强度。较小的 $\tau_{word}$ 放宽词级语义要求，允许更多候选被替换，水印更强但原文本措辞变化更大；默认取 $\tau_{word}=0.7$。

### 3.5 语音重新合成

水印文本由 TTS 模型转换回音频：

$$s_{wm}=T(x_{wm}).$$

实验使用 CosyVoice，参考音频取对应的无水印 $s_{pre}$，目的是让重合成语音继承原说话人。输入文本最长限制为 200 text tokens，平均输出时长约 40 秒。这里没有向波形叠加小扰动；$s_{wm}$ 是对新措辞的完整再合成，所以信号级 SNR/PESQ 与原语音并非天然适合作为水印嵌入保真指标。

### 3.6 检测端的统计检验

对可疑语音先执行：

$$x_{sus}=A(s_{sus}).$$

使用密钥 $k$ 按 Section 3.4 给每个可编码 token 分配 bit。令 $n_1$ 是 bit-1 数量、$N$ 是有效 token 总数，则：

$$\hat{p}=\frac{n_1}{N}.$$

无水印零假设设定 $p_0=0.5$，检测统计量为：

$$z=\frac{\hat{p}-p_0}{\sqrt{p_0(1-p_0)/N}}.$$

当文本够长时，$N$ 增大使相同比例偏移产生更高 z-score；反过来，短语音即使成功替换了若干词，统计功效也可能不足。最终检测过程写为：

$$z=D(x_{sus};k).$$

阈值不是按攻击条件重新选择，而是在 1,000 条干净无水印语音上分别校准到 FPR 1% 和 0.1%，随后固定用于所有攻击。这避免了检测器事先知道攻击类型，但没有验证被攻击负样本上的实际 FPR 是否仍保持目标值。

### 3.7 训练、嵌入与推理边界

- **SSTMark 本身不训练。** 文本水印规则、hash bit 和 z-test 都是固定算法。
- **依赖模块已经训练。** SpeechGPT、Whisper Large-v3、CosyVoice、BERT 同义词过程和 WavLM 均为预训练模型；“training-free”只表示作者不再训练水印系统。
- **嵌入时运行：** 生成器一次、STT 一次、逐 token 候选生成与过滤、TTS 一次。
- **检测时运行：** STT 一次和轻量文本统计检测。
- **密钥要求：** 嵌入和检测共享同一 $k$，论文未描述密钥长度、生成、轮换或多用户管理。

### 3.8 目标函数与优化过程

SSTMark 没有神经网络损失函数和反向传播。每个候选替换是离散搜索：先满足 $S_{sent}\ge\tau_{sent}$ 和 $S_{word}\ge\tau_{word}$，再从 bit-1 候选中选相似度最高者。整体权衡由 $\tau_{word}$ 控制，而不是通过联合损失学习。

检测阈值通过负样本经验分布校准，使：

$$P(z>\gamma\mid H_0)\approx\alpha_{FPR}.$$

其中 $\alpha_{FPR}$ 分别为 0.01 或 0.001。论文未提供实际阈值 $\gamma$ 数值，也未说明不同密钥是否独立校准。

### 3.9 复现配置表

| 项目 | 论文设置 | 来源 |
|---|---|---|
| 上游语音生成器 | SpeechGPT | Section 5.1 |
| prompt 来源 | LibriSpeech train-clean-100 全切分 | Section 5.1 |
| 默认 STT | Whisper Large-v3，默认解码 | Section 5.1 |
| 默认 TTS | CosyVoice，参考音频为对应 $s_{pre}$ | Section 5.1 |
| 替代 STT | Canary-1B-v2 | Appendix D |
| 替代 TTS | F5-TTS | Appendix D |
| 文本水印 | Yang et al. 的 BERT 上下文同义词替换 | Section 3.2 |
| 句子阈值 | $\tau_{sent}=0.83$ | Section 4.2 |
| 默认词级阈值 | $\tau_{word}=0.7$ | Section 4.2/5.2 |
| 文本长度上限 | 200 text tokens | Section 5.1 |
| 平均输出时长 | 约 40 s | Section 5.1 |
| 检测校准集 | 1,000 条干净无水印 SpeechGPT 语音 | Section 5.1 |
| FPR 工作点 | 1% 和 0.1% | Section 5.1 |
| 检测音频采样率 | 16 kHz | Section 5.1 |
| GPU | NVIDIA Tesla V100-SXM2-32GB | Section 5.1 |
| 信号基线 | AudiowMark、WavMark、Timbre、AudioSeal | Section 5.1 |
| 主 benchmark | AudioMarkBench no-box attacks | Section 5.1 |
| 随机种子 | 论文未提供 | — |
| 水印密钥格式与长度 | 论文未提供 | — |
| hash 函数与 RB 映射细节 | 论文正文未明确 | — |
| BERT/相似度模型精确 checkpoint | 论文正文未明确 | — |
| 运行时间与显存 | 论文未提供 | — |

### 3.10 最小可复现伪代码

```text
嵌入(prompt, key):
  pre_speech = SpeechGenerator(prompt)
  pre_text = STT(pre_speech)

  tokens = tokenize(pre_text)
  for each eligible token i from left to right:
    bit = RB(hash(key) xor hash(token[i]) xor hash(token[i-1]))
    if bit == 0:
      candidates = BERT_context_synonyms(token[i])
      candidates = filter_by_sentence_and_word_similarity(candidates)
      candidates = keep_candidates_with_keyed_bit_one(candidates, key, token[i-1])
      if candidates is not empty:
        token[i] = candidate_with_highest_word_similarity

  watermarked_text = detokenize(tokens)
  watermarked_speech = TTS(watermarked_text, reference=pre_speech)
  return watermarked_speech

检测(suspect_speech, key):
  suspect_text = STT(suspect_speech)
  bits = keyed_context_bits(suspect_text, key)
  p_hat = count(bits == 1) / count(bits)
  z = (p_hat - 0.5) / sqrt(0.25 / count(bits))
  return z
```

### 3.11 复现风险与信息缺口

1. 论文没有给出实际 hash、随机 bit 映射、token normalization、标点和大小写处理，任何差异都会让嵌入/检测不一致。
2. BERT 同义词生成、词性过滤、句子/词级相似度模型的精确实现依赖被引用工作，正文无法独立复现。
3. TTS 使用 $s_{pre}$ 作为 reference，但没有报告 prompt 长度、reference 截取长度、speaker embedding 或随机采样设置。
4. 检测阈值 $\gamma$、每条语音有效 token 数分布及最短可检测长度未给出。
5. “无需训练”不等于轻量：嵌入链需要 SpeechGPT、Whisper、BERT 和 CosyVoice，实际计算成本可能高于直接波形水印。
6. 默认平均 40 秒远长于许多真实短语音，平均鲁棒性可能受长文本统计优势影响。
7. 当前 PDF 保留 ACM 模板占位信息，但作者单位页面确认 ACM Multimedia 2026；引用时应使用已确认会议，不使用 PDF 中错误的 2018/Woodstock 占位内容。
8. 未找到官方公开代码仓库，补充材料只明确提供匿名音频样例。

### 3.12 Threat-model 映射

| 方法组件 | 对应攻击或目标 | 首先失效的更强设定 |
|---|---|---|
| STT 语义抽取 | 穿过噪声、滤波、codec 后恢复文本 | ASR 错误高、语言不支持、内容不可懂 |
| keyed token hash | 让检测依赖共享密钥 | 密钥泄露、hash 实现不一致、定向 bit 翻转 |
| 同义词偏置 | 在保持语义时提高 bit-1 比例 | 主动释义、翻译、文本摘要、短句 |
| TTS 重合成 | 把文本标记重新带回语音 | 音色/情感/韵律严格保持要求 |
| z-score 检测 | 在固定 FPR 下判断 presence | token 太少、负样本域偏移、被攻击负样本 FPR 上升 |
| backbone 替换 | 降低对单一 STT/TTS 的绑定 | 不同语言或更弱模型导致系统性转写错误 |
| HSJA 评测 | 衡量波形查询式移除代价 | 直接语义重写或知道密钥的文本级攻击 |

## 4. 与其他方法对比

### 4.1 与传统 post-hoc 波形水印的区别

WavMark、AudioSeal、Timbre 和 AudiowMark 都尽量保持原始波形，只加入难以感知的信号。SSTMark 则允许先改词再整段重合成，因此它不再追求波形不可察觉扰动，而追求语义和说话人层面的等价。其鲁棒性来自“攻击需保留语言内容”，不是来自某个声学水印模式本身特别稳固。

### 4.2 与生成式水印的区别

GROOT、ALIGNED-IS 等在扩散去噪或 autoregressive token 采样中嵌入水印，需要修改生成过程。SSTMark 完全位于模型输出之后，能包装黑盒语音生成器，但要付出一次 ASR 和 TTS 重合成。

### 4.3 与普通文本水印的区别

普通 post-hoc 文本水印的最终媒介就是文本；SSTMark 把文本水印夹在两个语音转换模块之间。STT/TTS 既是桥梁也是噪声源：它让水印跨越音频攻击，也可能因识别错误、同音词和合成差异破坏 token 偏置。

### 4.4 方法对比表

| 方法类别 | 水印载体 | 优点 | 缺点 | SSTMark 的差异 |
|---|---|---|---|---|
| AudiowMark | 传统信号域 | 简单、无神经训练 | 强噪声/codec 下脆弱 | 把载体提升到语言 token |
| WavMark | 神经波形特征 | 高保真、短窗消息 | 神经 codec 和部分滤波下崩溃 | 不依赖波形细节 |
| Timbre | 音色相关表示 | 多种 codec 下较强 | 强噪声与量化存在边界 | 通过文本保持更均匀的平均 TPR |
| AudioSeal | 学习式波形水印 | 高保真、定位能力 | 部分高通/Opus/SoundStream 下掉点 | 用 ASR 语义抵抗重构 |
| 生成式 token 水印 | 生成时 token/latent | 可穿过部分重合成 | 需控制生成模型内部 | SSTMark 是模型无关 post-hoc wrapper |
| SSTMark | 带密钥文本统计偏置 | 无需训练、跨强声学变化、平均鲁棒 | 改词并重合成、依赖长文本和 STT/TTS | 本文方法 |

### 4.5 适用范围

更适合长篇有声内容、播客、长回答、新闻朗读和允许轻微措辞变化的合成语音。不适合原文必须逐字一致的法律宣读、诗歌、歌词、代码、专有名词密集内容、短命令、实时低延迟语音，以及需要严格保留原始情感和韵律的场景。

## 5. 实验表现与优势

### 5.1 实验设计

作者使用 SpeechGPT 从 LibriSpeech train-clean-100 文本生成 pre-speech，Whisper Large-v3 转写，CosyVoice 以原语音为 reference 重合成。四个基线在相同生成语音和攻击条件下评估。每个方法用 1,000 条干净无水印语音独立校准 FPR 1%/0.1% 阈值，然后把阈值固定用于全部攻击。

质量不使用样本级 SNR/PESQ 作为主指标，而报告 BERTScore、BLEU-2/4、WavLM speaker similarity 和 FAD。这与方法改变文本并整段重合成的性质一致，但 FAD=0.001 的极低值只说明集合级音频分布接近，不能证明逐样本自然度或听感无差异。

### 5.2 语义、说话人和音频质量

默认 $\tau_{word}=0.7$ 时，SSTMark speaker similarity 为 0.985，低于信号级基线的 0.997–0.999，但仍较高；FAD 为 0.001，在表中最低。由于信号基线修改原语音而 SSTMark 重新合成，FAD 的横向公平性需谨慎解释。

Table 2 显示 $\tau_{word}$ 从 0.9 降至 0.6 时，BERTScore 从 0.984 降到 0.955，BLEU-2 从 0.858 降到 0.691，BLEU-4 从 0.808 降到 0.554；speaker similarity 基本稳定在 0.984–0.985，FAD 为 0.001–0.003。作者选择 0.7，此时 BERTScore 0.961、BLEU-2 0.714、BLEU-4 0.588，说明整体语义较近，但逐字措辞已有明显变化。

### 5.3 信号处理攻击鲁棒性

Table 3 在 FPR 1% 时，SSTMark 无攻击 TPR 只有 91.0%，低于所有基线的 100%；但在 10/5/0 dB Gaussian noise 下仍为 91.6/90.0/88.3%，在极端低通 250 Hz 下为 89.3%，高通 3 kHz 为 91.0%，三个速度条件约 89.5–91.6%，整体平均 90.2%。

对比方法在擅长条件中可达 100%，但在特定攻击下严重崩溃，例如 WavMark 在 250 Hz 低通为 0，AudioSeal 在 0 dB 噪声与 3 kHz 高通为 0/1.6。SSTMark 的优势不是峰值，而是所有攻击下相对平坦。其平均值比次优 Timbre 的 85.6% 高 4.6 个百分点，正对应摘要主张。

在 FPR 0.1% 时，SSTMark 平均 TPR 83.4%，仍高于 Timbre 81.4%、AudioSeal 71.5% 和 WavMark 66.2%，但 no-attack 只有 84.7%。这反映统计文本水印的基础检测上限不高，鲁棒性来自攻击前后都维持类似水平，而非干净条件接近完美。

### 5.4 压缩与神经 codec 鲁棒性

Table 4 在 FPR 1% 时，SSTMark 对 MP3 32/16 kbps 为 89.6/88.9%，16/4-bit quantization 为 90.2/90.4%，SoundStream 16/4 quantizers 为 90.2/88.9%，Opus 48/16 kbps 为 90.6/91.4%，EnCodec 为 90.2%，平均 90.0%。

次优 Timbre 平均 73.1%，SSTMark 提升 16.9 个百分点。基线表现呈明显攻击专化：Timbre 在 Opus 为 100% 但 SoundStream 只有 3.2%；AudioSeal 在 EnCodec 为 99.7% 但 Opus 为 0/0.1%；SSTMark 没有单项最高，却没有 codec 崩溃。

FPR 0.1% 时 SSTMark 平均 82.6%，仍显著高于 Timbre 66.5%、AudioSeal 56.1% 和 WavMark 23.7%。语义级载体确实避开了神经 codec 对低层波形模式的重构破坏。

### 5.5 HSJA 对抗移除

作者在波形域使用 HopSkipJumpAttack，以 100/300/500 次查询寻找检测失败样本。成功攻击 SSTMark 后，PESQ 只有 1.142/1.252/1.235，SI-SNR 为 2.688/4.619/4.132 dB，ViSQOL 为 1.652/1.797/1.785；AudioSeal 成功样本仍有 PESQ 4.088–4.416、SI-SNR 48.420–59.405 dB、ViSQOL 4.696–4.756。

这说明在相同波形决策攻击下，SSTMark 的边界不容易用小扰动跨越，因为攻击必须先显著影响 ASR 文本或统计 token 偏置。但该结论不能扩展到语义攻击；一次高质量 paraphrase + TTS 可能比在 640k 波形维度中盲搜高效得多。

### 5.6 STT/TTS backbone 消融

把 CosyVoice 换成 F5-TTS 后，信号攻击平均 TPR 在 FPR 1%/0.1% 下为 84.7%/75.4%，codec 攻击为 83.4%/73.8%；比默认 90.2%/83.4% 和 90.0%/82.6% 明显下降，但仍表现稳定。

把 Whisper 换成 Canary-1B-v2 后，信号攻击平均为 84.7%/73.0%，codec 攻击为 84.4%/74.0%。这证明方法不只适用于单一模型，但也表明论文主结果明显受 Whisper/CosyVoice 组合影响；“模型无关”应理解为可替换，而不是替换后性能不变。

### 5.7 消融充分性

$\tau_{word}$ 实验验证了强度—文本保真的权衡，STT/TTS replacement 验证了模块泛化。但论文没有报告：去掉 key 后的伪造率、不同文本长度的 TPR 曲线、不同语言、不同 prompt 类型、ASR WER 与 z-score 的关系、同义词替换比例、错误密钥检测、释义/翻译攻击，以及被攻击负样本 FPR。这些恰好是语义水印最重要的独特变量。

### 5.8 威胁模型覆盖与证据缺口

AudioMarkBench no-box 和 HSJA 都在波形层攻击，而 SSTMark 的载体在文本层。实验很好地证明“声学攻击打不到语义载体”，但没有证明“攻击者知道语义载体后仍难移除”。因此论文展示的是跨模态规避已有攻击面的优势，而非对自适应 semantic-level attacker 的完整安全性。

## 6. 学习与应用

### 6.1 开源情况

论文 Appendix B 明确提到匿名 demo 音频位于 supplementary material，但 PDF、arXiv 页面及公开搜索中没有找到官方代码仓库。因此当前应标记为 `Code: Not found`，不能把第三方论文索引仓库当作官方实现。

论文页面为 [arXiv:2607.17592](https://arxiv.org/abs/2607.17592)。作者单位和作者主页均将其列为 ACM Multimedia 2026 论文，但 PDF 仍保留 ACM 模板占位会议信息；以机构发布列表为较新依据。

### 6.2 推荐复现顺序

1. 先复现被引用文本水印，在纯文本上验证 $\tau_{word}$、z-score 和 FPR 校准。
2. 加入 key hash，测试相同文本在正确密钥与错误密钥下的 z-score 分布。
3. 用现成真实文本跳过 SpeechGPT，只做 text watermark → CosyVoice → Whisper → detection，确认跨语音通道闭环。
4. 再加入 SpeechGPT pre-speech 和 reference voice，测量 WER、speaker similarity、替换率及输出时长。
5. 最后跑 AudioMarkBench，并额外增加 paraphrase、翻译、摘要、ASR-TTS 和短文本攻击。

### 6.3 工程注意事项

tokenization、大小写、标点、数字规范化和 ASR 后处理必须在嵌入与检测端一致。密钥应使用明确 KDF 派生哈希种子，不能直接依赖不同语言运行时不稳定的内置 string hash。部署前应按语言、域和文本长度分别校准阈值，并在受攻击负样本上检查 FPR。对于短文本，可聚合多段、引入多 bit ECC 或改用生成时 token 水印。

### 6.4 可迁移方向

该思路可迁移到视频对白、语音代理长回复、有声书和语音翻译，只要最终内容允许受控释义。也可与 AudioSeal 等信号水印叠加：信号水印负责短片段、定位和逐字内容，SSTMark 负责穿过 codec 的语义级 presence。但两层需要共同定义密钥、身份和冲突处理，不能简单认为检测 OR 即等于可信来源。

## 7. 总结

**在语音文本语义中嵌入密钥偏置。**

## 8. 图表精读与证据链

| 图表 | 想证明什么 | 核心证据 | 证据边界 |
|---|---|---|---|
| Fig. 1 | SSTMark 的平均鲁棒性更均匀 | 多类 AudioMarkBench 攻击雷达图保持约 90% | 只涵盖声学攻击，不含语义自适应攻击 |
| Fig. 2 | 三阶段跨模态闭环 | SpeechGPT→STT→text WM→TTS；检测端 STT→z-test | 计算成本和时延未画出 |
| Table 1 | 重合成仍保持说话人和音频分布 | speaker sim 0.985，FAD 0.001 | FAD 是分布指标，不能替代逐样本听测 |
| Table 2 | $\tau_{word}$ 控制鲁棒—文本保真 | 阈值降低时 BERTScore/BLEU 下降，speaker sim 稳定 | 没直接同时展示每个阈值的 TPR |
| Table 3 | 信号攻击下无明显崩溃 | FPR 1% 平均 TPR 90.2%，所有条件约 88.3–91.6% | no-attack 仅 91%，长文本可能有利 |
| Table 4 | codec 尤其 neural codec 下稳定 | FPR 1% 平均 90.0%，比次优高 16.9 pp | 未测试 ASR-TTS/VC 语义保持攻击 |
| Table 5 | 波形查询攻击需要大失真 | 成功 HSJA 后 SSTMark PESQ 约 1.1–1.25 | 与 AudioSeal 比较的是不同检测机制，未含文本攻击 |
| Tables 6–9 | STT/TTS 可替换 | F5-TTS/Canary 下平均仍约 83%–85% | 较默认下降 5–10 pp，非性能无损替换 |

论文的主证据链非常直接：强声学处理保留语言内容 → 语义级水印不依赖波形细节 → Tables 3–4 显示各攻击 TPR 基本平坦。另一个重要链条是：水印通过改词实现 → 必须重新定义 fidelity → Tables 1–2 分别测量说话人、音频分布和文本语义。但安全证据只覆盖声学域攻击，没有闭合“攻击者针对文本水印”的自适应威胁。

## 9. 复现难度与适合人群

- **复现难度：中高。** 算法公式简单，但需要同时部署语音生成、Whisper、BERT 文本重写、CosyVoice 和完整 AudioMarkBench。
- **主要依赖：** SpeechGPT、Whisper Large-v3、CosyVoice/F5-TTS、BERT 同义词模块、WavLM speaker verifier、V100 级 GPU 和多个 codec。
- **最小可复现版本：** 使用真实文本和单一参考音频，先闭环 text watermark → TTS → STT → z-score，不必首先复现 SpeechGPT。
- **适合人群：** 语音水印研究者、文本水印研究者、语音代理治理、AudioMarkBench 攻防研究，以及探索跨模态 provenance 的研究人员。

## 10. 简短全面总结

SSTMark 针对波形和频谱水印在强噪声、压缩及神经 codec 下易失效的问题，将语音水印重新定义为经过语音通道的带密钥文本水印。系统先用 SpeechGPT 生成预语音，经 Whisper 转写后，按照密钥、当前 token 和前一 token 的哈希结果把词划分为 bit-0/1；对 bit-0 词执行受句子级和词级语义约束的同义词替换，增加 bit-1 比例，再由 CosyVoice 重新合成。检测端重新 ASR，并以 z-score 检验密钥相关偏置。默认配置在固定 1% FPR 下，对信号处理和 codec 攻击的平均 TPR 分别为 90.2% 和 90.0%，比次优基线高 4.6 和 16.9 个百分点，且 HSJA 成功移除需要显著波形退化。其核心代价是改写文本并整段重合成，no-attack TPR 仅 91%，依赖约 40 秒长文本及强 STT/TTS；论文也没有评估释义、翻译、ASR-TTS laundering 等直接攻击语义载体的自适应策略。

## 11. 论文写作逻辑分析

### 11.1 Intro 的问题铺垫

论文先用深度伪造风险引出主动 provenance，再将现有水印统一概括为 signal-level，并指出攻击可以破坏声学细节但必须保留欺骗语义。这个“攻击者仍需传递消息”的约束自然导出语义载体，问题铺垫非常清晰。

### 11.2 Insight 与 motivation

核心 insight 不是设计新的音频编码器，而是把文本水印嵌入语音生成后处理链。它利用模态转换改变攻击面：AudioMarkBench 大多数攻击只作用于波形，因此只要 ASR 仍能恢复文本，水印就存活。这个 insight 有效但也产生证据偏置——若攻击集合仍停留在信号域，新方法天然占优。

### 11.3 Threat model 的承接作用

论文在实验中明确写了 post-processing threat model 和 no-box 攻击，但没有独立安全小节，也没有讨论攻击者知道文本水印机制后的能力。因此方法和 AudioMarkBench 对齐良好，却没有完整限定 semantic attacker、key exposure 和 watermark forgery。

### 11.4 方法叙事

Preliminaries 先解释 STT/TTS 和被引用文本水印，再在 Proposed Method 中加入 key，并按 pre-speech、injection、detection 展开。模块出现顺序合理，每个公式都对应 Fig. 2 中一条数据流。方法本体创新主要是跨模态组合与密钥化改造，而非新的文本水印算法，论文对此表述较克制。

### 11.5 实验呼应

作者意识到整段重合成不能只用传统 waveform fidelity，于是先评估语义、说话人和 FAD，再评估鲁棒性；这个实验顺序与方法特性一致。固定 FPR 校准也比各方法使用默认阈值更公平。backbone replacement 进一步回应“是否只是 Whisper/CosyVoice 特例”。

### 11.6 证据链完整性

“信号攻击保留语义—语义水印鲁棒”证据完整；“改写与重合成仍忠实”只有自动指标，没有正式听测和细粒度情感/韵律评价；“对抗移除更难”只由 waveform HSJA 支撑，缺少针对文本层的自适应攻击。因此 strongest claim 应限定为 AudioMarkBench 声学攻击下的平均鲁棒性。

### 11.7 可借鉴写法

值得借鉴的是从攻击的功能约束推导新载体，而不是在既有信号编码器上继续堆 noise layer；同时根据方法改变重新设计 fidelity 指标，并在固定 FPR 下比较检测器。若进一步完善，应该在 threat model 中加入语义攻击，并新增“文本长度—替换率—z-score—TPR”四者关系，使检测统计功效更加透明。

## 12. 论文信息

- **论文题目：** SSTMark: Robust Training-Free Semantic-Level Speech Watermarking
- **中文题目：** SSTMark：无需训练的鲁棒语义级语音水印
- **作者：** Kuan-Lin Chu, Jun-Cheng Chen, Chun-Shien Lu
- **单位：** CITI, Academia Sinica；IIS, Academia Sinica
- **会议：** ACM International Conference on Multimedia（ACM MM）
- **年份：** 2026
- **arXiv：** [2607.17592](https://arxiv.org/abs/2607.17592)
- **PDF：** [arXiv PDF](https://arxiv.org/pdf/2607.17592)
- **代码：** 未找到官方公开代码仓库
- **备注：** PDF 仍保留 ACM 模板占位会议信息；会议归属依据 Academia Sinica 2026 publication list 与作者主页的 “to appear in ACM Multimedia” 信息

Overview 条目可写为：

```md
- [SSTMark: Robust Training-Free Semantic-Level Speech Watermarking](./Watermarking/post_hoc_watermarking/2026-SSTMark.md)  
  *ACM International Conference on Multimedia (ACM MM), 2026*  
  Citation: Chu, K.-L., Chen, J.-C., & Lu, C.-S. “SSTMark: Robust Training-Free Semantic-Level Speech Watermarking.” *Proceedings of the ACM International Conference on Multimedia*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2607.17592) | [PDF](https://arxiv.org/pdf/2607.17592) | Code: Not found
```
