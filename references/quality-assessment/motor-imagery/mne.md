# MNE 推荐预处理流程

MNE 没有适用于全部 EEG 数据集的单一固定参数流程。此处按照 MNE 官方预处理顺序组织运动想象任务的默认实现；所有由本流程给出的任务默认值均须在报告中标明，不能写成数据集原有参数。

## 执行范围

1. 预处理范围包括排除后的全部被试。
2. 全部被试除作图外使用同一操作流程。
3. 示例被试只用于完整 ICA 诊断图；Bad-channel 和 IC 删除数量均以全部成功完成该流程的被试为统计单位。

## 操作流程

### 1. 读取与校验

1. 使用对应的 `mne.io.read_raw_*()` 读取原始文件，保留连续数据、注释和触发器。
2. 使用 `raw.set_channel_types()` 校正 EEG、EOG、ECG、EMG、Stim 等通道类型，使用 `raw.set_montage()` 设置真实电极布局。
3. 核对数据单位、采样率、通道顺序、记录时长和事件样点；发现单位或事件错误时先纠正并记录，再进入预处理。
4. 用 `mne.events_from_annotations()` 或数据格式对应接口提取事件，并与实验 Trigger 表逐项核对。

### 2. 原始质量检查与坏段标注

1. 使用 `raw.plot()` 查看连续波形，使用 `raw.compute_psd().plot()` 查看全频 PSD，使用 `raw.plot_sensors()` 检查电极位置。
2. 建立仅用于坏道检测的 1–40 Hz QC 副本，不重参考、不插值；全部被试采用相同滤波和采样率。在该副本上调用 `mne.preprocessing.find_bad_channels_lof(return_scores=True)`，保存全部通道的 LOF score；`n_neighbors`、`metric` 和 `threshold` 在数据集内固定。
3. 在未滤波原始数据上调用 `mne.preprocessing.annotate_amplitude()`，保存每个通道的 `BAD_flat`、`BAD_peak` 时长比例及返回的坏道；`peak`、`flat`、`bad_percent` 和 `min_duration` 在数据集内固定，幅度参数以伏特记录。
4. 在陷波前原始数据上计算工频功率比：`工频 ±1 Hz 平均功率 / 两侧 2 Hz 参考频带平均功率`，两侧参考频带为 `(工频-4)–(工频-2) Hz` 与 `(工频+2)–(工频+4) Hz`。使用 `mne.channels.find_ch_adjacency()` 确定空间邻居，在 QC 副本中排除 `BAD_*` 时段后计算每个通道与相邻通道相关系数的中位数。二者用于解释候选坏道，除非预先规定阈值，否则不单独触发删除。
5. 根据固定规则合并 LOF、幅度/平坦检测及波形、PSD、邻近相关证据，生成最终坏道列表并写入 `raw.info["bads"]`。人工改动必须逐通道记录原自动判定、改动后判定和理由。
6. 使用 `annotate_muscle_zscore()` 或人工检查补充坏时段；保留每段起止时间、原因和持续时间。

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
3. 有 EOG、ECG 通道时分别运行 `ica.find_bads_eog()`、`ica.find_bads_ecg()`；需要检查肌电成分时运行 `ica.find_bads_muscle()`。保存每种方法返回的完整 score、候选 IC、阈值和最终采用的 IC。
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

raw_qc = raw.copy().filter(l_freq=1.0, h_freq=40.0).resample(qc_sfreq)
lof_bads, lof_scores = mne.preprocessing.find_bad_channels_lof(
    raw_qc,
    n_neighbors=lof_n_neighbors,
    threshold=lof_threshold,
    return_scores=True,
)
amplitude_annotations, amplitude_bads = mne.preprocessing.annotate_amplitude(
    raw,
    peak=peak_thresholds,
    flat=flat_thresholds,
    bad_percent=bad_percent,
    min_duration=min_duration,
)
raw.info["bads"] = final_bad_channels
raw.set_annotations(raw.annotations + amplitude_annotations + other_bad_annotations)
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

代码中的 `read_raw_function`、`montage`、`qc_sfreq`、坏道参数、`final_bad_channels`、`other_bad_annotations`、`line_frequencies`、`reference_channels`、Trial 边界、基线和剔除阈值必须根据当前数据集填写；无指定参考时令 `reference_channels="average"`。

## 输出流程

输出内容只包含此流程中提及的内容，其他内容在附录或聊天中报告。

### Bad-channel metric 图

#### 数据整理

对全部纳入被试保存被试 × 通道级数据：通道是否实际存在、LOF score、`BAD_flat` 时长比例、`BAD_peak` 时长比例、工频功率比、邻近通道中位相关、各自动检测结果、最终坏道状态及人工改动理由。

#### 输出的图

1. **被试 × 通道 metric 热图**：按固定顺序分别绘制 LOF score、`BAD_flat` 比例、`BAD_peak` 比例、工频功率比和邻近通道中位相关。每项 metric 单独使用固定色标，并在图注写明单位、方向和阈值；未采集或无法计算显示为灰色，不得填 0。
2. **最终坏道矩阵**：被试 × 通道矩阵至少区分保留、LOF 判坏、幅度/平坦判坏、多指标判坏、人工加入、人工保留和未采集。不得只展示被删除通道。
3. **逐通道数据集统计图**：对每个通道计算 `坏道被试数 / 实际含该通道且完成处理的被试数`，绘制坏道率及二项比例的 95% Wilson 置信区间，同时标注分子和分母。通道顺序与热图一致，不按坏道率重新排序。
4. **坏道率 Topography**：使用 `mne.viz.plot_topomap()` 绘制逐通道坏道率，色标固定为 0–1。该图只作为空间摘要，统计解释以第 3 项为准。
5. **逐被试坏道负担图**：每名被试显示坏道数、坏道比例、坏段时长比例和插值通道数，并叠加全体被试的中位数与 IQR。所有被试显示为独立点，不用只有箱线图的形式替代。
6. **示例证据图**：仅对已确认的示例被试，绘制标注坏道和坏时段的 `raw.plot()`、全频 PSD、电极位置及插值前后代表性波形；这些图用于解释，不替代前五项全数据集统计图。

所有图标题写明纳入被试数、成功处理被试数、标准通道数、MNE 版本和坏道参数集标识。连续 metric 的色标范围从全体纳入被试一次性确定并冻结，不按被试单独缩放；相同数据集的不同预处理分支使用相同被试顺序、通道顺序、画布尺寸和色标。

### ICA 图

#### 示例被试的完整 ICA 诊断图

对调用前确认的每个示例被试固定输出：

1. **全部 IC 总览**：使用 `ICA.plot_components()` 绘制全部 IC 的 Topography 网格；每个 IC 显示编号，以固定边框或标题颜色区分保留、EOG、ECG、肌电、其他人工排除和多原因排除，不得只画被排除 IC。
2. **自动判据总览**：对 EOG、ECG、肌电分别绘制 `ICA.plot_scores()`，显示全部 IC 的 score、检测阈值和最终排除状态。缺少相应生理通道或无法计算时明确标为不可用，不自行构造替代分数。
3. **每个排除 IC 的五联图**：使用 `ICA.plot_properties()`，完整保留 Topography、Epoch image、平均 IC 波形及置信区间、PSD、逐 Epoch 方差五部分。所有排除 IC 均需绘制；数量过多时分页，不得只挑代表性 IC。
4. **时序证据图**：使用 `ICA.plot_sources()` 展示排除 IC 在整段记录和真实 Trial 内的活动，并标出用于判断的 EOG、ECG、肌电或其他伪迹时段。
5. **处理效果图**：使用 `ICA.plot_overlay()` 在同一真实伪迹时窗展示处理前后通道波形和 EEG GFP。时间范围、通道、纵轴和颜色在示例被试之间保持一致；各实际存在的伪迹类型至少展示一个命中时段。

#### 全数据集 IC 删除统计

1. 在报告中提供逐被试表，字段固定为：被试编号、进入 ICA 的 EEG 通道数、数据秩、IC 总数、EOG IC 数、ECG IC 数、肌电 IC 数、其他人工排除 IC 数、多原因 IC 数、唯一删除 IC 数、删除比例、ICA 算法和收敛状态。
2. 唯一删除 IC 数按 IC 编号去重，不能把不同原因的数量直接相加；删除比例使用 `唯一删除 IC 数 / IC 总数`。
3. 绘制逐被试唯一删除 IC 数柱状或点图，横轴固定为全部被试，纵轴从 0 开始并使用整数刻度；每个被试标出精确数量。
4. 另绘逐被试删除比例图和 EOG、ECG、肌电、其他原因计数图。原因可能重叠，因此使用并列点或并列柱，不使用会暗示可直接求和的堆叠柱。
5. 报告全体被试唯一删除 IC 数和删除比例的中位数、IQR、范围及有效被试数；处理失败的被试单列原因，不纳入统计分母但不得从图表说明中省略。

### 其他 MNE 作图与判据

1. 绘制滤波前后 PSD；横轴、纵轴和频率范围保持一致。
2. 绘制 `epochs.plot_drop_log()`、逐条件 Trial 保留率及最终各条件试次数。
3. 绘制原始与 MNE 后的波形、PSD、分频带 Topo；比较图使用相同时间范围、频率范围、色标和纵向振幅量表。
4. 使用 `mne.Report` 汇总上述图和参数。

判据依次为：文件与事件正确；坏道和坏段有可追溯证据；ICA 候选同时满足空间、时间、频谱或生理参考证据；清理后伪迹下降但 μ/β 感觉运动节律没有被整体抹除；各条件 Trial 保留率可接受且无明显条件偏倚。

## 来源

- [MNE 预处理教程](https://mne.tools/stable/auto_tutorials/preprocessing/index.html)
- [MNE 坏道处理](https://mne.tools/stable/auto_tutorials/preprocessing/15_handling_bad_channels.html)
- [MNE LOF 坏道检测](https://mne.tools/stable/generated/mne.preprocessing.find_bad_channels_lof.html)
- [MNE 幅度与平坦时段检测](https://mne.tools/stable/generated/mne.preprocessing.annotate_amplitude.html)
- [MNE ICA 文档](https://mne.tools/stable/generated/mne.preprocessing.ICA.html)
- [MNE ICA 操作教程](https://mne.tools/stable/auto_tutorials/preprocessing/40_artifact_correction_ica.html)
- [MNE-BIDS-Pipeline 处理步骤](https://mne.tools/mne-bids-pipeline/stable/features/steps.html)
