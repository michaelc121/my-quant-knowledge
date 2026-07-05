# 数据与因子

量化研究的第二层。理解数据的生成规则，才能正确地计算因子、评估信号。

## 本篇目标

- 掌握 K 线数据的形成原理与规范
- 理解复权的深层机制（adj_factor、成交量调整）
- 学会处理缺失值、异常值、期货换月
- 理解因子的定义、分类与评估方法
- 建立从数据清洗到因子计算的完整 pipeline

## 文档索引

| 文件 | 内容 | 建议时间 |
|------|------|----------|
| [bar-conventions.md](bar-conventions.md) | K 线规范与数据清洗 | 第 3 周 |
| [factor-intro.md](factor-intro.md) | 因子定义与评估体系 | 第 4 周 |
| [technical-indicators.md](technical-indicators.md) | 常用技术指标速查（ATR/ADX/MACD/布林带/RSI/乖离率） | 第 3 周 |
| [factor-library.md](factor-library.md) | 经典因子公式与代码 | 第 4 周 |
| [neutralization.md](neutralization.md) | 行业中性化与市值中性化 | 第 5 周 |
| [data-pipeline.md](data-pipeline.md) | 数据获取→清洗→存储工程化 | 第 3 周 |
| [factor-discovery.md](factor-discovery.md) | 因子挖掘方法论 | 第 5 周 |

> **前置提醒**：本节内容建立在 docs/basics/ 基础之上，如果在阅读中遇到陌生的金融概念（如复权、除权除息），回到 [`docs/glossary/basics/`](../glossary/basics/) 查阅对应术语。
