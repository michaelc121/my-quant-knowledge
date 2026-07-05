# 术语表

存放知识点下钻产生的术语文档。与主文档分开管理，便于按模块检索和持续扩充。

## 说明

- 每个术语独立成一篇 `.md` 文件，放在对应知识模块的子目录下
- 术语文件仅聚焦**一个**概念，讲深讲透
- 文件内必须注明来源文档和返回链接，保持与主文档的双向引用关系

## 索引

### basics/（基础知识）

| 术语文件 | 来源文档 | 一句话简介 |
|----------|---------|-----------|
| [jupyter.md](basics/jupyter.md) | [Python 量化工具栈速查](../basics/python-stack.md) | Jupyter 是什么、为什么适合量化、正确用法与常见误区 |
| [vscode-notebook.md](basics/vscode-notebook.md) | [Jupyter 详解](basics/jupyter.md) | VS Code 内置 Notebook 的设置、操作、与 JupyterLab 的对比 |
| [forward-adjust.md](basics/forward-adjust.md) | [量化必备金融知识](../basics/market-finance.md) | 前复权的原理、公式与代码验证 |

### data/（数据与因子）

| 术语文件 | 来源文档 | 一句话简介 |
|----------|---------|-----------|
| [ic-ir.md](data/ic-ir.md) | [因子定义与评估体系](../data/factor-intro.md) | IC（信息系数）与 IR（信息比率）的定义、计算与评估标准 |

### strategies/（策略与回测）

| 术语文件 | 来源文档 | 一句话简介 |
|----------|---------|-----------|
| [overfitting.md](strategies/overfitting.md) | [回测原理与常见陷阱](../strategies/backtest-101.md) | 过拟合的本质、检测方法与预防清单 |
| [statistical-arbitrage.md](strategies/statistical-arbitrage.md) | [策略分类框架与案例](../strategies/strategy-taxonomy.md) | 统计套利的原理、协整vs相关、配对交易完整流程与失败模式 |

### trading/（实盘与风控）

| 术语文件 | 来源文档 | 一句话简介 |
|----------|---------|-----------|
| [kelly-criterion.md](trading/kelly-criterion.md) | [资金管理与风控框架](../trading/risk-management.md) | 凯利公式推导、半凯利策略与实盘局限性 |

---

> 术语文件由学习过程中的追问触发创建，详见 [AGENTS.md 知识下钻规则](../AGENTS.md#知识下钻规则)。
