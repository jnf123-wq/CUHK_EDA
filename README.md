# 电路划分器

本项目面向 EasyEDA Pro 原理图工程，依据器件信息、网络连接和电压特征，将电路整理为三级层级：**L0 单器件 → L1 功能簇 → L2 高/低电压域**。当前主流程为自下而上的三层划分。

## 运行环境

- Windows 10/11；
- Python 3.10 或更高版本（需要 `tkinter` 才能使用图形界面）；
- 命令行流程使用 Python 标准库；
- 若启用 L1 小器件模型补划分，需要可访问的 DeepSeek API 和 API Key。Dry-run 不需要 API Key。

项目未使用固定的本机绝对路径作为运行入口；请保留本仓库目录结构，特别是 `划分器算法部分/`、`extractors/l1/` 和 `extractors/l2/`。

## 运行方法

### 图形界面

在 Windows 上双击 `启动划分器.cmd`，选择 `.epro2` 工程、输出目录和可选的 DeepSeek 接口配置，然后点击“开始完整划分”。也可以直接运行：

```powershell
python gui.py
```

### 命令行

正常运行（需要模型时配置 API Key）：

```powershell
python run_flow.py path/to/project.epro2 `
  --api-key-file path/to/deepseek.env `
  --out path/to/output
```

仅检查本地解析和划分流程，不请求模型：

```powershell
python run_flow.py path/to/project.epro2 --dry-run --out path/to/output
```

API Key 可通过 `DEEPSEEK_API_KEY` 环境变量或接口配置文件提供。配置文件支持 `api_key`、`deepseek_api_key`、`base_url` 和 `model` 字段。模型、接口地址及 L2 电压阈值也可分别通过 `--model`、`--base-url`、`--threshold-v` 指定；默认电压阈值为 12 V。请勿把真实 API Key 提交到 GitHub。

## 划分思路

1. **读取与提取**：针对 `.epro2` 工程分别调用 L1、L2 专用提取器。L1 提取器提供功能划分所需的器件、拓扑及原理图位置信息；L2 提取器提取全局网络标签和电压证据。开始划分前会核对两边的器件 ID 集合。
2. **L0 单器件层**：为每个器件建立独立基础节点，并识别连接器标记；FPC 类器件也按连接器识别规则处理。
3. **L1 功能层**：先由确定性算法根据器件角色和连接拓扑生成初始功能簇。连接器和机械支撑器件各自单独成簇。对于符合条件的小器件，DeepSeek 只负责补充划分到已有大器件锚点簇的候选；程序校验目标合法性及原理图同页、距离不超过 4 英寸的限制。流程不运行 LLM 功能识别。
4. **L2 电压域层**：将完整 L1 簇作为不可拆分单元，根据簇内核心芯片的电源输入引脚和网络名称判断高、低压域。混合证据保留为未决边界；L2 不重新拆分 L1。
5. **校验和导出**：检查器件在各层归属完整且唯一，并校验 L2 → L1 → L0 父子成员能够完整重建。

Dry-run 会跳过 DeepSeek 小器件补划分，但仍执行确定性 L1 初划分和本地 L2 电压规则，因此适合检查输入和流程，不代表模型补划分结果。

## 主要输出

- `hierarchy.json`：完整层级树及归属关系；
- `hierarchical_assignments.csv`：每个器件对应的 L0、L1、L2 ID 和连接器标记；
- `L0_assignments.csv`、`L1_assignments.csv`、`L2_assignments.csv`：分层归属表；
- `l1/`、`l2/`：各阶段提取、划分、证据和审计文件；
- `run.json`：运行状态、参数、统计和使用的提取器信息。

多板 `.epro2` 工程会按工程内板级关系分别划分，并在输出中提供板级目录与汇总层级结果。

## 项目结构

```text
run_flow.py                 主命令行入口
gui.py                      图形界面
l1_flow.py                  L1 初划分与小器件补划分
l1_voltage_aggregate.py     L2 电压域聚合
extractors/l1/              L1 专用 .epro2 提取器
extractors/l2/              L2 专用 .epro2 提取器
划分器算法部分/             确定性功能划分算法
tests/                      测试
```

## 注意事项

- `.epro2` 是当前主流程支持的输入格式。
- 划分结果是基于提取信息和规则生成的工程辅助结果，仍建议结合原理图、器件数据手册进行审核。
- Dry-run 输出不包含 DeepSeek 小器件补划分结果。
