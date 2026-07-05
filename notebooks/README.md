# 实验笔记本

> 量化知识从"知道"到"会用"的实践空间。这里存放探索性代码、策略原型和项目归档。

## 目录约定

```
notebooks/
├── README.md                   # 本文件
├── project-01-dual-ma/         # 项目一：双均线策略全流程（第 6 周）
│   ├── dual_ma_backtest.ipynb  # 主回测笔记本
│   └── README.md               # 项目说明与结论
├── project-02-multi-factor/    # 项目二：多因子选股模型（第 8 周）
│   ├── factor_analysis.ipynb   # 因子 IC 分析
│   ├── factor_backtest.ipynb   # 分层回测
│   └── README.md
├── project-03-live-sim/        # 项目三：实盘模拟盘（第 10 周）
│   ├── binance_live.py         # 模拟盘脚本
│   └── README.md
└── scratch/                    # 临时实验，不归档
    └── .gitkeep
```

## 使用规范

- 每个项目完成后，核心代码和结论写回 `docs/` 对应模块固化
- 笔记本顶部注明日期、数据来源、运行环境
- `scratch/` 中的文件不入 git（已在 `.gitignore` 中忽略）
- 回测笔记本必须包含完整的性能指标输出，不能只有曲线图

## 三个实践项目

| 项目 | 目标 | 对应文档 | 建议时间 |
|------|------|---------|---------|
| 双均线策略全流程 | 从数据到回测报告，端到端跑通 | [strategy-taxonomy.md](../docs/strategies/strategy-taxonomy.md) | 第 6 周 |
| 多因子选股模型 | 构建 5 个因子，合成综合得分，分层回测 | [multi-factor-model.md](../docs/strategies/multi-factor-model.md) | 第 8 周 |
| 实盘模拟盘 | 在模拟盘上跑策略，观察真实市场行为 | [live-trading.md](../docs/trading/live-trading.md) | 第 10 周 |

---

> 实验完成后，将验证过的内容写回 `docs/` 固化，保持知识库的"源头活水"。
