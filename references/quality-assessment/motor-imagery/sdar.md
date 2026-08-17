# SD-AR 预处理与评估流程

对应论文：*Subject-Dependent Artifact Removal for Enhancing Motor Imagery Classifier Performance under Poor Skills*。

## 操作流程

1. 为每名被试建立 Raw、ICA、Surface Laplacian、ICA + Surface Laplacian 四条处理分支。
2. ICA 分支先执行 1 Hz、5 阶 Butterworth 高通滤波，再白化并运行 FastICA。使用 MNE 的 `mne.preprocessing.ICA(method="fastica")`；滤波时显式记录 IIR 阶数与 Butterworth 参数。
3. 有独立 EOG 时，计算每个 IC 与 EOG 的 Pearson 相关；无独立 EOG 时，使用 Fp1、Fpz、Fp2 作为眼动代理。将相关系数转换为 z-score，把超出 ±3σ 的 IC 标为眼动伪迹候选，排除后重构 EEG。
4. Surface Laplacian 分支使用 MNE 的 `mne.preprocessing.compute_current_source_density()`，比较处理前后以 Cz 为中心的空间扩散与功能连接。
5. 将各分支划分为 μ、低 β、中 β、高 β 等运动想象相关频带，计算 Pearson correlation、motif、Gaussian FC、spectral coherence 和 phase-locking value；将连接矩阵上三角向量化后输入 LDA。
6. 按每名被试的下游分类表现选择最优分支，同时结合伪迹减少、空间局部化和感觉运动区生理合理性解释判据，不单独以准确率认定数据质量。

## 作图

- ICA 质量面板：IC 空间权重、Scalp topography、时间序列、频谱及 IC–EOG/额区代理相关 z-score 分布，并标出 ±3σ 阈值。
- Raw 与 Surface Laplacian 的功能连接矩阵及头皮连接图并排对比。
- 逐被试、逐 Session 的 noisy-trial 比例图。
- Raw、固定预处理和 SD-AR 的逐被试分类表现对比。
- μ、β 频带下的 LDA 空间相关性、Scalp relevance 或功能连接图，并提供代表性被试的处理前后对比。
