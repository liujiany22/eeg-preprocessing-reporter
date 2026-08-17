---
name: eeg-preprocessing-reporter
description: 生成 EEG 数据集信息报告或质量评估报告，每次仅生成其中一种；质量评估按任务类型路由。用于用户要求整理 EEG 数据集信息或评估 EEG 数据质量与预处理时。
---

# EEG 数据集信息与质量评估报告

## 调用与确认

1. 确定本次报告类型，只选择一种：
   - 数据集信息报告
   - 质量评估报告
2. 仅询问当前调用路径缺失的信息；用户已明确的信息不重复询问。
3. 开始处理前，用一条简短消息汇总报告类型、数据集和相关设置，请用户确认后继续。

两种报告都确认：

- 数据集名称或链接，必填。
- 是否调用 `$humanizer-zh`，默认不调用。

质量评估报告另行确认：

- 任务类型，必填；用户已明确时直接采用。
- 运动想象任务中用于个体分析和图形展示的被试；未指定时，根据数据情况自动挑选并在报告中说明。
- 运动想象任务的波形图时间参数 `x`、`y`；未指定时，根据 Trial 和事件时长自动选择。

## 调用路径

### 数据集信息报告

读取 [数据集信息](references/dataset-information.md)，仅生成数据集信息报告，不生成质量评估报告。

### 质量评估报告

根据任务类型只读取对应文件，仅生成质量评估报告，不生成数据集信息报告：

- 运动想象任务：读取 [运动想象任务](references/quality-assessment/motor-imagery.md)。
- 心算任务：读取 [心算任务](references/quality-assessment/mental-arithmetic.md)。

在 `references/quality-assessment/` 中添加文件以扩展任务类型。

## Humanizer

若用户确认调用 `$humanizer-zh`，在报告完成后按照该 skill 的说明处理报告文本。
