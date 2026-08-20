# 运动想象质量评估统计图表与可视化

## 使用范围

- 本文件规定运动想象质量评估中固定采用或按条件采用的统计图表和可视化方法。报告仍须调研当前任务和所用预处理方法的原始文献，不以本文件代替调研。
- “单独被试图”只指整张图展示一名被试的波形、ICA 或其他诊断结果。逐被试统计图、被试内条件平均，以及先得到每名被试的条件平均再进行组级聚合的图，不属于应当省略的单独被试图。
- 单 Session 条件结果按“Trial → 被试×条件 → 组级”聚合。多 Session 条件结果按“Trial → 被试×Session×条件 → 被试×同条件 → 组级”聚合：先在 Session 内汇总同条件 Trial，再在被试内对同条件 Session 等权平均，最后对被试等权聚合。条件缺失保持为缺失，不跨条件合并。跨 Session 可靠性分析保留 Session 维度，不执行第二级 Session 平均。
- 所有 EEG 图使用 MNE 相关代码生成；读取外部流程输出时先核对被试、通道、事件、IC 编号和数据阶段，不为补图重建无法追溯的结果。

## 常规质量评估方法

以下方法不属于 RELAX 或 MNE 专有方法；只要当前预处理流程产生所需中间量，就按相同定义和图形规格使用。

| 统计图表或可视化及描述 | 含义和值得关注的点 | 前置预处理要求 | 来源 |
|---|---|---|---|
| 流程执行与数据保留表：按方法列出进入、成功、失败被试，失败阶段，输入与输出时长、通道数、Trial 数 | 先确认统计分母和数据损耗；关注失败是否集中于特定被试、Session 或条件 | 只需完整运行记录和输入、输出索引 | [MNE-BIDS-Pipeline 处理步骤](https://mne.tools/mne-bids-pipeline/stable/features/steps.html) |
| 被试×通道 metric 明细表及热图：每种坏道指标单独成表、成图，缺失值单独编码 | 展示坏道判据在全数据集中的分布；关注同一通道跨被试反复异常、单一被试多通道异常及阈值附近结果 | 需要预处理实际保存的逐被试逐通道 metric、阈值和判定；不得从最终坏道列表反推连续 metric | [PREP Pipeline](https://doi.org/10.3389/fninf.2015.00016)、[MNE LOF](https://mne.tools/stable/generated/mne.preprocessing.find_bad_channels_lof.html) |
| 最终坏道明细表及被试×通道状态矩阵：区分保留、各判据判坏、多原因、人工改动和未采集 | 展示最终决策及原因；关注方法间判定差异、人工覆盖及缺失通道 | 需要最终坏道状态、全部判定原因和人工改动记录 | [PREP Pipeline](https://doi.org/10.3389/fninf.2015.00016)、[MNE 坏道处理](https://mne.tools/stable/auto_tutorials/preprocessing/15_handling_bad_channels.html) |
| 逐通道坏道率及置信区间表、点图和坏道率 Topography | 量化各电极被判坏的频率及不确定性；Topography 只作空间摘要，结论以计数、比例和区间为准 | 需要全体成功被试的最终坏道列表和每名被试实际采集通道；Topography 需要真实电极位置 | [PREP Pipeline](https://doi.org/10.3389/fninf.2015.00016)、[MNE Topomap](https://mne.tools/stable/generated/mne.viz.plot_topomap.html) |
| 逐被试坏道、坏段和插值统计表及分布图 | 衡量各被试的数据损耗；关注分布尾部、预设阈值和失败被试，而不只报告均值 | 需要坏道列表、坏段 Annotation、插值记录和实际记录时长 | [HAPPE](https://doi.org/10.3389/fnins.2018.00097)、[PREP Pipeline](https://doi.org/10.3389/fninf.2015.00016) |
| 逐被试 IC 处理表及分布图：IC 总数、各原因命中数、唯一处理数和比例、秩与收敛状态 | 量化 ICA 对各被试的影响；关注多原因 IC、极端处理比例、秩不足和未收敛 | 需要同一次实际预处理保存的 ICA、分类或判据结果和最终处理列表 | [MNE ICA](https://mne.tools/stable/generated/mne.preprocessing.ICA.html)、[HAPPE](https://doi.org/10.3389/fnins.2018.00097) |
| 单独被试 ICA 完整诊断组图：全部 IC Topography、全部自动 score、全部被处理 IC 的 properties、sources 和处理前后 overlay | 用于解释明确异常或值得复核的 ICA 结果；不得只展示最符合预期的成分 | 仅在组级统计发现离群、判据冲突、异常处理比例、收敛问题或用户指定时生成；必须使用该被试实际运行保存的同一 ICA | [MNE ICA 教程](https://mne.tools/stable/auto_tutorials/preprocessing/40_artifact_correction_ica.html)、[MNE ICA API](https://mne.tools/stable/generated/mne.preprocessing.ICA.html) |
| 逐被试×条件 Trial 剔除表及保留率图 | 判断数据损耗及条件不均衡；关注某条件或某被试的系统性高剔除率 | 需要原始 Trial 索引、剔除原因和最终 Epoch 索引 | [MNE Epochs](https://mne.tools/stable/generated/mne.Epochs.html) |
| 组级 PSD 及预处理前后配对变化图：先计算被试级 PSD，再汇总中位数或均值及离散程度 | 检查工频、低频漂移、高频噪声及目标频带是否被过度衰减；关注预处理前后而不是单一阶段曲线 | 需要同一被试、同一数据范围和通道集合的可追溯前后数据；PSD 参数固定 | [MNE Raw](https://mne.tools/stable/generated/mne.io.Raw.html)、[MNE Spectrum](https://mne.tools/stable/auto_tutorials/time-freq/10_spectrum_class.html) |
| 单独被试的波形或 GFP 前后对比 | 只用于定位组级统计指出的具体伪迹、插值异常或清理过度，不作为数据集级质量结论 | 仅在有明确异常证据或用户指定时生成；前后必须为同一真实时间窗、通道、纵轴和事件区间 | [MNE ICA overlay](https://mne.tools/stable/generated/mne.preprocessing.ICA.html)、[MNE Raw 浏览](https://mne.tools/stable/auto_tutorials/raw/10_raw_overview.html) |

## 运动想象任务特异方法

| 统计图表或可视化及描述 | 含义和值得关注的点 | 前置预处理要求 | 来源 |
|---|---|---|---|
| 被试条件平均后的组级 ERDS 时频图：在实际感觉运动通道或预先定义 ROI 展示各条件相对基线的时频变化 | 直接观察运动想象期间 μ、β 节律的 ERD/ERS 时间和频率结构；关注效应是否出现在合理时间窗、频带和感觉运动区域，以及被试间是否一致 | 需要可验证的 Trial、条件标签和有效基线；需完成与来源一致的滤波、伪迹处理、Epoch 和基线归一化；显著性掩膜需被试级统计 | [MNE ERDS 示例](https://mne.tools/stable/auto_examples/time_frequency/time_frequency_erds.html)、[Pfurtscheller 与 Lopes da Silva 1999](https://doi.org/10.1016/S1388-2457(99)00141-8) |
| 被试条件平均后的 μ、β ERD/ERS 时间曲线：并列展示条件及组级区间 | 展示效应方向、起始、峰值和持续时间；关注左右手任务的对侧与同侧差异、足部任务的中央区变化以及基线稳定性 | 与 ERDS 时频图相同；频带、ROI 和时间窗须由范式及文献预先确定 | [GigaScience MI 数据集](https://doi.org/10.1093/gigascience/gix034)、[Multi-day MI 数据集](https://doi.org/10.1038/s41597-025-04826-y) |
| 被试条件平均后的 μ、β ERD/ERS Topography：条件并排、统一色标，在预先定义时间窗内平均 | 检查变化是否位于感觉运动区及是否存在合理的对侧化或中央分布；不把插值边缘或单点极值解释为效应 | 需要真实电极位置、有效基线、条件 Epoch 和预先确定的频带与时间窗 | [GigaScience MI 数据集](https://doi.org/10.1093/gigascience/gix034)、[MNE TFR Topomap](https://mne.tools/stable/generated/mne.time_frequency.AverageTFRArray.html) |
| 被试级任务效应明细表及分布图：每名被试、条件、ROI、频带和时间窗的 ERD/ERS 摘要 | 防止组平均掩盖方向相反或无效的被试；关注效应方向一致性、离群值和有效被试数 | 需要先得到每名被试的条件平均；不以 Trial 作为独立被试 | [GigaScience MI 数据集](https://doi.org/10.1093/gigascience/gix034)、[MNE ERDS 示例](https://mne.tools/stable/auto_examples/time_frequency/time_frequency_erds.html) |
| CSP pattern、filter 及被试级特征分布 | 检查分类所利用的空间模式是否与任务相关且不是单个异常通道主导；pattern 用于生理解释，不能把 filter 直接当头皮激活 | 仅在条件标签、Trial 数和交叉验证设计支持时执行；CSP 必须在训练折内拟合 | [MNE CSP](https://mne.tools/stable/generated/mne.decoding.CSP.html)、[GigaScience MI 数据集](https://doi.org/10.1093/gigascience/gix034) |
| CSP+SVM 被试级交叉验证指标和置信区间 | 作为统一简单模型的下游可分性检查；关注跨被试离散、类别不均衡和数据泄漏 | 按质量评估模板第 5 节及其分类配置执行；CSP、标准化和 SVM 均须在训练折内拟合 | [MNE CSP 解码示例](https://mne.tools/stable/auto_examples/decoding/decoding_csp_eeg.html) |
| 多 Session 的被试内配对图及可靠性统计 | 检查任务效应和质量指标在日期间的稳定性；关注系统漂移、Session 缺失及配对样本数 | 仅用于多 Session 数据；需要相同定义的被试条件平均和完整配对关系 | [Multi-day MI 数据集](https://doi.org/10.1038/s41597-025-04826-y) |

## 固定执行原则

1. 预处理前的原始数据固定输出四类非单独被试图：全部纳入被试整体平均 PSD、全部纳入被试分条件平均 PSD、全部纳入被试整体平均分频带 Topography、全部纳入被试分条件平均分频带 Topography。计算与报告配置以质量评估模板第 2.1 和 2.2 节为准。
2. 预处理方法的质量评估只要数据和必要前置处理可用，固定输出：流程执行与数据保留表、坏道表与数据集级图、IC 表与数据集级图、Trial 保留表与图、组级 PSD，以及 ERDS 时频图、μ/β ERD/ERS 曲线、μ/β Topography 和被试级任务效应表。
3. 某一流程不包含坏道、ICA 或 Trial 拒绝时，对应项标为“不适用”，不得为了凑齐常规图而加入该处理。
4. 第 5 节固定尝试 CSP+SVM 分类；标签、Trial、通道或数据秩不满足分类配置时报告无法执行。CSP pattern 和跨 Session 可靠性图只在其适用条件成立时采用。
5. 单独被试 ICA、波形、PSD、Topography 或其他诊断图不固定生成。只有组级或逐被试统计已经指出明确异常、方法判据需要复核、用户指定，或该图能够回答已写明的问题时才生成，并在图注说明选择依据。
6. 组级图同时报告被试数和每个条件的有效被试数；被试级汇总表保留无效或缺失状态，不用 0 填补。

## 常规质量评估图固定配置

### Bad-channel

- metric 连续值热图固定为一项 metric 一张图。行是被试×Session；方法只产生被试级结果时使用被试。列按标准电极布局顺序排列。使用 metric 原始单位，不对每名被试单独标准化；色标由该方法全部成功记录一次确定并冻结。满足该 metric 判坏阈值的单元格加黑色边框，实际阈值和方向写入副标题；不用改变色标伪造阈值边界。未采集与无法计算使用两种不同的缺失纹理或颜色。
- 最终坏道状态矩阵与 metric 热图使用完全相同的行列顺序，采用离散色图，至少区分保留、各单一判据、多判据、人工加入、人工保留和未采集。矩阵格不做插值，状态图例列出每类的精确记录数；不得把不同原因合并为一个二元坏道图。
- 逐通道表同时报告判坏被试×Session 数、实际采集被试×Session 数、主要原因及数量。单 Session 数据的通道坏道率以被试为单位，并使用 95% Wilson CI；多 Session 数据先计算每名被试每个通道的坏 Session 比例，再对被试等权平均，95% CI 使用随机种子 42 的 5000 次被试级 bootstrap。
- 坏道率 Topography 的输入是上一条得到的被试等权逐通道坏道率。调用 `mne.viz.plot_topomap()`，固定 `vlim=(0, 1)`、`cmap="YlOrRd"`、`sensors=True`、`contours=np.arange(0, 1.01, 0.2)`、`image_interp="cubic"`、`extrapolate="head"`、`border="mean"`、`res=256`。所有通道显示传感器点，Topography 旁并排放置按坏道率由高到低排列的横向点区间图，以通道名和 95% CI 给出精确位置；Topography 只作空间摘要。
- 逐被试表在多 Session 时同时报告至少一次判坏的唯一通道数、平均每 Session 坏道数、平均每 Session 坏道比例和插值数范围。被试整体分布固定为三面板：平均每 Session 坏道数横向点图、平均每 Session 坏道比例横向点图、被试级坏道比例直方图与底部 rug（轴边短线）。前两个面板显示全部被试并使用相同被试顺序；直方图用实线标中位数、虚线标 IQR。比例轴固定为 0–1，数量轴从 0 开始；被试顺序在各方法间一致。
- Topography 缺少足够的非共线真实电极位置时不插值作图，改为带通道名、坏道率和置信区间的布局点图，并报告原因。

### ICA

- 数据集级 IC 表按被试×Session 列出数据秩、IC 总数、唯一处理数、处理比例、各原因命中数、多原因 IC 数和收敛状态；另加被试汇总行，多 Session 时对 Session 等权汇总，不能把各 Session 的比例按 IC 数加权后冒充被试比例。
- 数据集级 IC 分布固定为三面板：左侧横向点图显示全部被试的唯一处理 IC 数，存在多 Session 时显示平均每 Session 数；中间绘制被试级处理比例直方图与 rug（轴边短线），并标出中位数和 IQR；右侧绘制“被试×处理原因”计数热图，多 Session 时每格为该被试的 Session 等权均值。数量轴从 0 开始，比例轴固定为 0–1，热图不对每名被试标准化；被试顺序与 Bad-channel 图一致。
- 多原因 IC 按 IC 编号去重后计算唯一处理数，不把各原因计数直接相加。不同方法统一被试顺序，但保留“删除 IC”和“清理 IC”等方法真实含义。
- 单独被试的完整 ICA 诊断仅在已有数据集级结果给出明确观察理由时生成。同一被试的全部图必须来自同一个 ICA 对象或经核对的同一外部 ICA 结果，并固定按以下顺序输出：

  1. 使用 `ica.plot_components()` 展示全部 IC Topography，每页最多 20 个、固定 5 列；使用相同发散色图，每个 IC 分别以 0 为中心对称定标，并明确 IC 间色值幅度不可直接比较；被处理 IC 在标题和边框中标记。
  2. 使用 `ica.plot_scores()` 或等价的 IC×判据 score 热图展示全部可用自动 score，保留实际阈值并标记最终处理 IC；不同 score 不混用数值色标。
  3. 对每个被处理 IC 使用 `ica.plot_properties()` 输出 Topography、时间序列、Epoch image、ERP/平均时程和 PSD；不只挑代表性 IC。
  4. 使用 `ica.plot_sources()` 分页展示全部 IC source；处理前后比较使用同一段有真实判据证据的时间窗、相同纵轴和组件顺序。
  5. 使用 `ica.plot_overlay()` 展示同一真实时窗的处理前后 EEG/GFP。数量过多时分页，不缩小到无法阅读，也不删去未符合预期的 IC。

### 坏段与 Trial

- Trial 表按被试、Session、同条件和剔除原因报告原始数、各原因命中数、唯一剔除数、保留数、保留率和坏段时长比例。多原因命中不得直接相加作为唯一剔除数；图中保留精确分母。
- 保留率热图的行固定为被试，列为 Session×条件；只有一个 Session 时只保留条件。色标固定为 0–1，使用顺序色图；未采集、条件不存在、无法计算使用不同的缺失编码，格内在空间允许时标注 `保留数/原始数`。
- 条件分布图先在被试×Session×条件内计算保留率，再在被试内对同条件 Session 等权平均。每个条件显示全部被试点；同一被试跨条件以细线连接，叠加中位数和 IQR，不使用只显示箱线而隐藏原始点的图。条件顺序和颜色在各方法间固定。
- 剔除原因图按条件并排绘制两个分组条形面板：一个显示原因命中数，一个显示以该条件原始 Trial 数为分母的原因命中率。原因计数允许重叠，因此不用堆叠高度表示唯一剔除总数；唯一剔除数和总保留率单独列在表中。

### 处理前后 PSD 与其他配对比较

- 处理前后 PSD 使用质量评估模板第 2.1 节相同的通道、时间窗、Welch、单位和聚合规则。频率范围取配对前后数据共同支持的范围；只使用前后均可追溯的配对被试、Session、条件和 Trial，另表列出仅单侧可用的对象。
- 整体 PSD 与分条件 PSD 分别成图，每张固定为双面板。左侧叠加处理前、处理后的组级均值和 95% 被试级 bootstrap CI；右侧绘制每名被试在线性功率域计算后再转为 dB 的配对差值“处理后－处理前”的组均值、95% CI 和 0 dB 参考线。相同条件使用同一颜色，处理前用虚线、处理后用实线；同一比较共用频率轴，左侧处理前后共用纵轴。
- 整体图使用 0.5–100 Hz，分条件图使用 4–40 Hz，并受第 2.1 节奈奎斯特频率和既有低通限制。图注报告实际配对被试数、Session 数、Trial 数、通道数、频率分辨率和 CI 方法。
- 第 1.1 节另有适用于该方法的常规前后指标时，正文先给逐被试配对结果表，再用显示全部配对被试的连线点图；纵轴使用指标原始单位，处理前后位置固定，未配对对象不进入连线图并在表中单列。

## 运动想象任务特异图固定配置

以下为默认配置。数据集范式、所复现方法或可靠来源给出不同定义时，以其定义替换默认值，并在第 3.2.6 节配置表中记录实际值和依据；不得混用两套定义。

### Epoch、条件和聚合

- Epoch 由真实 Trial 和刺激事件定义。TFR 计算窗在报告时间窗两侧各保留至少 0.5 s 卷积边缘；计算后裁切到真实报告时间窗。数据不支持边缘扩展时保留现有范围，在图中遮罩半个最长分析窗宽度的边缘并注明。
- 所有条件使用相同的相对事件时间轴和共同分析窗。静息只有在实验范式中作为真实条件存在时才与任务条件并列，不从任务 Trial 的任意片段构造静息条件。
- 单 Session 按 Trial → 被试×条件 → 组级聚合；多 Session 先得到被试×Session×条件平均，再对同条件 Session 等权平均，最后对被试等权聚合。组级均值和 95% CI 以被试为统计单位；CI 默认使用随机种子 42 的 5000 次 bootstrap，有效被试少于 2 名时不计算。

### ERDS 时频图

- 使用 `epochs.compute_tfr(method="multitaper", freqs=np.arange(4, 41, 1), n_cycles=freqs, time_bandwidth=4.0, use_fft=True, return_itc=False, average=False, decim=1)`。上限受奈奎斯特频率或既有低通限制时降低；低于 30 Hz 时明确说明 β 覆盖不完整。
- 基线必须是范式中真实存在且不与刺激或运动想象重叠的预任务区间。对每个 Trial 使用 `apply_baseline(baseline=实际区间, mode="percent")`；ERD 为负、ERS 为正。没有有效基线时不计算 ERDS，不用整段均值或其他条件替代。
- ROI 在查看结果前依据范式和电极布局确定：左右手任务优先分别报告对侧与同侧感觉运动 ROI，足部任务优先报告中央感觉运动 ROI。数据只有 C3、Cz、C4 等少量通道时直接按真实通道报告，不构造不存在的 ROI。
- 同一任务比较中的全部预处理方法、条件和 ROI 共用一个以 0 为中心的发散色标。默认使用 `RdBu`，负值显示为红色、正值显示为蓝色；色限取全部对应组级面板有限值绝对值的第 98 百分位后向外取整为对称范围，超出范围的值保留并在色条上标出扩展端。
- 默认显著性遮罩以被试级条件结果对 0 做双侧 cluster permutation，调用 `permutation_cluster_1samp_test(..., n_permutations=5000, tail=0, seed=42, out_type="mask")`，保留校正后 `p ≤ 0.05` 的 cluster；每个预先定义的条件×ROI 面板内校正，并在多个面板间对 cluster p 值执行 Benjamini–Hochberg FDR。有效被试不足或邻接关系无法可靠建立时不显示显著性遮罩，只报告描述性结果。

### μ/β ERD/ERS 时间曲线

- 默认频带为 μ 8–12 Hz、β 13–30 Hz。先在每个 Trial 的线性功率域内对频带频点和 ROI 通道取平均，再相对同一 Trial 的真实基线计算百分比变化；随后按既定 Trial、Session、被试层级聚合。
- 使用与 ERDS 相同的时间轴、基线和边缘裁切。仅为显示采用居中的 250 ms 移动平均，不用平滑后的值计算统计检验；数据时间分辨率不足 250 ms 时不额外平滑。
- 条件颜色在所有频带、方法和图中固定；同一频带的全部方法与条件共用纵轴范围。组均值绘制实线，95% CI 绘制半透明带，刺激、运动想象和静息区间使用固定背景色并标出事件边界。

### μ/β ERD/ERS Topography

- 使用与 ERDS 相同的 Trial 级百分比功率变化，在范式定义的运动想象有效时间窗内平均；默认频带为 μ 8–12 Hz、β 13–30 Hz。各条件并排显示，不能用某一条件的峰值时窗替代共同时间窗。
- 使用 `mne.viz.plot_topomap()`，固定 `sensors=True`、`contours=6`、`image_interp="cubic"`、`extrapolate="head"`、`border="mean"`、`res=256`。同一频带在所有预处理方法和条件间共用以 0 为中心的对称色标，使用与 ERDS 相同的红负蓝正方向；不同频带分别定标。
- 组级图只使用具有真实位置且在相应比较对象中共同存在的 EEG 通道，不在组平均前填补缺失通道。电极布局不足以可靠插值时不生成 Topography，改为逐通道结果表和布局点图。

### 被试级任务效应、CSP 和跨 Session

- 被试级任务效应表至少包括被试、Session、条件、ROI、频带、分析时间窗、基线、有效 Trial 数、ERD/ERS 值和状态；组级图的每个值都必须能追溯到该表。
- CSP 仅在标签、样本量和验证设计支持时执行。CSP 必须在训练折内拟合；pattern 用于空间解释，filter 不作为头皮激活解释。不同被试的 CSP 成分没有完成可验证的成分匹配时，不对 CSP Topography 直接做跨被试平均。
- 跨 Session 可靠性分析保留被试×Session×同条件结果，使用相同参数和完整配对关系；缺失 Session 不填补，不把不完整配对纳入成对统计，并同时报告完整配对数。
