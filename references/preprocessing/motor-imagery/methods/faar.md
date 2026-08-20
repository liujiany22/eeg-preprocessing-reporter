# FAAR Epoch 质量评估流程

对应论文：*From EEG Cleaning to Decoding: The Role of Artifact Rejection in MI-based BCIs*。

## 方法边界

- FAAR 是 Epoch 级伪迹拒绝与质量评估方法，不替代连续数据的坏道修复、重参考或 ICA。
- 只处理第 3 步保留被试中具有可验证运动想象事件边界的数据。
- 特征标准化、序数等级、通道聚合和 knee detection 必须使用论文全文或作者实现中的确切定义。无法取得足以复现的定义时，停止在相应步骤并标为未执行，不自行设定公式或阈值。

## 预处理操作流程

1. 使用 MNE 对原始 EEG 执行 8–32 Hz 带通滤波，按真实运动想象事件构造 Epoch；记录 `l_freq`、`h_freq`、`tmin`、`tmax`、事件映射和边界处理方式。
2. 对每个 Epoch、每个通道计算五项特征：8–32 Hz 平均 FFT 幅值、RMS、相邻采样点最大绝对梯度、过零率和归一化四阶中心矩。计算代码、单位和数组维度写入运行记录。
3. 将同一被试与 Session 的连续数据切成 1 s 参考窗，逐通道计算 RMS 并在通道内按窗标准化；严格按论文或作者实现的 truncated-Gaussian inlier criterion 选择参考窗，保存参考窗掩码和每通道参考分布。
4. 按论文或作者实现将 Epoch 特征相对参考分布标准化、映射到序数严重度，并聚合成 Epoch 级 SQI；SQI 越大表示质量越差。保存逐特征、逐通道分数和最终 SQI。
5. 对每个被试与 Session 的 SQI 经验分布分别运行论文指定的 Kneeliverse knee detection；保存算法名、参数、版本、膝点和保留或剔除标签。不得用全数据集统一阈值替代。
6. 统计每个被试、Session 和条件的原始 Epoch 数、剔除数、保留数及剔除率，并核查剔除是否集中于特定条件。
## 预处理报告输出

FAAR 在预处理报告中只输出：

1. 执行状态、适用的被试与 Session、实际 Epoch 定义、实现来源和实际参数；未执行步骤及缺失项。
2. 每个被试、Session 和条件的原始 Epoch 数、剔除数、保留数及剔除率。

## 质量评估步骤

1. 使用预处理保存的逐特征、逐通道分数、SQI、knee threshold 和保留或剔除标签生成质量评估结果，不重新运行 Epoch 拒绝。
2. 只有用户明确要求解码验证且当前数据具有所需标签与分组时，才按论文的 covariance estimation → tangent-space projection → logistic regression 执行；划分必须按被试或 Session 隔离，FAAR 参考分布和阈值只在训练折内拟合。

## 质量评估报告输出

FAAR 在正文中只输出：

1. 执行状态、适用的被试与 Session、实际 Epoch 定义、实现来源和实际参数；未执行步骤及缺失项。
2. 全数据集 SQI 分布摘要和剔除率摘要；不得把较高 SQI 直接解释成具体伪迹类型。
3. 逐被试与 Session 的剔除率图；显示全部成功单位，并报告中位数、IQR、范围和有效单位数。
4. 按条件的原始、剔除和保留 Epoch 数；没有预先判据时只报告条件差异，不判断偏倚是否可接受。
5. 仅在执行了解码验证时，报告与不拒绝基线的 Balanced Accuracy、F1、Precision 和跨被试离散程度；结论限定在实际验证方案内。

## 质量评估报告附录

以下内容只放在附录 11.4：

1. 每个被试与 Session 的 SQI 直方图或密度图，标出实际 knee threshold，并区分保留与剔除 Epoch。
2. 逐被试、Session、条件的 Epoch 明细表；字段固定为原始数、剔除数、保留数、剔除率、阈值和处理状态。
3. 实际使用的五项特征定义、序数映射、聚合公式、Kneeliverse 参数和核心代码。
4. 执行解码验证时，放置逐被试基线与 FAAR 配对图及完整指标表；不执行时不保留空白分类章节。

附录 11.1 另行生成保留 Epoch 支持的 Trial 波形、PSD 和 Topography；不重建整段连续波形，也不在附录 11.4 重复。

## 不生成的内容

- 不自动比较 AutoReject、Isolation Forest 或其他拒绝方法。
- 不把解码性能当作 FAAR 必然改善数据的证据，也不根据论文总体结果预写当前数据集结论。
- 不输出 ICA、Bad-channel metric、连续数据清理结果或具体伪迹类别。

## 来源

- [FAAR 预印本](https://arxiv.org/abs/2605.12408)

该链接用于约束流程，不作为固定报告章节；报告生成时按主 skill 的规定说明核查材料。
