# Speech_Security_Papers

这是一个面向**语音安全**的论文精读库，主要整理语音水印论文，同时也整理语对抗样本以及音频大模型越狱等安全相关的代表性研究。每篇条目提供精读笔记、发表信息、论文链接和可获得的官方代码/项目链接，便于按研究问题追踪方法演进并快速定位可复现工作。By [Weizhi Liu](https://scholar.google.com/citations?user=4y-6mXgAAAAJ&hl=zh-CN)

## 快速导航

- [语音水印](#speech-watermarking)：[生成式水印](#generative-watermarking) ｜ [后处理水印](#post-hoc-watermarking)
- [语音隐写](#steganography)：主要集中于生成式隐写
- [对抗样本](#adversarial-examples)：面向语音合成与音频大模型的攻击和防护
- [越狱](#jailbreak)：音频大语言模型越狱攻击、红队评测与基准
- [Other Security](#other-security)：有意思的论文（主要来自四大会及各顶会顶刊）

## Speech Watermarking

### Generative Watermarking

#### 2026

- [Hidden in Plain Tokens: Simply Robust, Gradient-Free Watermark for Synthetic Audio](./Watermarking/generative_watermarking/2026-HPT.md)  
  *International Conference on Machine Learning (ICML), 2026*  
  Citation: Milis, G., Qin, Y., Wu, Y., & Huang, H. “Hidden in Plain Tokens: Simply Robust, Gradient-Free Watermark for Synthetic Audio.” *Proceedings of the 43rd International Conference on Machine Learning*, 2026.  
  Links: [Paper](https://openreview.net/forum?id=h4bSJMaNgb) | [Code](https://github.com/g-milis/nograd-audio-wm)

#### 2025

- [Robust Distortion-Free Watermark for Autoregressive Audio Generation Models (ALIGNED-IS)](./Watermarking/generative_watermarking/2025-ALIGNED-IS.md)  
  *Conference on Neural Information Processing Systems (NeurIPS), 2025*  
  Citation: Wu, Y., Milis, G., Chen, R., & Huang, H. “Robust Distortion-Free Watermark for Autoregressive Audio Generation Models.” *Advances in Neural Information Processing Systems 38*, 2025.  
  Links: [Paper](https://neurips.cc/virtual/2025/poster/117426) | [Code](https://github.com/g-milis/AlignedIS)

- [A Watermark for Auto-Regressive Speech Generation Models](./Watermarking/generative_watermarking/2025-AR-Speech-Statistical-Watermark.md)  
  *Annual Conference of the International Speech Communication Association (Interspeech), 2025*  
  Citation: Wu, Y., Chen, R., Milis, G., Guo, J., & Huang, H. “A Watermark for Auto-Regressive Speech Generation Models.” *Interspeech 2025*, 3474–3478, 2025.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2025/wu25k_interspeech.html) | [Code](https://github.com/g-milis/AlignedIS)

- [Latent Watermarking of Audio Generative Models](./Watermarking/generative_watermarking/2025-Latent-Watermarking.md)  
  *IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), 2025*  
  Citation: San Roman, R., Fernandez, P., Deleforge, A., Adi, Y., & Serizel, R. “Latent Watermarking of Audio Generative Models.” *ICASSP 2025*, 1–5, 2025.  
  Links: [Paper](https://doi.org/10.1109/ICASSP49660.2025.10889782)

- [Poisoning The Diffusion: A Simple and Robust Watermarking Method for Audio Generation (PTD)](./Watermarking/generative_watermarking/2025-PTD.md)  
  *IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), 2025*  
  Citation: Tang, Y. “Poisoning The Diffusion: A Simple and Robust Watermarking Method for Audio Generation.” *ICASSP 2025*, 1–5, 2025.  
  Links: [Paper](https://doi.org/10.1109/ICASSP49660.2025.10889187)

- [Robust and Imperceptible Watermarking Framework for Generative Audio Models (RIWF)](./Watermarking/generative_watermarking/2025-RIWF.md)  
  *IEEE Signal Processing Letters, 2025*  
  Citation: Feng, Y., Zhang, X., Feng, F., Zhang, G., & Xu, L. “Robust and Imperceptible Watermarking Framework for Generative Audio Models.” *IEEE Signal Processing Letters*, 32, 3196–3200, 2025.  
  Links: [Paper](https://doi.org/10.1109/LSP.2025.3596015) | [Code](https://github.com/DHUspeech/watermark-framework-for-generative-audio-models)

#### 2024

- [GROOT: Generating Robust Watermark for Diffusion-Model-Based Audio Synthesis](./Watermarking/generative_watermarking/2024-GROOT.md)  
  *ACM International Conference on Multimedia (ACM MM), 2024*  
  Citation: Liu, W., Li, Y., Lin, D., Tian, H., & Li, H. “GROOT: Generating Robust Watermark for Diffusion-Model-Based Audio Synthesis.” *Proceedings of the 32nd ACM International Conference on Multimedia*, 3294–3302, 2024.  
  Links: [Paper](https://doi.org/10.1145/3664647.3680596) | [Code](https://github.com/Groot-GAW/Groot) | [Demo](https://groot-gaw.github.io/)

- [HiFi-GANw: Watermarked Speech Synthesis via Fine-Tuning of HiFi-GAN](./Watermarking/generative_watermarking/2024-HiFi-GANw.md)  
  *IEEE Signal Processing Letters, 2024*  
  Citation: Cheng, X., Wang, Y., Liu, C., Hu, D., & Su, Z. “HiFi-GANw: Watermarked Speech Synthesis via Fine-Tuning of HiFi-GAN.” *IEEE Signal Processing Letters*, 31, 2440–2444, 2024.  
  Links: [Paper](https://doi.org/10.1109/LSP.2024.3456673)

- [TraceableSpeech: Towards Proactively Traceable Text-to-Speech with Watermarking](./Watermarking/generative_watermarking/2024-TraceableSpeech.md)  
  *Annual Conference of the International Speech Communication Association (Interspeech), 2024*  
  Citation: Zhou, J., Yi, J., Wang, T., Tao, J., Bai, Y., Zhang, C. Y., Ren, Y., & Wen, Z. “TraceableSpeech: Towards Proactively Traceable Text-to-Speech with Watermarking.” *Interspeech 2024*, 2250–2254, 2024.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2024/zhou24b_interspeech.html) | [Code](https://github.com/zjzser/TraceableSpeech)


### Post-hoc Watermarking

#### 2025

- [WMCodec: End-to-End Neural Speech Codec with Deep Watermarking for Authenticity Verification](./Watermarking/post_hoc_watermarking/2024-WMCodec.md)  
  *IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), 2025*  
  Citation: Zhou, J., Yi, J., Ren, Y., Tao, J., Wang, T., & Zhang, C. Y. “WMCodec: End-to-End Neural Speech Codec with Deep Watermarking for Authenticity Verification.” *ICASSP*, 2025.  
  Links: [Paper](https://arxiv.org/abs/2409.12121) | [Code](https://github.com/zjzser/WMCodec)

- [ASSMark: Dual Defense Against Speech Synthesis Attack via Adversarial Robust Watermarking](./Watermarking/post_hoc_watermarking/2025-ASSMark.md)  
  *IEEE Signal Processing Letters, 2025*  
  Citation: He, Y., Wang, H., Qiu, Y., & Cao, H. “ASSMark: Dual Defense Against Speech Synthesis Attack via Adversarial Robust Watermarking.” *IEEE Signal Processing Letters*, 32, 1870–1874, 2025.  
  Links: [Paper](https://doi.org/10.1109/LSP.2025.3562817)

- [AudioMarkNet: Audio Watermarking for Deepfake Speech Detection](./Watermarking/post_hoc_watermarking/2025-AudioMarkNet.md)  
  *34th USENIX Security Symposium (USENIX Security), 2025*  
  Citation: Zong, W., Chow, Y.-W., Susilo, W., Baek, J., & Camtepe, S. “AudioMarkNet: Audio Watermarking for Deepfake Speech Detection.” *34th USENIX Security Symposium*, 2025.  
  Links: [Paper](https://www.usenix.org/conference/usenixsecurity25/presentation/zong) | [Code & Artifacts](https://zenodo.org/records/14722182)

- [Audio WAtermArk: Dynamic and Harmless Watermark for Black-box Voice Dataset Copyright Protection](./Watermarking/post_hoc_watermarking/2025-Audio-WAtermArk.md)  
  *34th USENIX Security Symposium (USENIX Security), 2025*  
  Citation: Guo, H., Guo, J., Chen, B., Wang, Y., Chen, X., Huang, H., Yan, Q., & Xiao, L. “Audio Watermark: Dynamic and Harmless Watermark for Black-box Voice Dataset Copyright Protection.” *34th USENIX Security Symposium*, 2025.  
  Links: [Paper](https://www.usenix.org/conference/usenixsecurity25/presentation/guo-hanqing) | [Code & Artifacts](https://zenodo.org/records/14738544)

- [DeepAWR: An Audio Watermarking Method Against Re-recording Distortions](./Watermarking/post_hoc_watermarking/2025-DeepAWR.md)  
  *Pattern Recognition, 2025*  
  Citation: Lin, G., Luo, W., Zheng, P., & Huang, J. “An Audio Watermarking Method Against Re-recording Distortions.” *Pattern Recognition*, 162, 111366, 2025.  
  Links: [Paper](https://doi.org/10.1016/j.patcog.2025.111366) | [Code](https://github.com/gylin2/DeepAWR)

- [DiscreteWM: Speech Watermarking with Discrete Intermediate Representations](./Watermarking/post_hoc_watermarking/2025-DiscreteWM.md)  
  *AAAI Conference on Artificial Intelligence (AAAI), 2025*  
  Citation: Ji, S., Jiang, Z., Zuo, J., Fang, M., Chen, Y., Jin, T., & Zhao, Z. “Speech Watermarking with Discrete Intermediate Representations.” *Proceedings of the 39th AAAI Conference on Artificial Intelligence*, 2025.  
  Links: [Paper](https://ojs.aaai.org/index.php/AAAI/article/view/34600)

- [SyncGuard: Robust Audio Watermarking Capable of Countering Desynchronization Attacks](./Watermarking/post_hoc_watermarking/2025-SyncGuard.md)  
  *European Conference on Artificial Intelligence (ECAI), 2025*  
  Citation: Gan, Z., Hu, X., Li, S., Qian, Z., & Zhang, X. “SyncGuard: Robust Audio Watermarking Capable of Countering Desynchronization Attacks.” *ECAI 2025*, 1237–1244, 2025.  
  Links: [Paper](https://doi.org/10.3233/FAIA250937)

- [VoiceMark: Zero-Shot Voice Cloning-Resistant Watermarking Approach Leveraging Speaker-Specific Latents](./Watermarking/post_hoc_watermarking/2025-VoiceMark.md)  
  *Annual Conference of the International Speech Communication Association (Interspeech), 2025*  
  Citation: Li, H., Wu, Z., Xie, X., Xie, J., Xu, Y., & Peng, H. “VoiceMark: Zero-Shot Voice Cloning-Resistant Watermarking Approach Leveraging Speaker-Specific Latents.” *Interspeech 2025*, 2025.  
  Links: [Paper](https://arxiv.org/abs/2505.21568) | [Code & Demo](https://huggingface.co/spaces/haiyunli/VoiceMark)

- [WAKE: Watermarking Audio with Key Enrichment](./Watermarking/post_hoc_watermarking/2025-WAKE.md)  
  *Annual Conference of the International Speech Communication Association (Interspeech), 2025*  
  Citation: Xu, Y., Yu, J., Chen, H., Wu, Z., Wu, X., Yu, D., Gu, R., & Luo, Y. “WAKE: Watermarking Audio with Key Enrichment.” *Interspeech 2025*, 5093–5097, 2025.  
  Links: [Paper](https://arxiv.org/abs/2506.05891) | [Code](https://github.com/thuhcsi/WAKE)

- [WaveVerify: A Novel Audio Watermarking Framework for Media Authentication and Combatting Deepfakes](./Watermarking/post_hoc_watermarking/2025-WaveVerify.md)  
  *IEEE/IAPR International Joint Conference on Biometrics (IJCB), 2025*  
  Citation: Pujari, A., & Rattani, A. “WaveVerify: A Novel Audio Watermarking Framework for Media Authentication and Combatting Deepfakes.” *IEEE/IAPR International Joint Conference on Biometrics*, 2025.  
  Links: [Paper](https://arxiv.org/abs/2507.21150) | [Code](https://github.com/vcbsl/WaveVerify)

- [XAttnMark: Learning Robust Audio Watermarking with Cross-Attention](./Watermarking/post_hoc_watermarking/2025-XAttnMark.md)  
  *International Conference on Machine Learning (ICML), 2025*  
  Citation: Liu, Y., Lu, L., Jin, J., Sun, L., & Fanelli, A. “XAttnMark: Learning Robust Audio Watermarking with Cross-Attention.” *Proceedings of the 42nd International Conference on Machine Learning*, 2025.  
  Links: [Paper](https://arxiv.org/abs/2502.04230)

#### 2024

- [Proactive Detection of Voice Cloning with Localized Watermarking (AudioSeal)](./Watermarking/post_hoc_watermarking/2024-AudioSeal.md)  
  *International Conference on Machine Learning (ICML), 2024*  
  Citation: San Roman, R., Fernandez, P., Elsahar, H., D\'efossez, A., Furon, T., & Tran, T. “Proactive Detection of Voice Cloning with Localized Watermarking.” *Proceedings of the 41st International Conference on Machine Learning*, 2024.
  Links: [Paper](https://proceedings.mlr.press/v235/san-roman24a.html) | [Code](https://github.com/facebookresearch/audioseal)

- [DRAW: Dual-Decoder-Based Robust Audio Watermarking Against Desynchronization and Replay Attacks](./Watermarking/post_hoc_watermarking/2024-DRAW.md)  
  *IEEE Transactions on Information Forensics and Security, 2024*  
  Citation: Li, B., Chen, J., Xu, Y., Li, W., & Liu, Z. “DRAW: Dual-Decoder-Based Robust Audio Watermarking Against Desynchronization and Replay Attacks.” *IEEE Transactions on Information Forensics and Security*, 19, 6529–6544, 2024.  
  Links: [Paper](https://doi.org/10.1109/TIFS.2024.3416047)

- [IDEAW: Robust Neural Audio Watermarking with Invertible Dual-Embedding](./Watermarking/post_hoc_watermarking/2024-IDEAW.md)  
  *Conference on Empirical Methods in Natural Language Processing (EMNLP), 2024*  
  Citation: Li, P., Zhang, X., Xiao, J., & Wang, J. “IDEAW: Robust Neural Audio Watermarking with Invertible Dual-Embedding.” *Proceedings of EMNLP 2024*, 4500–4511, 2024.  
  Links: [Paper](https://aclanthology.org/2024.emnlp-main.258/) | [Code](https://github.com/PecholaL/IDEAW)

- [MaskMark: Robust Neural Watermarking for Real and Synthetic Speech](./Watermarking/post_hoc_watermarking/2024-MaskMark.md)  
  *IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), 2024*  
  Citation: O’Reilly, P., Jin, Z., Su, J., & Pardo, B. “MaskMark: Robust Neural Watermarking for Real and Synthetic Speech.” *ICASSP 2024*, 2024.  
  Links: [Paper](https://oreillyp.github.io/maskmark/)

- [SilentCipher: Deep Audio Watermarking](./Watermarking/post_hoc_watermarking/2024-SilentCipher.md)  
  *Annual Conference of the International Speech Communication Association (Interspeech), 2024*  
  Citation: Singh, M. K., Takahashi, N., Liao, W.-H., & Mitsufuji, Y. “SilentCipher: Deep Audio Watermarking.” *Interspeech 2024*, 2024.  
  Links: [Paper](https://www.isca-archive.org/interspeech_2024/singh24_interspeech.html) | [Code](https://github.com/sony/silentcipher)

- [An Imperceptible and Robust Audio Watermarking Algorithm Based on SNGAN](./Watermarking/post_hoc_watermarking/2024-SNGAN-AudioWM.md)  
  *International Joint Conference on Neural Networks (IJCNN), 2024*  
  Citation: Zhou, W., Zhou, J., & Yang, S. “An Imperceptible and Robust Audio Watermarking Algorithm Based on SNGAN.” *IJCNN 2024*, 1–8, 2024.  
  Links: [Paper](https://doi.org/10.1109/IJCNN60899.2024.10651036)

- [Detecting Voice Cloning Attacks via Timbre Watermarking](./Watermarking/post_hoc_watermarking/2024-Timbre-Watermarking.md)  
  *Network and Distributed System Security Symposium (NDSS), 2024*  
  Citation: Liu, C., Zhang, J., Zhang, T., Yang, X., Zhang, W., & Yu, N. “Detecting Voice Cloning Attacks via Timbre Watermarking.” *NDSS Symposium*, 2024.  
  Links: [Paper](https://www.ndss-symposium.org/ndss-paper/detecting-voice-cloning-attacks-via-timbre-watermarking/) | [Code](https://github.com/TimbreWatermarking/TimbreWatermarking)

- [Enhancing Robustness of Speech Watermarking Using a Transformer-Based Framework Exploiting Acoustic Features](./Watermarking/post_hoc_watermarking/2024-Transformer-Acoustic-Watermarking.md)  
  *IEEE/ACM Transactions on Audio, Speech, and Language Processing, 2024*  
  Citation: Tong, C., Natgunanathan, I., Xiang, Y., Li, J., Zong, T., Zheng, X., & Gao, L. “Enhancing Robustness of Speech Watermarking Using a Transformer-Based Framework Exploiting Acoustic Features.” *IEEE/ACM Transactions on Audio, Speech, and Language Processing*, 32, 4822–4837, 2024.  
  Links: [Paper](https://doi.org/10.1109/TASLP.2024.3486206)

#### 2023

- [AudioQR: Deep Neural Audio Watermarks for QR Code](./Watermarking/post_hoc_watermarking/2023-AudioQR.md)  
  *International Joint Conference on Artificial Intelligence (IJCAI), 2023*  
  Citation: Qu, X., Yin, X., Wei, P., Lu, L., & Ma, Z. “AudioQR: Deep Neural Audio Watermarks for QR Code.” *Proceedings of the 32nd International Joint Conference on Artificial Intelligence*, 6192–6200, 2023.  
  Links: [Paper](https://www.ijcai.org/proceedings/2023/687) | [Code](https://github.com/xinghua-qu/AudioQR)

- [WavMark: Watermarking for Audio Generation](./Watermarking/post_hoc_watermarking/2023-WavMark.md)  
  *arXiv preprint arXiv:2308.12770, 2023*  
  Citation: Chen, G., Wu, Y., Liu, S., Liu, T., Du, X., & Wei, F. “WavMark: Watermarking for Audio Generation.” *arXiv preprint arXiv:2308.12770*, 2023.  
  Links: [Paper](https://arxiv.org/abs/2308.12770) | [Code](https://github.com/wavmark/wavmark)

- [DeAR: A Deep-Learning-Based Audio Re-recording Resilient Watermarking](./Watermarking/post_hoc_watermarking/2023-DeAR.md)  
  *Proceedings of the AAAI Conference on Artificial Intelligence, 2023*  
  Citation: Liu, C., Zhang, J., Fang, H., Ma, Z., Zhang, W., & Yu, N. “DeAR: A Deep-Learning-Based Audio Re-recording Resilient Watermarking.” *Proceedings of the 37th AAAI Conference on Artificial Intelligence*, 2023.  
  Links: [Paper](https://ojs.aaai.org/index.php/AAAI/article/view/26550)

#### 2022

- [Robust Speech Watermarking by a Jointly Trained Embedder and Detector Using a DNN](./Watermarking/post_hoc_watermarking/2022-RSW-DNN.md)  
  *Digital Signal Processing, 2022*  
  Citation: Pavlović, K., Kovačević, S., Djurović, I., & Wojciechowski, A. “Robust Speech Watermarking by a Jointly Trained Embedder and Detector Using a DNN.” *Digital Signal Processing*, 122, 103381, 2022.  
  Links: [Paper](https://doi.org/10.1016/j.dsp.2021.103381) | [Code](https://github.com/kosta-pmf/dnn-audio-watermarking)

## Steganography

#### 2026

- [PRoADS: Provably Secure and Robust Audio Diffusion Steganography with Latent Optimization and Backward Euler Inversion](./Watermarking/Steganography/2026-PRoADS.md)  
  *IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), 2026*  
  Citation: Yan, Y., Li, Y., Xiao, Q., & Ren, Y. “PRoADS: Provably Secure and Robust Audio Diffusion Steganography with Latent Optimization and Backward Euler Inversion.” *Proceedings of the IEEE International Conference on Acoustics, Speech and Signal Processing*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2603.10314)

- [WavInWav: Time-domain Speech Hiding via Invertible Neural Network](./Watermarking/Steganography/2026-WavInWav.md)  
  *IEEE Transactions on Dependable and Secure Computing, 2026*  
  Citation: Fan, W., Chen, K., Wang, X., Zhang, W., & Yu, N. “WavInWav: Time-domain Speech Hiding via Invertible Neural Network.” *IEEE Transactions on Dependable and Secure Computing*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2510.02915)

#### 2025

- [HIFI-Stego: A High-Fidelity Embedding Audio Steganography Based on Audio Features Decoupling](./Watermarking/Steganography/2025-HIFI-Stego.md)  
  *IEEE Transactions on Audio, Speech and Language Processing, 2025*  
  Citation: Zhang, S., Tian, B., Gao, Y., Liu, X., & Yang, W. “HIFI-Stego: A High-Fidelity Embedding Audio Steganography Based on Audio Features Decoupling.” *IEEE Transactions on Audio, Speech and Language Processing*, vol. 33, pp. 2032–2044, 2025.  
  Links: [Paper](https://doi.org/10.1109/TASLPRO.2025.3570942) | [Code](https://github.com/emptybodys/HIFI-Stego)

- [Provably Secure and Robust Audio Steganography Under Multi-Format Low-Bitrate Compression](./Watermarking/Steganography/2025-Provably-Secure-Robust-Audio-Steganography.md)  
  *IEEE Transactions on Information Forensics and Security, 2025*  
  Citation: Li, Y., Xiao, Q., Wang, Z., Ren, Y., & Wang, L. “Provably Secure and Robust Audio Steganography Under Multi-Format Low-Bitrate Compression.” *IEEE Transactions on Information Forensics and Security*, vol. 20, pp. 12596–12608, 2025.  
  Links: [Paper](https://doi.org/10.1109/TIFS.2025.3636668)

- [A Dynamically Interactable Framework with Dual-Channel Security: GAN-Based Speech Steganography for Concealed Dialogues](./Watermarking/Steganography/2025-DialogStego.md)  
  *Knowledge-Based Systems, 2025*  
  Citation: Ge, X., Zhang, X., Li, Y., & Sun, M. “A Dynamically Interactable Framework with Dual-Channel Security: GAN-Based Speech Steganography for Concealed Dialogues.” *Knowledge-Based Systems*, vol. 330, Article 114618, 2025.  
  Links: [Paper](https://doi.org/10.1016/j.knosys.2025.114618)

- [CoAS: Composite Audio Steganography Based on Text and Speech Synthesis](./Watermarking/Steganography/2025-CoAS.md)  
  *IEEE Transactions on Information Forensics and Security, 2025*  
  Citation: Li, Y., Chen, K., Wang, Y., Zhang, X., Wang, G., Zhang, W., & Yu, N. “CoAS: Composite Audio Steganography Based on Text and Speech Synthesis.” *IEEE Transactions on Information Forensics and Security*, vol. 20, pp. 5978–5991, 2025.  
  Links: [Paper](https://doi.org/10.1109/TIFS.2025.3579581) | [Samples](https://meterial.github.io/coas.github.io)

## Adversarial Examples

### 2026

- [Attacker’s Noise Can Manipulate Your Audio-based LLM in the Real World (AN-LALM)](./Adversarial_Examples/2026-AN-LALM.md)  
  *European Chapter of the Association for Computational Linguistics (EACL), 2026*  
  Citation: Sadasivan, V. S., Feizi, S., Mathews, R., & Wang, L. “Attacker’s Noise Can Manipulate Your Audio-based LLM in the Real World.” *EACL 2026*, 2026.  
  Links: [Paper](https://aclanthology.org/2026.eacl-long.66.pdf)

- [MUSICSHIELD: Protection for Musicians in the Era of Generative AI](./Adversarial_Examples/2026-MUSICSHIELD.md)  
  *IEEE Symposium on Security and Privacy (IEEE S&P), 2026*  
  Citation: “MUSICSHIELD: Protection for Musicians in the Era of Generative AI.” *IEEE S&P 2026*.  
  Links: [Project](https://musicshield.org/)

### 2025

- [HarmonyCloak: Defending Music Against Generative AI](./Adversarial_Examples/2025-HarmonyCloak.md)  
  *IEEE Symposium on Security and Privacy (IEEE S&P), 2025*  
  Citation: “HarmonyCloak: Defending Music Against Generative AI.” *IEEE S&P 2025*.  
  Links: [Project & Paper](https://mosis.eecs.utk.edu/harmonycloak.html)

- [SafeSpeech: Robust and Universal Voice Protection Against Malicious Speech Synthesis](./Adversarial_Examples/2025-SafeSpeech.md)  
  *34th USENIX Security Symposium (USENIX Security), 2025*  
  Citation: Zhang, Z., Wang, D., Yang, Q., et al. “SafeSpeech: Robust and Universal Voice Protection Against Malicious Speech Synthesis.” *USENIX Security 2025*.  
  Links: [Paper](https://www.usenix.org/conference/usenixsecurity25/presentation/zhang-zhisheng) | [Code](https://github.com/wxzyd123/SafeSpeech)

- [Adversarial Attacks and Robust Defenses in Speaker Embedding based Zero-Shot Text-to-Speech System (SEAE)](./Adversarial_Examples/2025-SEAE.md)  
  *IEEE International Conference on Multimedia and Expo (ICME), 2025*  
  Citation: Li, Z., Shi, Y., Xu, Y., & Li, M. “Adversarial Attacks and Robust Defenses in Speaker Embedding based Zero-Shot Text-to-Speech System.” *ICME 2025*, 2025.  
  Links: [Paper](https://doi.org/10.1109/ICME59968.2025.11210164)
- [Universal Acoustic Adversarial Attacks for Flexible Control of Speech-LLMs (UAAA-SLLM)](./Adversarial_Examples/2025-UAAA-SLLM.md)  
  *Findings of the Association for Computational Linguistics: EMNLP, 2025*  
  Citation: Ma, R., Qian, M., Raina, V., Gales, M., & Knill, K. “Universal Acoustic Adversarial Attacks for Flexible Control of Speech-LLMs.” *Findings of EMNLP 2025*, 2025.  
  Links: [Paper](https://aclanthology.org/2025.findings-emnlp.990.pdf)

### 2024

- [Adversarial Speech for Voice Privacy Protection from Personalized Speech Generation (AS-PSG)](./Adversarial_Examples/2024-AS-PSG.md)  
  *IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP), 2024*  
  Citation: Chen, S., Chen, L., Zhang, J., Lee, K. A., Ling, Z., & Dai, L. “Adversarial Speech for Voice Privacy Protection from Personalized Speech Generation.” *ICASSP 2024*, 2024.  
  Links: [Paper](https://arxiv.org/abs/2401.11857) | [Audio Demo](https://voiceprivacy.github.io/Adeversarial-Speech-with-YourTTS)

### 2023

- [VSMask: Defending Against Voice Synthesis Attack via Real-Time Predictive Perturbation](./Adversarial_Examples/2023-VSMask.md)  
  *ACM Conference on Security and Privacy in Wireless and Mobile Networks (WiSec), 2023*  
  Citation: Wang, Y., Guo, H., Wang, G., Chen, B., & Yan, Q. “VSMask: Defending Against Voice Synthesis Attack via Real-Time Predictive Perturbation.” *WiSec 2023*, 2023.  
  Links: [Paper](https://doi.org/10.1145/3558482.3590189)

### 2021

- [Defending Your Voice: Adversarial Attack on Voice Conversion (HIN)](./Adversarial_Examples/2021-HIN.md)  
  *IEEE Spoken Language Technology Workshop (SLT), 2021*  
  Citation: Huang, C.-Y., Lin, Y. Y., Lee, H.-Y., & Lee, L.-S. “Defending Your Voice: Adversarial Attack on Voice Conversion.” *SLT 2021*, 552–559, 2021.  
  Links: [Paper](https://doi.org/10.1109/SLT48900.2021.9383529) | [Code](https://github.com/cyhuang-tw/attack-vc)


## Jailbreak

### 2026

- [StyleBreak: Revealing Alignment Vulnerabilities in Large Audio-Language Models via Style-Aware Audio Jailbreak](./Jailbreak/2026-StyleBreak.md)  
  *AAAI Conference on Artificial Intelligence (AAAI), 2026*  
  Citation: Li, H., Zhou, C., Wang, C., et al. “StyleBreak: Revealing Alignment Vulnerabilities in Large Audio-Language Models via Style-Aware Audio Jailbreak.” *AAAI 2026*, 2026.  
  Links: [Paper](https://doi.org/10.1609/aaai.v40i44.41093)
- [AudioJailbreak: Jailbreak Attacks Against End-to-End Large Audio-Language Models](./Jailbreak/2026-AudioJailbreak.md)  
  *IEEE Transactions on Dependable and Secure Computing, 2026*  
  Citation: Chen, G., Song, F., Zhao, Z., et al. “AudioJailbreak: Jailbreak Attacks Against End-to-End Large Audio-Language Models.” *IEEE Transactions on Dependable and Secure Computing*, 23(3), 2026.  
  Links: [Paper](https://arxiv.org/abs/2505.14103) | [Code & Audio Samples](https://audiojailbreak.github.io/AudioJailbreak)

- [Acoustic Interference: A New Paradigm Weaponizing Acoustic Latent Semantic for Universal Jailbreak against Large Audio Language Models (AIA)](./Jailbreak/2026-AIA.md)  
  *International Conference on Machine Learning (ICML), 2026*  
  Citation: Wang, Y., Huang, Y., Liang, Z., Wu, X., & Liu, L. “Acoustic Interference: A New Paradigm Weaponizing Acoustic Latent Semantic for Universal Jailbreak against Large Audio Language Models.” *ICML 2026*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2605.18168) | [Code & ALS Arsenal](https://flaai.github.io/AIA_page)

- [Audio Jailbreak: An Open Comprehensive Benchmark for Jailbreaking Large Audio-Language Models (AJailBench)](./Jailbreak/2026-AJailBench.md)  
  *Annual Meeting of the Association for Computational Linguistics (ACL), 2026*  
  Citation: Song, Z., Jiang, Q., Cui, M., et al. “Audio Jailbreak: An Open Comprehensive Benchmark for Jailbreaking Large Audio-Language Models.” *ACL 2026*, 2026.  
  Links: [Paper](https://aclanthology.org/2026.acl-long.1259.pdf) | [Code & Data](https://github.com/PbRQianJiang/AudioJailbreak)

- [JALMBench: Benchmarking Jailbreak Vulnerabilities in Audio Language Models](./Jailbreak/2025-JALMBench.md)  
  *International Conference on Learning Representations (ICLR), 2026*  
  Citation: Peng, Z., Liu, Y., Sun, Z., et al. “JALMBench: Benchmarking Jailbreak Vulnerabilities in Audio Language Models.” *ICLR 2026*, 2026.  
  Links: [Paper](https://openreview.net/forum?id=ZMQaoaF5tQ) | [Code](https://github.com/sfofgalaxy/JALMBench)

### 2025

- [AdvWave: Stealthy Adversarial Jailbreak Attack against Large Audio-Language Models](./Jailbreak/2025-AdvWave.md)  
  *International Conference on Learning Representations (ICLR), 2025*  
  Citation: Kang, M., Xu, C., & Li, B. “AdvWave: Stealthy Adversarial Jailbreak Attack against Large Audio-Language Models.” *ICLR 2025*, 2025.  
  Links: [Paper](https://arxiv.org/abs/2412.08608) | [Audio Demo](https://violademo.github.io/) | Code: Not publicly available

- [Who Can Withstand Chat-Audio Attacks? An Evaluation Benchmark for Large Audio-Language Models (CAA)](./Jailbreak/2025-CAA.md)  
  *Findings of the Association for Computational Linguistics: ACL, 2025*  
  Citation: Yang, W., Li, Y., Fang, M., Wei, Y., & Chen, L. “Who Can Withstand Chat-Audio Attacks? An Evaluation Benchmark for Large Audio-Language Models.” *Findings of ACL 2025*, 2025.  
  Links: [Paper](https://aclanthology.org/2025.findings-acl.884.pdf)

- [Jailbreak-AudioBench: In-Depth Evaluation and Analysis of Jailbreak Threats for Large Audio Language Models](./Jailbreak/2025-Jailbreak-AudioBench.md)  
  *Conference on Neural Information Processing Systems (NeurIPS), Datasets and Benchmarks Track, 2025*  
  Citation: Cheng, H., Xiao, E., Shao, J., et al. “Jailbreak-AudioBench: In-Depth Evaluation and Analysis of Jailbreak Threats for Large Audio Language Models.” *NeurIPS 2025 Datasets and Benchmarks Track*, 2025.  
  Links: [Paper](https://papers.nips.cc/paper_files/paper/2025/file/0ff38d72a2e0aa6dbe42de83a17b2223-Paper-Datasets_and_Benchmarks_Track.pdf) | [Project](https://researchtopic.github.io/Jailbreak-AudioBench_Page/)

- [Audio Is the Achilles’ Heel: Red Teaming Audio Large Multimodal Models (RT-LALM)](./Jailbreak/2025-RT-LALM.md)  
  *NAACL Human Language Technologies, 2025*  
  Citation: Yang, H., Qu, L., Shareghi, E., & Haffari, G. “Audio Is the Achilles’ Heel: Red Teaming Audio Large Multimodal Models.” *NAACL 2025*, 2025.  
  Links: [Paper](https://aclanthology.org/2025.naacl-long.468.pdf) 

## Other Security

### 2026

- [PathMark: Protecting Intellectual Property of Mixture-of-Expert LLMs via Path Watermarks](./Watermarking/model_watermarking/2026-PathMark.md)  
  *ACM SIGSAC Conference on Computer and Communications Security (CCS), 2026*  
  Citation: Gao, Y., Wang, Q., Yuan, Y., Huang, R., Chen, L., Ji, Z., & Wang, S. “PathMark: Protecting Intellectual Property of Mixture-of-Expert LLMs via Path Watermarks.” *Proceedings of the 33rd ACM SIGSAC Conference on Computer and Communications Security*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2607.03688) | [Code](https://github.com/ifen1/PathMark_in) 

- [WRATH: Turning Watermark Robustness Against Itself via a Watermark-Agnostic Black-Box Invalidation Attack](./Other_Security/2026-WRATH.md)  
  *IEEE Symposium on Security and Privacy (S&P), 2026*  
  Citation: Jiang, N., Hu, J., Sun, B., Sim, T., & Han, J. “WRATH: Turning Watermark Robustness Against Itself via a Watermark-Agnostic Black-Box Invalidation Attack.” *2026 IEEE Symposium on Security and Privacy (S&P)*, pp. 379–397, 2026.  
  Links: [Paper](https://doi.org/10.1109/SP63933.2026.00197)

- [A Distortion-minimization Watermarking Framework for Large Language Models: Larger Capacity, Stronger Robustness and Higher Quality](./Other_Security/2026-DMW.md)  
  *USENIX Security Symposium, 2026*  
  Citation: Zhai, L., Shang, X., Zhang, L., & Hu, P. “A Distortion-minimization Watermarking Framework for Large Language Models: Larger Capacity, Stronger Robustness and Higher Quality.” *USENIX Security Symposium*, 2026.  
  Links: [Paper](https://www.usenix.org/conference/usenixsecurity26/presentation/zhai) 

- [AgentMark: Utility-Preserving Behavioral Watermarking for Agents](./Other_Security/2026-AgentMark.md)  
  *Annual Meeting of the Association for Computational Linguistics (ACL), 2026*  
  Citation: Huang, K., Tan, J., Wei, Y., Li, W., Zhang, Z., Tian, H., Yang, Z., & Zhou, L. “AgentMark: Utility-Preserving Behavioral Watermarking for Agents.” *Proceedings of the 64th Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)*, 2026.  
  Links: [Paper](https://aclanthology.org/2026.acl-long.573/) | [Code](https://github.com/Tooooa/AgentMark)

- [Watermarking LLM Agent Trajectories (ACTHOOK)](./Other_Security/2026-ACTHOOK.md)  
  *International Conference on Machine Learning (ICML), 2026*  
  Citation: Meng, W., Gong, C., Zhuo, T. Y., Zhang, F., Li, K., Liu, Z., Yang, Z., Wei, C., & Chen, W. “Watermarking LLM Agent Trajectories.” *Proceedings of the 43rd International Conference on Machine Learning*, 2026.  
  Links: [Paper](https://arxiv.org/pdf/2602.18700) | [Code](https://github.com/meng-wenlong/AgentWmk)

- [Learning to Watermark in the Latent Space of Generative Models (DistSeal)](./Other_Security/2026-DistSeal.md)  
  *International Conference on Machine Learning (ICML), 2026*   
  Citation: Rebuffi, S.-A., Tran, T., Lacatusu, V., Fernandez, P., Souček, T., Jovanović, N., Sander, T., Elsahar, H., & Mourachko, A. “Learning to Watermark in the Latent Space of Generative Models.” *arXiv preprint arXiv:2601.16140*, 2026.  
  Links: [Paper](https://arxiv.org/abs/2601.16140) | [Code](https://github.com/facebookresearch/distseal)

- [SLIM: Stable Latent Integration for Robust Watermark in Diffusion Model](./Other_Security/2026-SLIM.md)  
  *IEEE Transactions on Circuits and Systems for Video Technology, 2026*  
  Citation: Kong, X., Chen, P., Li, B., Yuan, J., Cai, Z., Wu, H., & Liang, L. “SLIM: Stable Latent Integration for Robust Watermark in Diffusion Model.” *IEEE Transactions on Circuits and Systems for Video Technology*, 2026.  
  Links: [Paper](https://doi.org/10.1109/TCSVT.2026.3676184) | [Code](https://github.com/XiaoxiKong/SLIM)
  

### 2025

- [Watermarking Autoregressive Image Generation](./Other_Security/2025-WMAR.md")  
  *Advances in Neural Information Processing Systems (NeurIPS), 2025*  
  Citation: Jovanović, N., Labiad, I., Souček, T., Vechev, M., & Fernandez, P. “Watermarking Autoregressive Image Generation.” *Advances in Neural Information Processing Systems*, vol. 38, 2025.  
  Links: [Paper](https://arxiv.org/abs/2506.16349) | [Code](https://github.com/facebookresearch/wmar)

- [Robust Watermarking Using Generative Priors Against Image Editing: From Benchmarking to Advances (VINE)](./Other_Security/2025-VINE.md)  
  *International Conference on Learning Representations (ICLR), 2025*  
  Citation: Lu, S., Zhou, Z., Lu, J., Zhu, Y., & Kong, A. W.-K. “Robust Watermarking Using Generative Priors Against Image Editing: From Benchmarking to Advances.” *International Conference on Learning Representations*, 2025.  
  Links: [Paper](https://openreview.net/forum?id=OHp20fyvs2) | [Code](https://github.com/Shilin-LU/VINE)

### 2024

- [AquaLoRA: Toward White-box Protection for Customized Stable Diffusion Models via Watermark LoRA](./Other_Security/2024-AquaLoRA.md)  
  *International Conference on Machine Learning (ICML), 2024*  
  Citation: Feng, W., Zhou, W., He, J., Zhang, J., Wei, T., Li, G., Zhang, T., Zhang, W., & Yu, N. “AquaLoRA: Toward White-box Protection for Customized Stable Diffusion Models via Watermark LoRA.” *Proceedings of the 41st International Conference on Machine Learning*, 2024.  
  Links: [Paper](https://proceedings.mlr.press/v235/feng24d.html) | [Code](https://github.com/Georgefwt/AquaLoRA)

## Voice Agent
