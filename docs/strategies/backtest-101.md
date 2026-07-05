# 回测原理、常见陷阱与评估指标

> **前置要求**：已阅读 strategy-taxonomy.md，理解四大策略范式。
> **内容概览**：回测的核心逻辑、三大陷阱、评估指标体系、Python 向量化回测实现。
> **更新日期**：2026-07-02

---

## 一、回测在做什么

回测就是用历史数据模拟策略过去的表现，回答三个问题：

```
1. 这个策略历史上赚钱吗？     →  累计收益曲线
2. 赚钱的代价是什么？         →  最大回撤、波动率
3. 运气成分有多大？           →  样本外测试、参数敏感性
```

### 回测的最小单元

```
t 日收盘 → 产生信号 → t+1 日开盘执行 → 持有 → t+n 日卖出
```

**关键约束**：t 日的信号只能用到 t 日收盘及之前的数据。任何用到 t+1 日数据的信号都是**作弊**。

---

## 二、向量化回测（最简形式）

向量化回测一次性对整段历史计算信号和收益，不逐行循环。适合策略快速原型：

```python
import pandas as pd
import numpy as np

def vectorized_backtest(
    close: pd.Series,
    signal: pd.Series,
    cost_rate: float = 0.001
) -> pd.DataFrame:
    """
    向量化回测
    - signal: 1（持仓）或 0（空仓），t 日收盘生成
    - cost: 单边交易成本比例
    """
    df = pd.DataFrame({"close": close, "signal": signal})

    # 每日收益率
    df["daily_ret"] = df["close"].pct_change()

    # 策略收益：信号在 t 日生成，t+1 日执行（shift(1)）
    df["strategy_ret"] = df["signal"].shift(1) * df["daily_ret"]

    # 交易成本：信号变化时产生交易
    df["trade"] = (df["signal"] != df["signal"].shift(1)).astype(int)
    df["cost"]  = df["trade"] * cost_rate * df["signal"].abs()
    df["strategy_ret_net"] = df["strategy_ret"] - df["cost"]

    # 累计收益
    df["cum_bench"]  = (1 + df["daily_ret"]).cumprod()
    df["cum_strat"]  = (1 + df["strategy_ret_net"]).cumprod()

    return df


# 使用示例
signal = (close.rolling(5).mean() > close.rolling(20).mean()).astype(int)
result = vectorized_backtest(close, signal)
```

**为什么 shift(1)**：t 日收盘产生的信号，最早在 t+1 日开盘执行。没有 shift(1) 就产生了前视偏差。

---

## 三、三大回测陷阱

### 陷阱 1：前视偏差

最常见的回测错误。表现为在信号中用了未来才知的数据。

```python
# ❌ 错误：用了当天收盘价做买入信号，同时用当天收盘价算收益
df["signal"] = (df["close"] > df["close"].shift(1))
df["daily_ret"] = df["close"].pct_change()
df["strategy_ret"] = df["signal"] * df["daily_ret"]   # 同一天，作弊！

# ✓ 正确：t 日信号 → t+1 日执行
df["signal"] = (df["close"] > df["close"].shift(1))
df["daily_ret"] = df["close"].pct_change()
df["strategy_ret"] = df["signal"].shift(1) * df["daily_ret"]   # shift(1)
```

### 陷阱 2：幸存者偏差

只在今日还活着的股票上回测，忽略了在历史中已经退市的。

**影响**：高估收益（退市股票通常表现极差）。

**修正**：回测时使用**当时的成份股列表**（比如 2020 年回测就用 2020 年的指数成份股，不是 2026 年的）。

### 陷阱 3：忽略交易成本

手续费 + 滑点 + 冲击成本，足以让一个日频策略的收益腰斩。

```python
# 不同频率下的成本差异（单边按 1‰ 估算）
# 日频（一年 250 笔交易）：成本 ≈ 250 × 0.1% = 25% 的年化损耗
# 周频（一年 50 笔交易）：  成本 ≈ 50 × 0.1%  = 5% 的年化损耗
# 月频（一年 12 笔交易）：  成本 ≈ 12 × 0.1%  = 1.2% 的年化损耗
```

---

## 四、评估指标体系

> 更深入的绩效归因方法见 [策略评价与绩效归因](strategy-evaluation.md)。


### 核心指标

```python
def backtest_metrics(result: pd.DataFrame, rf_rate: float = 0.02) -> dict:
    """
    计算回测评估指标
    result: vectorized_backtest 的返回值
    rf_rate: 无风险利率（年化）
    """
    strat_rets = result["strategy_ret_net"].dropna()
    bench_rets = result["daily_ret"].dropna()

    total_days = len(strat_rets)
    trading_days = 252

    # 年化收益
    total_return = result["cum_strat"].iloc[-1]
    years = total_days / trading_days
    annual_return = total_return ** (1 / years) - 1

    # 年化波动率
    annual_vol = strat_rets.std() * np.sqrt(trading_days)

    # 夏普比率
    sharpe = (annual_return - rf_rate) / annual_vol if annual_vol > 0 else 0

    # 最大回撤
    cum = result["cum_strat"]
    peak = cum.expanding().max()
    drawdown = (cum - peak) / peak
    max_drawdown = drawdown.min()

    # 卡玛比率
    calmar = annual_return / abs(max_drawdown) if max_drawdown != 0 else 0

    # 胜率
    winning_trades = (strat_rets > 0).sum()
    total_trades = (strat_rets != 0).sum()
    win_rate = winning_trades / total_trades if total_trades > 0 else 0

    # 盈亏比
    avg_win = strat_rets[strat_rets > 0].mean() if (strat_rets > 0).any() else 0
    avg_loss = abs(strat_rets[strat_rets < 0].mean()) if (strat_rets < 0).any() else 1
    profit_loss_ratio = avg_win / avg_loss if avg_loss > 0 else 0

    return {
        "年化收益":     f"{annual_return:.2%}",
        "年化波动率":   f"{annual_vol:.2%}",
        "夏普比率":     f"{sharpe:.2f}",
        "最大回撤":     f"{max_drawdown:.2%}",
        "卡玛比率":     f"{calmar:.2f}",
        "胜率":         f"{win_rate:.2%}",
        "盈亏比":       f"{profit_loss_ratio:.2f}",
        "总交易次数":   total_trades,
        "回测天数":     total_days
    }
```

### 各指标含义速查

| 指标 | 含义 | 合格线 | 好 | 优秀 |
|------|------|--------|----|------|
| 年化收益 | 投一年赚多少 | > 0% | > 10% | > 20% |
| 夏普比率 | 每单位风险换多少收益 | > 0.5 | > 1.0 | > 2.0 |
| 最大回撤 | 最惨的时候亏多少 | < 30% | < 15% | < 10% |
| 卡玛比率 | 收益/最大回撤 | > 0.5 | > 1.0 | > 2.0 |
| 胜率 | 赚钱的交易比例 | > 30% | > 50% | > 60% |
| 盈亏比 | 平均赚 / 平均亏 | > 1.0 | > 1.5 | > 2.0 |

### 不要只看一个指标

- 年化收益 50% + 最大回撤 60% → 迟早爆仓（夏普低）
- 胜率 80% + 盈亏比 0.5 → 一次大的亏损覆盖十次小赚（趋势跟踪的反面）
- **夏普 + 最大回撤 + 收益曲线** 三个一起看，能排除大多数垃圾策略

---

## 五、样本内 vs 样本外

把数据分成两部分：

```
训练集（样本内）          测试集（样本外）
2020-01-01 ~ 2023-12-31   2024-01-01 ~ 2024-12-31
↑ 参数在这里调优          ↑ 只用一次，作为最终验证
```

```python
def train_test_split(df, split_date="2024-01-01"):
    train = df[df.index < split_date].copy()
    test  = df[df.index >= split_date].copy()
    return train, test

# 用法：
# 1. 在 train 上调参数
# 2. 固定参数后跑 test
# 3. test 上的表现 = 策略的真实预期
```

**原则**：样本外只跑一次。如果在样本外表现不好，说明策略[过拟合](../glossary/strategies/overfitting.md)了，回去重新设计。

---

## 六、参数敏感性

一个策略的好坏不应该取决于一组精确的参数：

```python
def parameter_sweep(close, ma_range=(5, 60)):
    """扫描不同参数组合，观察策略表现分布"""
    results = []
    for short in range(ma_range[0], ma_range[1], 5):
        for long in range(short + 10, ma_range[1] + 10, 5):
            signal = (close.rolling(short).mean() > close.rolling(long).mean()).astype(int)
            bt = vectorized_backtest(close, signal, cost_rate=0.001)
            metrics = backtest_metrics(bt)
            results.append({
                "short": short, "long": long,
                "sharpe": float(metrics["夏普比率"]),
                "annual_return": float(metrics["年化收益"].strip("%")),
                "max_drawdown": float(metrics["最大回撤"].strip("%"))
            })
    return pd.DataFrame(results)
```

**观察点**：
- 最好的几组参数和周围的参数差距大吗？ → 如果差距大说明过拟合
- 大部分参数组合都亏钱，只有一组赚钱？ → 碰巧的，不要相信

---

## 七、回测报告模板

```python
def generate_backtest_report(result: pd.DataFrame, name: str = "策略"):
    """生成回测报告（文字 + 图表）"""
    metrics = backtest_metrics(result)

    print(f"{'='*40}")
    print(f"  {name} 回测报告")
    print(f"{'='*40}")
    for k, v in metrics.items():
        print(f"  {k}: {v}")

    # 绘制收益曲线对比
    import matplotlib.pyplot as plt
    plt.figure(figsize=(12, 6))
    plt.plot(result.index, result["cum_strat"], label="策略", linewidth=1)
    plt.plot(result.index, result["cum_bench"], label="基准", linewidth=1, alpha=0.6)
    plt.legend()
    plt.title(f"{name} — 夏普={metrics['夏普比率']}")
    plt.grid(alpha=0.3)
    plt.show()

    # 绘制回撤曲线
    cum = result["cum_strat"]
    peak = cum.expanding().max()
    drawdown = (cum - peak) / peak
    plt.figure(figsize=(12, 3))
    plt.fill_between(drawdown.index, 0, drawdown.values, color="red", alpha=0.3)
    plt.title(f"回撤曲线 — 最大回撤={metrics['最大回撤']}")
    plt.grid(alpha=0.3)
    plt.show()
```

---

## 八、回测检查清单

每次回测完，对照这个清单过一遍：

- [ ] 是否用了 shift(1) 来避免前视偏差？
- [ ] 是否包含了交易成本（佣金 + 滑点）？
- [ ] 是否做了样本外测试？
- [ ] 参数敏感性分析是否通过？
- [ ] 最大回撤是否在可接受范围内？
- [ ] 去掉最好的 N 天，策略是否还赚钱？（运气排除）
- [ ] 换一个时间段（如牛市转熊市），策略是否还稳定？

> 全部打 ✓ 的策略才值得考虑实盘。

---

> **进阶阅读**：[策略研发方法论](strategy-rd.md) —— 从回测到实盘的全流程工程框架。
> **进阶阅读**：[策略评价与归因诊断](strategy-evaluation.md) —— 回测跑完后回答钱从哪来和是否可持续。 —— 从回测到实盘的全流程工程框架。

> **动手练习**：用 strategy-taxonomy.md 末尾的双均线信号，在本篇的 `vectorized_backtest` 上跑一次完整回测。然后调参、加成本、跑样本外，生成一份回测报告。
