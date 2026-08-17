# FAAR 评估流程

对应论文：*From EEG Cleaning to Decoding: The Role of Artifact Rejection in MI-based BCIs*。

## 操作流程

1. 使用 MNE 将原始 EEG 带通滤波至 8–32 Hz，并依据运动想象事件构造 Epochs。
2. 对每个 Epoch、每个通道计算五类特征：8–32 Hz 平均频谱幅值、RMS、最大时间梯度、过零率和峰度。频谱特征使用 MNE 的 `Epochs.compute_psd()` 或 `mne.time_frequency.psd_array_welch()` 计算。
3. 将连续数据切为 1 s 短窗，计算逐通道 RMS 并在通道内标准化；使用 truncated-Gaussian/inlier criterion 选取相对干净的参考窗，建立各特征的参考分布。
4. 将 Epoch 特征相对参考分布标准化并离散为异常严重等级，再将通道级分数聚合为 Epoch 级 SQI。SQI 越大表示质量越差。
5. 使用 Kneeliverse 对每段 Recording 的 SQI 经验分布执行 knee-point detection，以膝点作为自适应阈值，剔除阈值以上的 Epoch。记录实现、参数和版本。
6. 统计每名被试、每个 Session 的剔除 Epoch 数、保留 Epoch 数及剔除率。需要解码验证时，固定下游流程为 covariance estimation → tangent-space projection → logistic regression，并比较处理前后的 Balanced Accuracy、F1、Precision 和跨被试稳定性。

## 作图

- SQI 直方图或密度图，标出 knee threshold，并区分保留与剔除 Epoch。
- 被试与 Session 级 Epoch 剔除率及保留率图。
- 执行解码验证时，绘制处理前后指标对比、逐被试散点图和跨被试变异性图。
