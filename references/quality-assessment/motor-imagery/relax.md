# RELAX 自动推荐流程

## 执行原则

- 使用 RELAX 当前稳定版本，在 MATLAB/EEGLAB 中完成清理；记录 MATLAB、EEGLAB、RELAX、PREP、FieldTrip、PICARD 和 ICLabel 的版本。
- 从 `RELAX_SET_PARAMETERS_AND_RUN.m` 设置输入目录、电极位置、工频和输出选项，再由 `RELAX_Wrapper.m` 批量执行。
- 当前官方仓库 v2.0.1 将 targeted wICA 标为推荐的伪迹削减方法。按该版本的自动默认分支执行，不同时启用 MWF、普通 ICLabel-wICA 或 ICA subtraction。
- RELAX 的清理在 MATLAB/EEGLAB 中执行；全部报告图将清理前后及必要的中间 EEGLAB 数据导入 MNE 后绘制。

## 执行范围

1. 预处理范围包括排除后的全部被试。
2. 全部被试除作图外使用同一操作流程。
3. 示例被试只用于展示完整 ICA 诊断图；Bad-channel 和 IC 清理数量均以全部成功完成该流程的被试为统计单位。

## v2.0.1 推荐配置

```matlab
RELAX_cfg.Perform_targeted_wICA=1;
RELAX_cfg.Do_MWF_Once=0;
RELAX_cfg.Do_MWF_Twice=0;
RELAX_cfg.Do_MWF_Thrice=0;
RELAX_cfg.Perform_wICA_on_ICLabel=0;
RELAX_cfg.Perform_ICA_subtract=0;
RELAX_cfg.ICA_method='picard';
RELAX_cfg.Clean_other_comps='no';
RELAX_cfg.ICLabel_thresholds=[0 0 0 0 0 0 0];

RELAX_cfg.FilterType='Butterworth';
RELAX_cfg.causal_or_acausal_filter='acausal';
RELAX_cfg.HighPassFilter=0.5;
RELAX_cfg.LowPassFilter=80;
RELAX_cfg.NotchFilterType='Butterworth';
RELAX_cfg.LineNoiseFrequency=50; % 60 Hz 数据改为 60
RELAX_cfg.DownSample='no';

RELAX_cfg.computerawmetrics=1;
RELAX_cfg.computecleanedmetrics=1;
RELAX_cfg.saveextremesrejected=1;
RELAX_cfg.Report_all_wICA_info='yes';
```

运行其他 RELAX 版本时，以该版本 `RELAX_SET_PARAMETERS_AND_RUN.m` 中明确标注的推荐设置为准，并在报告中列出与上述配置的差异。

## 操作流程

### 1. 输入检查

1. 保留连续 EEG、全部事件和原始采样率；核对电极名称、坐标、EEG/EOG 通道类型及数据单位。
2. 确认实际工频为 50 Hz 或 60 Hz；不能从元数据确定时，先用原始 PSD 判断。
3. 用 `caploc` 指向真实电极坐标文件；在 `ElectrodesToDelete` 和 `electrodes_2_keep_but_not_clean` 中分别配置不参与清理的非头皮通道和需要保留的辅助通道。
4. 将输入转换为 EEGLAB `.set`；转换后再次核对通道、事件样点和时长。

### 2. 滤波

1. 使用四阶、零相位 Butterworth 进行 0.5–80 Hz 带通滤波。
2. 50 Hz 工频使用四阶、零相位 Butterworth 进行 47–53 Hz 陷波；60 Hz 数据使用 57–63 Hz。
3. 默认不降采样。确需降采样时，在低通后执行并记录目标采样率；该分支标为相对默认配置的修改。

### 3. 坏道与极端时段

1. 调用 PREP 检测坏道，再运行 RELAX 的通道级极端噪声检查；保存每个坏道及判定原因。
2. 将连续数据划分为 1 s 窗并以 500 ms 重叠，运行 RELAX 的极端值检测。
3. 保持 v2.0.1 默认阈值：电压位移 `25 × MAD`、绝对幅值 `1000 µV`、异常分布 `10 SD`、单通道与全通道峰度阈值 `10`、极端漂移频谱斜率 `-4`。不得把用于肌电检测的 `MuscleSlopeThreshold` 与极端漂移阈值混用。
4. 单通道受肌电影响的窗口比例超过 `0.50` 或受极端噪声影响的窗口比例超过 `0.25` 时，按 RELAX 规则处理；被删电极比例上限为 `0.10`。
5. 保存坏道列表、被标记窗口起止时间、剔除时长、保留比例及 `RELAX_issues_to_check`。

### 4. 重参考与 targeted wICA

1. 使用 PREP 稳健平均参考：临时插值缺失电极后计算平均参考，再在 ICA 前重新移除坏道，避免插值导致额外秩亏。
2. 用 PICARD-O 拟合 ICA，参数为 `mode='ortho'`、`tol=1e-6`、`maxiter=500`；记录数据秩、成分数和收敛状态。
3. 用 ICLabel 分类。`Clean_other_comps='no'` 时只清理肌电和眼动类成分；`ICLabel_thresholds=[0 0 0 0 0 0 0]` 表示按最大类别概率归类。
4. 眼动类成分使用 5 层 `coif5` stationary wavelet transform；自动阈值由 `ddencmp` 取得，眼动成分阈值乘 2，并只在检测到的眨眼时段保留待减伪迹。
5. 肌电成分以 7–70 Hz、排除工频 ±2 Hz 后的 log-power/log-frequency 斜率判定，默认阈值为 `-0.31`；对命中的成分用二阶、零相位 15 Hz 高通取得待减肌电。
6. 将眼动与肌电待减信号投影回通道空间，从连续 EEG 中减去；保留原事件位置，并记录各类 IC 概率、被清理成分比例和数据是否过短的警告。
7. 为质量评估另存每名被试的 `EEG_with_ICA`、完整 ICLabel 概率、眼动 IC、肌电 IC、最终被清理 IC 及收敛信息；该保存步骤不得改变 RELAX 的清理结果。

### 5. 分段与最终剔除

1. 按真实运动想象事件构造 Epoch；不得用固定时长替代数据集已经定义的 Trial 边界。
2. 使用 `RELAX_EPOCH_CLEAN_DATA_FOR_ANALYSIS.m` 执行最终分段与坏 Epoch 检测，记录逐条件剔除数和保留数。
3. 需要基线校正时使用 `RELAX_RegressionBL_Correction.m`；若数据集方法明确要求减法基线，则按原方法执行并报告偏离。

## 输出

输出内容只包含此流程中提及的内容，其他内容在附录或聊天中报告。

### Bad-channel metric 图

#### 数据整理

对全部纳入被试提取并保存被试 × 通道级数据：通道是否实际存在、PREP 是否判为坏道、`ProportionExtremeForEachChannel`、`ProportionOfEpochsShowingMuscleAboveThresholdPerChannel`、最终是否删除及删除原因。一个通道满足多个原因时保留全部原因。

#### 输出的图

1. **被试 × 通道 metric 热图**：分别绘制极端噪声时段比例和肌电污染 Epoch 比例。行按固定被试顺序，列按标准电极布局顺序；两项比例的色标固定为 0–1，并在色标中标明 `0.25` 和 `0.50` 判据。未采集或无法计算显示为灰色，不得填 0。
2. **最终坏道矩阵**：被试 × 通道矩阵至少区分保留、PREP 删除、极端噪声删除、肌电删除、多原因删除和未采集。不得只展示被删除通道。
3. **逐通道数据集统计图**：对每个通道计算 `坏道被试数 / 实际含该通道且完成处理的被试数`，绘制坏道率及二项比例的 95% Wilson 置信区间；同时标注分子和分母。通道顺序与热图一致，不按坏道率重新排序。
4. **坏道率 Topography**：使用 MNE 的 `mne.viz.plot_topomap()` 绘制逐通道坏道率，色标固定为 0–1。该图只作为空间分布摘要，统计解释以第 3 项的点估计和置信区间为准。
5. **逐被试坏道负担图**：每名被试显示坏道数和坏道比例，并叠加全体被试的中位数与 IQR；在比例图标出 RELAX 的 `0.10` 电极删除上限。所有被试均显示为独立点，不用只有箱线图的形式替代。

所有图标题写明纳入被试数、成功处理被试数、标准通道数和 RELAX 版本。相同数据集的不同预处理分支使用相同被试顺序、通道顺序、画布尺寸和色标。

### ICA 图

#### 示例被试的完整 ICA 诊断图

对调用前确认的每个示例被试，读取报告用 ICA 中间文件，并固定输出以下内容：

1. **全部 IC 总览**：使用 MNE 绘制全部 IC 的 Topography 网格；每个 IC 显示编号，以固定边框或标题颜色区分保留、眼动清理、肌电清理和同时满足两类判据，不得只画被清理 IC。
2. **分类与判据总览**：绘制 IC × ICLabel 类别概率热图，类别顺序固定为 Brain、Muscle、Eye、Heart、Line Noise、Channel Noise、Other；并列显示肌电频谱斜率、`-0.31` 阈值、最终清理状态和是否由 `icablinkmetrics` 补充识别。
3. **每个被清理 IC 的五联图**：使用 MNE `ICA.plot_properties()`，完整保留 Topography、Epoch image、平均 IC 波形及置信区间、PSD、逐 Epoch 方差五部分。所有被清理 IC 均需绘制；数量过多时分页，不得只挑代表性 IC。
4. **时序证据图**：使用 `ICA.plot_sources()` 展示被清理 IC 在整段记录和真实 Trial 内的活动；眼动 IC 标出眨眼时段，肌电 IC 的 PSD 标出 15 Hz 以及用于斜率判断的 7–70 Hz 范围。
5. **清理效果图**：在同一伪迹时间窗内绘制清理前后通道波形和 GFP 叠加图，时间范围、通道、纵轴和颜色在示例被试之间保持一致。眼动和肌电至少各展示一个真实命中时段；不存在对应伪迹时明确标为无，不用其他时段替代。

#### 全数据集 IC 清理统计

1. 在报告中提供逐被试表，字段固定为：被试编号、进入 ICA 的 EEG 通道数、数据秩、IC 总数、眼动 IC 数、肌电 IC 数、同时命中两类的 IC 数、唯一被清理 IC 数、被清理 IC 比例、收敛状态和 `DataMaybeTooShortForValidICA`。
2. RELAX targeted wICA 只清理 IC 中的目标伪迹成分，不删除整个 IC；报告中使用“被清理 IC 数”，不得写成“删除 IC 数”。唯一被清理 IC 数按 IC 编号去重，不能把眼动数与肌电数直接相加。
3. 绘制逐被试唯一被清理 IC 数柱状或点图，横轴固定为全部被试，纵轴从 0 开始并使用整数刻度；每个被试标出精确数量。
4. 另绘逐被试被清理 IC 比例图和眼动、肌电分类计数图。分类可能重叠，因此使用并列点或并列柱，不使用会暗示可直接求和的堆叠柱。
5. 报告全体被试唯一被清理 IC 数和比例的中位数、IQR、范围及有效被试数；处理失败的被试单列原因，不纳入统计分母但不得从图表说明中省略。

## 来源

- [RELAX 官方仓库](https://github.com/NeilwBailey/RELAX)
- [RELAX 安装与运行说明](https://github.com/NeilwBailey/RELAX/wiki/1-%E2%80%90-Installing-and-Running-RELAX)
- [RELAX 坏道与通道级指标实现](https://github.com/NeilwBailey/RELAX/blob/v2.0.1/RELAX_excluding_channels_and_epoching.m)
- [RELAX targeted wICA 实现](https://github.com/NeilwBailey/RELAX/blob/v2.0.1/RELAX_targeted_wICA.m)
- [RELAX Part 1 方法论文](https://github.com/NeilwBailey/RELAX/blob/v2.0.1/Bailey%20et%20al%20%282023%29%20RELAX%20part%201%20-%20algorithm%20and%20application%20to%20oscillations.pdf)
- [Targeted wICA 方法](https://doi.org/10.1101/2024.06.06.597688)
