# MNE 推荐预处理流程

MNE 没有适用于全部 EEG 数据集的单一固定参数流程。此处按照 MNE 官方预处理顺序组织运动想象任务的默认实现；所有由本流程给出的任务默认值均须在报告中标明，不能写成数据集原有参数。

## 目录

- 方法边界
- 执行范围
- 预处理操作流程
- 核心代码顺序
- 预处理报告输出
- 质量评估步骤
- 质量评估报告输出
- 质量评估报告附录
- 不生成的内容
- 来源

## 方法边界

- 数据集资料明确给出的参考、滤波、Trial、基线或剔除参数优先于本流程默认值。
- 本文件中的默认值必须标为“MNE 流程默认值”；实际数据不支持时修改并记录，不得写成数据集原有设置。
- 阈值、通道类型、事件映射和 Trial 边界必须由资料或实际数据确定。未确定前不运行受影响步骤，也不以常见数据集设置代替。
- 报告中的坏道、ICA 和 Epoch 结果必须来自最终导出数据所使用的同一次 MNE 运行；不得为补图另行拟合 ICA 或重新判定坏道。

## 执行范围

1. 预处理范围包括排除后的全部被试。
2. 全部被试使用同一版本、步骤和数据集级参数集；只有示例 ICA 诊断图限于调用前确认的被试。
3. Bad-channel 和 IC 删除统计以全部进入本流程的被试为范围。成功被试进入统计分母；失败被试单列阶段和原因。

## 预处理操作流程

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

## 预处理报告输出

MNE 在预处理报告中只输出以下内容，顺序固定：

1. **执行状态**：进入流程、成功、失败的被试数；失败阶段和原因摘要；MNE 版本和数据集级参数集标识。
2. **实际流程与参数**：用一条按执行顺序排列的流程区分数据集原有参数、MNE 流程默认值和因数据而修改的参数；用参数表给出实际值及依据。占位变量和未执行分支不得进入报告。
3. **预处理结果**：逐条件给出总 Trial 数、各原因剔除数、保留数和保留率。

## 质量评估步骤

1. 只读取最终导出数据所对应的同一次 MNE 预处理保存的坏道、Bad-channel metric、ICA、Annotations 和 Epoch 结果，不重新运行预处理。
2. 使用这些保存结果生成下述数据集级统计、诊断图和附录明细。

## 质量评估报告输出

MNE 在正文中只输出以下内容，顺序固定：

1. **执行状态**：进入流程、成功、失败的被试数；失败阶段和原因摘要；MNE 版本和数据集级参数集标识。
2. **实际流程与参数**：用一条按执行顺序排列的流程区分数据集原有参数、MNE 流程默认值和因数据而修改的参数；用参数表给出实际值及依据。占位变量和未执行分支不得进入报告。
3. **Bad-channel 数据集级结果**：
   - 逐通道坏道率及 95% Wilson 置信区间，逐点标注 `坏道被试数 / 实际含该通道且成功处理的被试数`，通道按标准布局排序。
   - 使用 `mne.viz.plot_topomap()` 绘制逐通道坏道率，色标固定为 0–1；只作空间摘要，结论以点估计和置信区间为准。
   - 逐被试坏道数、坏道比例、坏段时长比例和插值通道数；显示全部成功被试、中位数和 IQR。
4. **IC 删除数据集级结果**：
   - 逐被试唯一删除 IC 数和删除比例；所有成功被试均显示，数量使用从 0 开始的整数轴并标注精确值。
   - EOG、ECG、肌电、其他原因和多原因 IC 使用并列点或并列柱，不使用堆叠柱。
   - 报告唯一删除 IC 数和比例的中位数、IQR、范围及有效被试数，并引用附录 11.3 的逐被试表。
5. **Epoch 结果**：逐条件给出总 Trial 数、各原因剔除数、保留数和保留率，并说明是否存在条件间不均衡；没有预先判据时只报告差异，不自行判定“可接受”。

唯一删除 IC 按 IC 编号去重，不能把不同原因计数直接相加；删除比例固定为 `唯一删除 IC 数 / IC 总数`。

所有数据集级图写明进入流程被试数、成功被试数、标准通道数、MNE 版本和参数集标识。连续 metric 的色标范围用全部成功被试一次确定并冻结，不按被试单独缩放。

## 质量评估报告附录

以下内容只放在附录 11.3，正文引用而不重复：

### Bad-channel 明细

1. **逐被试逐通道表**：字段固定为通道是否存在、LOF score、`BAD_flat` 时长比例、`BAD_peak` 时长比例、工频功率比、邻近通道中位相关、各自动判定、最终状态和人工改动理由。
2. **metric 热图**：LOF score、`BAD_flat` 比例、`BAD_peak` 比例、工频功率比和邻近通道中位相关分别成图；每项使用固定色标，图注写明单位、方向和实际阈值；缺失显示为灰色。
3. **最终坏道矩阵**：至少区分保留、LOF 判坏、幅度或平坦判坏、多指标判坏、人工加入、人工保留和未采集，不得只显示坏道。
4. **示例证据**：只对已确认示例被试绘制带坏道和坏段标记的连续波形、全频 PSD、电极位置以及插值前后代表性波形；这些图不替代数据集级统计。

### 示例被试 ICA 完整诊断

对每个已确认的示例被试，使用该被试最终处理时保存的同一个 ICA 对象固定输出：

1. **全部 IC 总览**：使用 `ICA.plot_components()` 显示所有 IC 编号，以固定视觉编码区分保留、EOG、ECG、肌电、其他人工排除和多原因排除。
2. **自动判据总览**：对实际可计算的 EOG、ECG、肌电分别使用 `ICA.plot_scores()` 显示全部 IC score、实际阈值和最终状态；不可计算时标为不可用，不构造代理分数。
3. **全部排除 IC 的完整属性图**：使用 `ICA.plot_properties()`，保留 Topography、Epoch image、平均 IC 波形及置信区间、PSD、逐 Epoch 方差；数量过多时分页，不挑代表性 IC 代替全集。
4. **时序证据**：使用 `ICA.plot_sources()` 展示排除 IC 在整段记录和真实 Trial 内的活动，并标出实际用于判断的伪迹时段。
5. **处理效果**：使用 `ICA.plot_overlay()` 在同一真实伪迹时窗展示处理前后通道波形和 EEG GFP；示例被试间固定时间范围、通道、纵轴和颜色。某类伪迹不存在时标为无，不用其他类型替代。

### 逐被试 IC 与 Epoch 表

- IC 表字段固定为：被试编号、进入 ICA 的 EEG 通道数、数据秩、IC 总数、EOG IC 数、ECG IC 数、肌电 IC 数、其他人工排除 IC 数、多原因 IC 数、唯一删除 IC 数、删除比例、ICA 算法、收敛状态、处理状态和失败原因摘要。
- Epoch 表按被试和条件列出原始 Trial 数、各原因剔除数、保留数和保留率，并附 `epochs.plot_drop_log()`。

## 不生成的内容

- 原始与 MNE 后的波形、PSD 和分频带 Topography 只在质量评估报告附录 11.1 中生成，不在本节重复。
- `mne.Report` 仅作为承载正文和附录的容器，不因此新增图、表或自动结论。
- 不自动给出被试排除建议、跨方法排名、综合质量分或没有预先判据支持的“质量合格”结论。
- 缺少 EOG、ECG 或肌电判据时，只把对应检测标为不可用，不用额区通道或其他指标替代，除非用户明确要求并将其标为方法偏离。

## 来源

以下链接用于约束流程，不作为固定报告章节；报告生成时按主 skill 的规定说明核查材料。

- [MNE 预处理教程](https://mne.tools/stable/auto_tutorials/preprocessing/index.html)
- [MNE 坏道处理](https://mne.tools/stable/auto_tutorials/preprocessing/15_handling_bad_channels.html)
- [MNE LOF 坏道检测](https://mne.tools/stable/generated/mne.preprocessing.find_bad_channels_lof.html)
- [MNE 幅度与平坦时段检测](https://mne.tools/stable/generated/mne.preprocessing.annotate_amplitude.html)
- [MNE ICA 文档](https://mne.tools/stable/generated/mne.preprocessing.ICA.html)
- [MNE ICA 操作教程](https://mne.tools/stable/auto_tutorials/preprocessing/40_artifact_correction_ica.html)
- [MNE-BIDS-Pipeline 处理步骤](https://mne.tools/mne-bids-pipeline/stable/features/steps.html)
