# SD-AR 被试依赖预处理与评估流程

对应论文：*Subject-Dependent Artifact Removal for Enhancing Motor Imagery Classifier Performance under Poor Skills*。

## 方法边界

- SD-AR 的输出是每名被试在 Raw、ICA、Surface Laplacian、ICA + Surface Laplacian 四个分支中的选择，不是固定对全部被试应用同一分支。
- 分支选择依赖同一特征、分类器和验证方案下的被试内分类表现；标签或可复现的验证划分缺失时不能完成 SD-AR 选择。
- MNE 用于实现和作图；使用 MNE 等价实现时，必须记录相对论文与作者代码的差异。

## 操作流程

1. 对第 3 步保留的每名被试建立 Raw、ICA、Surface Laplacian、ICA + Surface Laplacian 四个分支；联合分支的顺序固定为 ICA 后 Surface Laplacian。
2. ICA 分支执行 1 Hz、5 阶 Butterworth 高通、白化和 FastICA。使用 `mne.preprocessing.ICA(method="fastica")` 时记录滤波 IIR 参数、成分数、随机种子和收敛状态。
3. 有 EOG 时，逐 Trial 计算每个 IC 与可用 EOG 参考的 Pearson 相关；没有 EOG 时，仅使用论文或作者代码为当前电极布局指定的额区参考。把全体被试的相关值转换为 z-score，按论文的 ±3σ 规则标记 IC，并按原方法重复两次。不得自行指定额区通道名称。
4. Surface Laplacian 分支使用球面样条 Surface Laplacian。复现论文时使用最高 Legendre 阶数 10、平滑常数 4、正则化参数 `1e-5`；若 MNE `compute_current_source_density()` 的参数化不能一一对应，记录等价关系或偏离，不声称完全复现。
5. 对四个分支使用同一 Trial 时间窗和四个频带：μ 8–12 Hz、低 β 12–15 Hz、中 β 15–20 Hz、高 β 18–40 Hz。若数据采样率或低通不支持某频带，该频带标为不可用，不调整边界冒充论文设置。
6. 按论文实现计算 Pearson correlation、motif、Gaussian FC、spectral coherence 和 phase-locking value；连接矩阵上三角向量化并拼接四个频带后输入 LDA。核函数带宽、motif degree/lag、特征缩放和验证划分使用论文或作者代码值。
7. 在完全相同的数据划分下比较四个分支；每名被试选择验证表现最高的分支。并列时使用预先写明且对全部被试一致的规则，不根据测试集结果决定。
8. 保存逐被试四分支得分、选择结果、IC 判据、Surface Laplacian 参数和实际使用的功能连接配置。

## 正文输出

SD-AR 在正文中只输出：

1. 执行状态、可用分支、实际参数、验证方案和相对论文流程的偏离；不能完成分支选择时明确停在哪一步。
2. 逐被试四分支分类表现的紧凑配对图，以及每名被试最终选择的分支；不得只显示被选中分支。
3. 全数据集四种分支的入选人数与比例，并报告并列和处理失败被试。
4. ICA 分支的逐被试唯一删除 IC 数与比例，以及全体被试的中位数、IQR、范围和有效被试数；具体 IC 明细放附录 11.5。
5. Surface Laplacian 前后以论文指定感觉运动通道为中心的空间扩散或功能连接数据集级摘要；只陈述实际观察，不预写“改善”。

## 附录输出

以下内容只放在附录 11.5：

1. 逐被试分支选择表：四分支得分、最终分支、并列处理、验证方案和处理状态。
2. 示例被试 ICA 完整诊断：全部 IC Topography、全部相关 z-score 与 ±3σ 阈值、全部排除 IC 的 MNE `ICA.plot_properties()`、`ICA.plot_sources()` 和处理前后 overlay；不得只画代表性 IC。
3. 逐被试 IC 表：IC 总数、各参考命中数、唯一删除数、删除比例、收敛状态和失败原因摘要。
4. Raw 与 Surface Laplacian 的功能连接矩阵、头皮连接图，以及论文规定的感觉运动通道空间扩散对比；比较图固定通道顺序和色标。
5. μ、低 β、中 β、高 β 下实际计算的 LDA relevance 或功能连接图；没有实现的指标不以其他指标替代。

附录 11.1 另行对四个分支中实际产生的预处理数据分别重复波形、PSD 和 Topography，不在附录 11.5 重复。

## 不生成的内容

- 不把 SD-AR 改写成数据集统一的 ICA 或 Surface Laplacian 流程。
- 不用测试集、全数据拟合结果或用户未授权的新分类器选择分支。
- 不输出论文未定义的综合质量分，不把分类提高直接写成伪迹一定减少。

## 来源

- [SD-AR 论文](https://doi.org/10.3390/s22155771)
- [SD-AR 作者代码](https://github.com/mtobonh/SD-AR)

以上链接用于约束流程，不作为固定报告章节；报告生成时按主 skill 的规定说明核查材料。
