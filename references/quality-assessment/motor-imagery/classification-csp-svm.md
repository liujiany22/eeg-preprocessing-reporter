# CSP+SVM 分类配置

## 使用范围

- 本配置只用于运动想象质量评估报告第 5 节，以统一简单模型查看预处理前原始数据和各预处理结果的分类可分性。
- 逐被试独立建模，不将不同被试的 Trial 混合训练同一个模型。只纳入数据集正式定义的运动想象目标类别；静息仅在数据集将其定义为正式分类目标时纳入。
- 使用实际可追溯的 Trial、标签和数据。分类性能不作为数据质量、预处理优劣或被试排除的单独判据。

## 共同比较输入

1. 数据来源包括预处理前原始数据、每种实际可用的预处理结果，以及“预处理前原始数据 + RELAX 数据 double”。
2. 主表各行使用共同被试、共同类别、共同原始 Trial ID、共同 EEG 通道和共同相对事件时间窗。共同 Trial 集取各实际比较来源可追溯 Trial ID 的交集；同时报告每个来源原有 Trial 数和进入共同集的 Trial 数，不把来源缺失的 Trial 补为 0。
3. 通道按统一标准顺序排列，只保留各比较来源共同存在的实际 EEG 通道。共同通道少于 2 个、数据秩小于 2、少于 2 个类别，或任一类别的有效 Trial 少于 2 个时不运行，并报告原因。
4. 默认分析窗为运动想象真实开始后 0.5–3.5 s。实际范式的稳定运动想象阶段不支持该范围时，在查看分类结果前依据范式确定所有来源和类别共同支持的单一时间窗；不得根据分类分数挑选时间窗。
5. 所有来源使用同一采样率，默认取 `min(250 Hz, 各来源实际采样率的最小值)`。需要降采样时先完成抗混叠低通并核对事件和 Trial 样点。
6. 所有来源都执行相同的 8–30 Hz 分类前带通。滤波前核对各来源的奈奎斯特频率、采集低通和可用过渡带；共同输入无法完整支持 30 Hz 时，在查看分类结果前为全部来源确定同一个可支持上限并标明相对 8–30 Hz 的偏离，无法形成有效共同通带时不运行。默认调用 `inst.filter(l_freq=8, h_freq=30, picks="eeg", method="fir", phase="zero", fir_design="firwin", filter_length="auto", l_trans_bandwidth="auto", h_trans_bandwidth="auto")`，并保存 MNE 实际确定的 transition bandwidth 和 filter length。能够从连续数据构造时先在连续数据上滤波再切 Epoch；只有 Epoch 时在分析窗两侧保留滤波边缘，滤波后裁切到共同时间窗。

## 模型参数

使用同一个 scikit-learn `Pipeline`，并在每个训练折内依次拟合：

```python
Pipeline([
    ("csp", CSP(
        n_components=actual_n_components,
        reg=None,
        log=True,
        cov_est="concat",
        transform_into="average_power",
        norm_trace=False,
        component_order="mutual_info",
        rank=None,
    )),
    ("scale", StandardScaler()),
    ("svm", SVC(
        kernel="linear",
        C=1.0,
        class_weight="balanced",
        probability=False,
        decision_function_shape="ovr",
    )),
])
```

- `actual_n_components = min(4, 共同通道数, 各比较来源的最小可用数据秩)`。数据秩只按数据结构和 MNE 秩估计确定，不使用类别标签或分类结果；同一被试的所有数据来源使用相同组件数。最多 4 个组件用于保持简单基线，不通过测试折或全数据分类分数选择组件数。
- `reg=None` 使用经验协方差，与经典 CSP 方案保持接近；若某一训练折因协方差或秩问题无法拟合，记录为失败，不单独为该来源改用其他正则化。
- `norm_trace=False` 沿用 MNE 运动想象 CSP 示例的配置。经典 CSP 包含 trace normalization，但 MNE 文档提示该步骤在后续实现中通常不采用并可能影响 pattern 排序；本基线不把它设为可调参数。
- CSP 输出对数平均功率；`StandardScaler`、CSP 和 SVM 都只在训练折拟合。SVM 固定使用线性核、`C=1.0` 和训练折内计算的平衡类别权重，不搜索 C、核函数或其他超参数。
- 二分类和多分类使用同一实现。多分类采用 MNE CSP 的多分类实现；SVC 内部使用 one-vs-one 训练，并以 `decision_function_shape="ovr"` 输出类别分数。

## 交叉验证

1. 每名被试单独建立划分，并把同一划分映射到所有数据来源。划分单位是原始 Trial ID，任何由同一 Trial 派生的数据都必须进入同一折。
2. 被试具有至少 2 个可用 Session 或 Run，且按 Session 或 Run 分组后每个训练折和测试折均包含全部类别时，使用 `StratifiedGroupKFold(n_splits=min(5, 实际组数), shuffle=True, random_state=42)`：优先以 Run、没有 Run 时以 Session 为 `groups`。
3. 分组交叉验证不满足类别覆盖时，使用 `StratifiedKFold(n_splits=min(10, 各类别最小 Trial 数), shuffle=True, random_state=42)`。折数不得少于 2；报告此时结果属于被试内 Trial 级交叉验证，不能解释为跨 Session 泛化。
4. CSP、标准化和 SVM 全部在训练折内拟合。除“模型参数”中仅依据共同通道数和数据秩确定组件数上限外，不得先用全体 Trial 拟合 CSP、根据分类结果选择组件、选择通道、选择时间窗或调参后再做交叉验证。
5. 各折保存训练和测试的原始 Trial ID、类别数、训练样本数、测试 Trial 数、失败原因和预测结果。

## 预处理与交叉验证隔离

1. 任何依据当前数据估计的阈值、参考分布、坏道或 Trial 选择、ICA 模型、特征变换或方法分支，只能在外层交叉验证的训练数据中拟合，再以冻结参数应用于对应测试数据。固定采集参数、预先确定的事件定义和不使用数据估计的常量不受此限制。
2. FAAR 的参考分布、特征标准化和 knee threshold 在每个外层训练折内确定，再用于测试折；测试折不参与阈值拟合。
3. SD-AR 使用嵌套交叉验证：只在外层训练折内部比较并选择 Raw、ICA、Surface Laplacian 或联合分支，再将选定分支应用于外层测试折。外层测试结果不得参与分支选择或并列处理。
4. 其他预处理方法中由数据学习的步骤同样按训练折拟合。若只能取得在全量数据上拟合后保存的预处理结果，该行仍可运行，但必须标为“全量预处理后的探索性结果”，不得与严格折内处理的行作确认性优劣比较。
5. “预处理前原始数据 + RELAX 数据 double”只有在 RELAX 的数据依赖步骤按训练折隔离时才属于严格交叉验证结果；否则按上一条标为探索性结果。

## double 数据

1. 只使用同时具有预处理前原始版本和 RELAX 版本的成对原始 Trial。两个版本使用相同标签、通道、采样率和时间窗。
2. 训练折内把每个原始 Trial 的两个版本沿样本维拼接，因此训练样本数为成对原始 Trial 数的 2 倍。CSP、标准化和 SVM 在这个加倍后的训练折上拟合。
3. 测试折也生成每个原始 Trial 的两个版本。二分类时平均两个版本的有符号 `decision_function` 分数后按 0 判定；多分类时平均两个版本的 OVR 类别分数后取最大值类别。每个原始 Trial 只产生一个最终预测并只计一次指标。
4. double 行与其他行使用相同的原始 Trial 折划分。不得让同一原始 Trial 的一个版本进入训练折、另一个版本进入测试折。

## 结果统计

- 主要指标为 Balanced Accuracy；同时报告 Accuracy 和 Macro-F1。全部指标根据测试折原始 Trial 级预测计算。
- 先汇总每名被试的全部测试折预测，再对被试等权计算数据集结果。多次覆盖同一 Trial 时先在被试内对该 Trial 的 decision score 求平均，得到一个最终预测后再计算被试指标。
- 95% CI 只对主要指标计算，使用固定随机种子 `42` 的 5000 次被试级 bootstrap；有效被试少于 2 名时不计算 CI。
- 主表报告成功和失败被试、类别、每个来源原有及进入共同集的原始 Trial 数、训练样本数范围、`1 / 类别数` 的 Balanced Accuracy chance level、三个指标、Balanced Accuracy 的 95% CI 和执行状态。逐被试、逐折及预测明细放入附录。

## 来源

- [Ramoser、Müller-Gerking 与 Pfurtscheller：经典 CSP 运动想象分类](https://doi.org/10.1109/86.895946)
- [Yi 等：8–30 Hz、CSP 与 SVM 的运动想象分类](https://doi.org/10.1186/1743-0003-10-106)
- [MNE CSP 运动想象示例](https://mne.tools/stable/auto_examples/decoding/decoding_csp_eeg.html)
- [MNE CSP API](https://mne.tools/stable/generated/mne.decoding.CSP.html)
- [scikit-learn SVC](https://scikit-learn.org/stable/modules/generated/sklearn.svm.SVC.html)
- [scikit-learn StratifiedGroupKFold](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.StratifiedGroupKFold.html)
- [scikit-learn StratifiedKFold](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.StratifiedKFold.html)
