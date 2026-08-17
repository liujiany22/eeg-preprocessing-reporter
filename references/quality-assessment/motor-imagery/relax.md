# RELAX 自动推荐流程

## 目录

- 执行原则与推荐配置
- 操作流程
- Bad-channel metric 图
- ICA 图
- 其他 MNE 作图与输出
- 来源

## 执行原则

- 使用 RELAX 当前稳定版本，在 MATLAB/EEGLAB 中完成清理；记录 MATLAB、EEGLAB、RELAX、PREP、FieldTrip、PICARD 和 ICLabel 的版本。
- 从 `RELAX_SET_PARAMETERS_AND_RUN.m` 设置输入目录、电极位置、工频和输出选项，再由 `RELAX_Wrapper.m` 批量执行。
- 当前官方仓库 v2.0.1 将 targeted wICA 标为推荐的伪迹削减方法。按该版本的自动默认分支执行，不同时启用 MWF、普通 ICLabel-wICA 或 ICA subtraction。
- RELAX 的清理在 MATLAB/EEGLAB 中执行；全部报告图将清理前后及必要的中间 EEGLAB 数据导入 MNE 后绘制。

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

### 5. 分段与最终剔除

1. 按真实运动想象事件构造 Epoch；不得用固定时长替代数据集已经定义的 Trial 边界。
2. 使用 `RELAX_EPOCH_CLEAN_DATA_FOR_ANALYSIS.m` 执行最终分段与坏 Epoch 检测，记录逐条件剔除数和保留数。
3. 需要基线校正时使用 `RELAX_RegressionBL_Correction.m`；若数据集方法明确要求减法基线，则按原方法执行并报告偏离。

## Bad-channel metric 图

将原始和 RELAX 输出的 `.set` 文件分别用 `mne.io.read_raw_eeglab()` 读入，并保证通道顺序、事件和测量单位一致。

1. 绘制坏道指标图和坏道头皮位置图，标出每个坏道的判定原因及对应阈值。
2. 绘制 1 s 异常窗口时间轴，分别标出各类极端值及其阈值。
3. 报告坏道数、坏道比例、剔除时长和保留比例。

## ICA 图

用 `mne.preprocessing.read_ica_eeglab()` 读取包含 PICARD/ICLabel 结果的中间 `.set`；当前版本未保存该中间对象时，在 `iclabel()` 后仅增加一次报告用保存，不改变清理计算。

1. 绘制 ICA 成分 Topography、时间序列和 PSD。
2. 绘制 ICLabel 类别概率并标出被清理的眼动与肌电成分。
3. 绘制眼动和肌电目标信号清理前后叠加图。
4. 报告 ICA 成分总数、被清理 IC 数、分类依据和收敛状态。

## 其他 MNE 作图与输出

1. 绘制原始与 RELAX 后的波形、PSD、分频带 Topo；比较图使用相同时间范围、频率范围、色标和纵向振幅量表。
2. 报告逐条件 Trial 保留率。
3. 绘制 RELAX 输出的 Blink amplitude ratio、肌电强度、肌电污染窗口比例、SER 和 ARR；指标缺失时不自行补造。

核心 MNE 接口包括 `mne.io.read_raw_eeglab()`、`Raw.plot()`、`Raw.compute_psd()`、`ICA.plot_components()`、`ICA.plot_properties()`、`ICA.plot_overlay()`、`mne.viz.plot_topomap()` 和 `mne.Report`。

## 来源

- [RELAX 官方仓库](https://github.com/NeilwBailey/RELAX)
- [RELAX 安装与运行说明](https://github.com/NeilwBailey/RELAX/wiki/1-%E2%80%90-Installing-and-Running-RELAX)
- [RELAX Part 1 方法论文](https://github.com/NeilwBailey/RELAX/blob/v2.0.1/Bailey%20et%20al%20%282023%29%20RELAX%20part%201%20-%20algorithm%20and%20application%20to%20oscillations.pdf)
- [Targeted wICA 方法](https://doi.org/10.1101/2024.06.06.597688)
