# RELAX 自动推荐流程

## 目录

- 方法边界
- 执行范围
- v2.0.1 推荐配置
- 操作流程
- 正文输出
- 附录输出
- 不生成的内容
- 来源

## 方法边界

- 使用 RELAX 当前稳定版本，在 MATLAB/EEGLAB 中完成清理；记录 MATLAB、EEGLAB、RELAX、PREP、FieldTrip、PICARD 和 ICLabel 的版本。
- 从 `RELAX_SET_PARAMETERS_AND_RUN.m` 设置输入目录、电极位置、工频和输出选项，再由 `RELAX_Wrapper.m` 批量执行。
- 当前官方仓库 v2.0.1 将 targeted wICA 标为推荐的伪迹削减方法。按该版本的自动默认分支执行，不同时启用 MWF、普通 ICLabel-wICA 或 ICA subtraction。
- RELAX 的清理在 MATLAB/EEGLAB 中执行；全部报告图将清理前后及必要的中间 EEGLAB 数据导入 MNE 后绘制。
- 报告图必须使用 RELAX 实际运行产生的坏道、ICA 权重、IC 源活动和清理状态；导入 MNE 后不得重新拟合 ICA 或重新判定坏道。导入前后须核对 IC 编号、Topography、源活动和事件样点。

## 执行范围

1. 预处理范围包括排除后的全部被试。
2. 全部被试使用同一版本、配置、步骤和判据；只有示例 ICA 诊断图限于调用前确认的被试。
3. Bad-channel 和 IC 清理统计以全部进入本流程的被试为范围。成功被试进入统计分母；失败被试单列阶段和原因，不从流程记录中删除。

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

## 正文输出

RELAX 在正文中只输出以下内容，顺序固定：

1. **执行状态**：进入流程、成功、失败的被试数；失败阶段和原因摘要；实际 RELAX 版本；实际使用的配置分支及相对本文件推荐配置的差异。
2. **实际流程与参数**：用一条按执行顺序排列的流程和一张参数表记录实际执行项。未启用的可选分支不展开。
3. **Bad-channel 数据集级结果**：
   - 逐通道坏道率及 95% Wilson 置信区间，逐点标注 `坏道被试数 / 实际含该通道且成功处理的被试数`；通道按标准布局顺序，不按坏道率排序。
   - 使用 `mne.viz.plot_topomap()` 绘制逐通道坏道率，色标固定为 0–1；只作空间摘要，结论以点估计和置信区间为准。
   - 逐被试坏道数与坏道比例，显示全部成功被试、中位数和 IQR，并标出 `0.10` 电极删除上限。
4. **IC 清理数据集级结果**：
   - 逐被试唯一被清理 IC 数与比例；所有成功被试均显示，数量使用从 0 开始的整数轴并标注精确值。
   - 逐被试眼动、肌电及重叠类别计数使用并列点或并列柱，不使用堆叠柱。
   - 报告唯一被清理 IC 数及比例的中位数、IQR、范围和有效被试数，并引用附录 11.2 的逐被试表。
5. **Epoch 结果**：按条件给出最终剔除数、保留数和保留率。

RELAX targeted wICA 只清理 IC 内的目标伪迹信号，不删除整个 IC。正文和附录统一使用“被清理 IC”，不得写成“删除 IC”。唯一被清理 IC 按 IC 编号去重，不能把眼动数与肌电数直接相加。

所有数据集级图写明进入流程被试数、成功被试数、标准通道数、RELAX 版本和配置标识。同一数据集不同流程使用相同被试顺序、通道顺序和画布规格。

## 附录输出

以下内容只放在附录 11.2，正文引用而不重复：

### 完整配置

- 实际运行的核心配置代码、全部实际生效参数、软件版本和相对推荐配置的偏离。
- 逐被试处理状态表。技术栈追踪和原始日志不写入报告。

### Bad-channel 明细

1. **逐被试逐通道表**：字段固定为通道是否实际存在、PREP 是否判坏、`ProportionExtremeForEachChannel`、`ProportionOfEpochsShowingMuscleAboveThresholdPerChannel`、最终状态和全部判定原因。
2. **两张 metric 热图**：分别显示极端噪声时段比例和肌电污染 Epoch 比例；行列顺序固定，色标为 0–1，标出 `0.25` 和 `0.50` 判据；缺失显示为灰色。
3. **最终坏道矩阵**：至少区分保留、PREP 删除、极端噪声删除、肌电删除、多原因删除和未采集，不得只显示坏道。

### 示例被试 ICA 完整诊断

对每个已确认的示例被试，使用该被试实际 RELAX 运行保存的同一 ICA 结果固定输出：

1. **全部 IC Topography 总览**：显示所有 IC 编号，以固定视觉编码区分保留、眼动清理、肌电清理和重叠命中；不得只画被清理 IC。
2. **分类与判据总览**：IC × ICLabel 概率热图，类别顺序固定为 Brain、Muscle、Eye、Heart、Line Noise、Channel Noise、Other；并列显示肌电频谱斜率、`-0.31` 阈值、最终状态和 `icablinkmetrics` 补充识别状态。
3. **全部被清理 IC 的完整属性图**：在实际 RELAX ICA 能无损映射并通过 IC 编号、Topography 和源活动核对时，使用 MNE `ICA.plot_properties()`；否则使用导出的实际 IC 源活动与权重，通过 MNE 的 Epoch、PSD 和 Topography 计算组成相同五部分，并在图注标明实现。完整保留 Topography、Epoch image、平均 IC 波形及置信区间、PSD、逐 Epoch 方差；数量过多时分页，不挑代表性 IC 代替全集。
4. **时序证据**：使用 `ICA.plot_sources()` 展示被清理 IC 在整段记录和真实 Trial 内的活动；眼动 IC 标出实际眨眼时段，肌电 IC 标出 15 Hz 及 7–70 Hz 判定范围。
5. **清理效果**：在同一真实伪迹时间窗绘制清理前后通道波形和 GFP；示例被试间固定时间范围、通道、纵轴和颜色。某类伪迹没有真实命中时标为无，不以其他时段代替。

### 逐被试 IC 表

字段固定为：被试编号、进入 ICA 的 EEG 通道数、数据秩、IC 总数、眼动 IC 数、肌电 IC 数、重叠 IC 数、唯一被清理 IC 数、被清理比例、收敛状态、`DataMaybeTooShortForValidICA`、处理状态和失败原因摘要。

## 不生成的内容

- 原始与 RELAX 后的波形、PSD 和分频带 Topography 只按附录 11.1 生成，不在本节重复。
- 不根据 RELAX 结果自动给出被试排除建议、跨方法排名、综合质量分或未由实际结果支持的原因解释。
- 没有保存相应 ICA 中间量时，相关诊断图标为未生成，不从清理后的通道数据反推 IC 结果。
- 不为补图在 MNE 中重新拟合 ICA；重新拟合得到的成分不得标为 RELAX 的 IC。

## 来源

以下链接用于约束流程，不作为固定报告章节；报告生成时按主 skill 的规定说明核查材料。

- [RELAX 官方仓库](https://github.com/NeilwBailey/RELAX)
- [RELAX 安装与运行说明](https://github.com/NeilwBailey/RELAX/wiki/1-%E2%80%90-Installing-and-Running-RELAX)
- [RELAX 坏道与通道级指标实现](https://github.com/NeilwBailey/RELAX/blob/v2.0.1/RELAX_excluding_channels_and_epoching.m)
- [RELAX targeted wICA 实现](https://github.com/NeilwBailey/RELAX/blob/v2.0.1/RELAX_targeted_wICA.m)
- [RELAX Part 1 方法论文](<https://github.com/NeilwBailey/RELAX/blob/v2.0.1/Bailey%20et%20al%20%282023%29%20RELAX%20part%201%20-%20algorithm%20and%20application%20to%20oscillations.pdf>)
- [Targeted wICA 方法](https://doi.org/10.1101/2024.06.06.597688)
