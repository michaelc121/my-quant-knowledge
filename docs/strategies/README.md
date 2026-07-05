# 策略与回测

量化研究的第三层。了解策略类型，掌握回测方法，才能把数据和因子转化为可执行的交易逻辑。

## 本篇目标

- 理解四大策略范式的原理与适用场景
- 掌握回测设计原则，避免常见陷阱
- 学会用 Python 实现完整的回测流程
- 读懂回测报告中的数据（夏普、回撤、胜率）
- 通过具体策略案例（海龟交易法）串联全流程

## 文档索引

| 文件 | 内容 | 建议时间 |
|------|------|----------|
| [strategy-taxonomy.md](strategy-taxonomy.md) | 策略分类框架与案例 | 第 6 周 |
| [backtest-101.md](backtest-101.md) | 回测原理、常见陷阱与评估指标 | 第 6 周 |
| [turtle-trading.md](turtle-trading.md) | 海龟交易法完整拆解 | 第 7 周 |
| [multi-factor-model.md](multi-factor-model.md) | 多因子选股模型实战 | 第 8 周 |
| [strategy-evaluation.md](strategy-evaluation.md) | 策略评价与归因诊断（含 Brinson 归因 + 策略诊断流程） | 第 9 周 |
| [strategy-rd.md](strategy-rd.md) | 策略研发方法论 | 第 9 周 |

> **前置提醒**：本节内容需要调用数据、计算因子——栈回 [`docs/data/`](../data/) 查阅具体方法。
