# Multi-day high-quality MI 预处理与技术验证流程

对应论文：*A multi-day and high-quality EEG dataset for motor imagery brain-computer interface*。

## 操作流程

1. 数据提供采集阻抗时，检查各通道阻抗，并以尽量低于 5 kΩ 为原论文采集判据。
2. 去除 ECG、EOG 等非 EEG 通道，以 Pz 重参考，执行 0.5–40 Hz FIR 带通滤波和 50 Hz 抑制；按 Cue 截取 4 s 运动想象 Epoch，执行基线校正，再从 1000 Hz 降采样至 250 Hz。使用 MNE 的通道选择、`set_eeg_reference()`、`filter()`、`notch_filter()`、`Epochs` 和 `resample()` 完成并记录参数。
3. 不额外加入该论文未报告的 ICA、自动坏道检测、坏道插值或自动坏 Trial 剔除；如因当前数据需要增加，必须作为偏离原方法的独立步骤报告。
4. 在 8–12 Hz 计算 ERD/ERS：带通滤波 → 每个采样点平方 → 同类 Trial 平均 → 滑动窗平滑，并计算 `(A-R)/R × 100%`。左、右手任务检查 C3/C4 的对侧感觉运动调制，足部任务检查 Cz 附近活动。
5. 使用 STFT 和运动想象前基线计算 ERSP，检查约 8–30 Hz 的 μ/β 时频变化。
6. 计算 CSP 并绘制空间模式或 Topography，检查感觉运动区及对侧空间组织。需要解码验证时，使用交叉验证比较 CSP+SVM、FBCSP+SVM、EEGNet、DeepConvNet、FBCNet 等方法；分类结果仅作为间接质量证据。

## 作图

- 数据提供阻抗时，绘制逐电极阻抗图，并标出参考电极与接地。
- 2-class 或 3-class 条件下的 ERD/ERS 曲线、ERSP 时频图和 CSP Topography 组合图。
- 不同分类方法的准确率及 Chance-level 参考线。
- 存在多 Session 数据时，绘制同一被试跨 Session 的分类表现散点图。
