# MNE 推荐预处理流程

MNE 没有适用于全部 EEG 数据集的单一固定参数流程。此处按照 MNE 官方预处理顺序组织运动想象任务的默认实现；所有由本流程给出的任务默认值均须在报告中标明，不能写成数据集原有参数。

## 目录

- 操作流程
- 核心代码顺序
- Bad-channel metric 图
- ICA 图
- 其他 MNE 作图与判据
- 来源

## 操作流程

### 1. 读取与校验

1. 使用对应的 `mne.io.read_raw_*()` 读取原始文件，保留连续数据、注释和触发器。
2. 使用 `raw.set_channel_types()` 校正 EEG、EOG、ECG、EMG、Stim 等通道类型，使用 `raw.set_montage()` 设置真实电极布局。
3. 核对数据单位、采样率、通道顺序、记录时长和事件样点；发现单位或事件错误时先纠正并记录，再进入预处理。
4. 用 `mne.events_from_annotations()` 或数据格式对应接口提取事件，并与实验 Trigger 表逐项核对。

### 2. 原始质量检查与坏段标注

1. 使用 `raw.plot()` 查看连续波形，使用 `raw.compute_psd().plot()` 查看全频 PSD，使用 `raw.plot_sensors()` 检查电极位置。
2. 检查平坦、异常高方差、持续工频、断联、桥接和瞬时爆发通道；把确认的坏道写入 `raw.info["bads"]`。
3. 可使用 `mne.preprocessing.find_bad_channels_lof()` 提供坏道候选，但必须结合波形、PSD 和邻近通道关系确认，不以单一算法结果直接删除通道。
4. 使用 `mne.preprocessing.annotate_amplitude()`、`annotate_muscle_zscore()` 或人工检查标注坏时段；保留每段起止时间、原因和持续时间。

### 3. 滤波与采样率

1. 在连续 Raw 上滤波，以减少 Epoch 边缘效应。
2. 根据元数据和原始 PSD 确认工频，仅在存在对应噪声时用 `raw.notch_filter()` 处理 50 Hz、60 Hz 及落在分析频率内的谐波。
3. 运动想象分析副本默认使用 `raw.copy().filter(l_freq=0.5, h_freq=40.0)`，以覆盖 μ、β 及其邻近频率；分析目标超过 40 Hz、需要保留更慢活动或 40 Hz 不低于奈奎斯特频率时，根据目标与采样率修改上下限并记录。
4. ICA 拟合副本单独使用至少 1 Hz 高通，即 `raw.copy().filter(l_freq=1.0, h_freq=None)`；ICA 解应用到第 3 项的分析副本。
5. 需要降采样时，在低通和陷波之后调用 `raw.resample()`，并再次核对事件样点。

### 4. 参考与数据秩

1. 遵循数据集明确规定的参考方式；数据集没有规定时，默认用 `raw.set_eeg_reference("average")` 做平均参考。
2. 计算平均参考时排除已标记坏道，并记录参考前后数据秩。
3. ICA 前不插值坏道。平均参考使秩减少 1；已有投影、插值或其他降秩操作时，使用 `mne.compute_rank()` 确认可用秩，并据此确定 ICA 成分数。

### 5. ICA 伪迹处理

1. 在 1 Hz 高通副本上拟合 `mne.preprocessing.ICA()`，排除坏道和 `BAD_*` 注释时段，并设置固定 `random_state`。
2. 默认优先使用 Extended Infomax：`method="infomax", fit_params={"extended": True}`；如使用 PICARD 或 FastICA，记录算法、版本、收敛状态和成分数。
3. 有 EOG、ECG 通道时分别运行 `ica.find_bads_eog()`、`ica.find_bads_ecg()`；需要检查肌电成分时运行 `ica.find_bads_muscle()`。
4. 自动结果只作为候选。结合成分 Topography、时间序列、PSD、与 EOG/ECG 的得分及任务事件锁定特征确认排除成分。
5. 将确认后的 `ica.exclude` 应用到分析副本；保存 ICA 对象、排除列表、判据和处理前后叠加结果。

### 6. 坏道插值

1. ICA 完成后使用 `raw.interpolate_bads()` 插值确认坏道。
2. 同时保留插值前坏道列表，不以插值后的通道数掩盖原始坏道数量。
3. 用相邻通道波形、PSD 和 Topography 检查插值结果；插值明显失真时保留坏道标记并报告。

### 7. Epoch、基线与坏 Trial

1. 根据完整实验流程使用 `mne.Epochs()` 构造运动想象 Trial，设置 `reject_by_annotation=True`。
2. `tmin`、`tmax` 和事件映射必须来自真实范式；基线区间仅在范式提供有效基线时设置。
3. 峰峰值剔除阈值由当前数据的幅值分布和单位确定，不使用跨数据集固定阈值。把最终阈值传给 `reject` 或用 `epochs.drop_bad()` 执行。
4. 使用 `epochs.plot_drop_log()` 核查逐 Trial 原因，并统计各被试、各条件的总数、剔除数和保留数。

### 8. 保存与复现

保存清理后的 Raw、Epochs、ICA 对象、坏道列表、Annotations、事件映射、处理参数、软件版本和随机种子。BIDS 数据优先使用 MNE-BIDS-Pipeline 配置文件执行，并保留配置与自动生成的报告。

## 核心代码顺序

```python
raw = read_raw_function(input_path, preload=True)
raw.set_montage(montage)
events, event_id = mne.events_from_annotations(raw)

raw.info["bads"] = confirmed_bad_channels
raw.set_annotations(raw.annotations + bad_annotations)
raw.notch_filter(freqs=line_frequencies)
raw.set_eeg_reference(reference_channels)

raw_analysis = raw.copy().filter(l_freq=0.5, h_freq=40.0)
raw_ica = raw.copy().filter(l_freq=1.0, h_freq=None)
rank = mne.compute_rank(raw_ica)

ica = mne.preprocessing.ICA(
    method="infomax",
    fit_params={"extended": True},
    n_components=rank["eeg"],
    random_state=fixed_seed,
)
ica.fit(raw_ica, reject_by_annotation=True)
ica.exclude = confirmed_artifact_components
ica.apply(raw_analysis)
raw_analysis.interpolate_bads(reset_bads=False)

epochs = mne.Epochs(
    raw_analysis,
    events,
    event_id,
    tmin=trial_tmin,
    tmax=trial_tmax,
    baseline=baseline,
    reject=reject_thresholds,
    reject_by_annotation=True,
    preload=True,
)
```

代码中的 `read_raw_function`、`montage`、`line_frequencies`、`reference_channels`、Trial 边界、基线和剔除阈值必须根据当前数据集填写；无指定参考时令 `reference_channels="average"`。

## Bad-channel metric 图

1. 绘制标注坏道和坏时段的原始波形、全频 PSD 与电极位置图。
2. 绘制坏道 metric 图，并显示每个通道被判为候选或保留的依据及对应阈值。
3. 绘制坏道插值前后 Topography 和代表性通道波形。
4. 报告坏道数、坏道比例、坏段时长及插值通道数。

## ICA 图

1. 绘制 `plot_components()`、`plot_properties()`、`plot_scores()` 和 `plot_sources()` 结果。
2. 使用 `plot_overlay()` 展示确认排除的成分在处理前后的影响。
3. 报告 ICA 算法、数据秩、成分总数、排除成分、判据和收敛状态。

## 其他 MNE 作图与判据

1. 绘制滤波前后 PSD；横轴、纵轴和频率范围保持一致。
2. 绘制 `epochs.plot_drop_log()`、逐条件 Trial 保留率及最终各条件试次数。
3. 绘制原始与 MNE 后的波形、PSD、分频带 Topo；比较图使用相同时间范围、频率范围、色标和纵向振幅量表。
4. 使用 `mne.Report` 汇总上述图和参数。

判据依次为：文件与事件正确；坏道和坏段有可追溯证据；ICA 候选同时满足空间、时间、频谱或生理参考证据；清理后伪迹下降但 μ/β 感觉运动节律没有被整体抹除；各条件 Trial 保留率可接受且无明显条件偏倚。

## 来源

- [MNE 预处理教程](https://mne.tools/stable/auto_tutorials/preprocessing/index.html)
- [MNE 坏道处理](https://mne.tools/stable/auto_tutorials/preprocessing/15_handling_bad_channels.html)
- [MNE ICA 文档](https://mne.tools/stable/generated/mne.preprocessing.ICA.html)
- [MNE ICA 操作教程](https://mne.tools/stable/auto_tutorials/preprocessing/40_artifact_correction_ica.html)
- [MNE-BIDS-Pipeline 处理步骤](https://mne.tools/mne-bids-pipeline/stable/features/steps.html)
