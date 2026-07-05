# 收益率计算：从日收益到年化收益

> **前置要求**：已阅读 statistics-primer.md，理解均值和标准差。
> **内容概览**：简单收益率 vs 对数收益率、复利效应、年化公式、分红再投资假设。
> **更新日期**：2026-07-02

---

收益率计算是量化策略中最基础的运算，几乎所有评估指标都建立在它上面。但其中有几个容易犯错的细节，直接影响回测结果的准确性。

## 一、简单收益率 vs 对数收益率

```python
import numpy as np

price_today = 10.5
price_yesterday = 10.0

# 简单收益率（算术收益率）
simple_ret = (price_today - price_yesterday) / price_yesterday  # 0.05 = 5%

# 对数收益率（连续复利收益率）
log_ret = np.log(price_today / price_yesterday)  # 0.04879 = 4.879%
```

### 什么时候用哪个

| | 简单收益率 | 对数收益率 |
|---|---|---|
| **时间可加性** | ✗ 多期不能直接相加 | ✓ 多期直接相加 = 累计收益 |
| **组合可加性** | ✓ 多资产可直接加权平均 | ✗ 不能直接加权平均 |
| **正态分布假设** | 左偏（最小 -100%） | 对称，接近正态 |
| **实际盈亏计算** | ✓ 直接对应金额 | ✗ 需 exp 转回 |
| **常见用途** | 组合收益计算、回测收益曲线 | 统计检验、风险模型、VaR |

```python
# 对数收益率的核心优势：多期累加
daily_log_rets = np.random.normal(0.001, 0.02, 252)
cum_log_ret = daily_log_rets.sum()  # 直接相加 = 年收益（连续复利）

# 简单收益率必须连乘
daily_simple_rets = np.exp(daily_log_rets) - 1
cum_simple_ret = (1 + daily_simple_rets).prod() - 1  # 必须连乘

print(f"对数: {cum_log_ret:.4f}, 简单: {cum_simple_ret:.4f}")
# 两者近似但不完全相同，差异在小数点后第三位开始显现
```

---

## 二、日收益 → 年化收益

### 公式

```python
# 年化收益率
annual_return = (1 + daily_return).prod() ** (252 / n_days) - 1

# 年化波动率
annual_vol = daily_return.std() * np.sqrt(252)

# 年化夏普比率（假设无风险利率 r_f=3%）
r_f_annual = 0.03
sharpe = (annual_return - r_f_annual) / annual_vol
```

### 为什么是 252 而非 365

| 市场 | 年交易日 | 月交易日 |
|------|---------|---------|
| A 股 | ~244 | ~20 |
| 美股 | ~252 | ~21 |
| 港股 | ~247 | ~20.5 |
| 加密货币 | 365 | 30 |

**使用交易天数而非日历天数**，是因为非交易日没有新的信息进入，波动率在跨假期时不变。

```python
def annualize_returns(daily_rets, trading_days=252):
    """将日收益率序列年化"""
    daily_rets = daily_rets.dropna()
    n = len(daily_rets)
    
    cum_return = (1 + daily_rets).prod() - 1
    annual_return = (1 + cum_return) ** (trading_days / n) - 1
    annual_vol = daily_rets.std() * np.sqrt(trading_days)
    max_drawdown = compute_max_drawdown(daily_rets)  # 最大回撤不年化
    
    return {
        "年化收益率": annual_return,
        "年化波动率": annual_vol,
        "夏普比率": (annual_return - 0.03) / annual_vol,
        "最大回撤": max_drawdown,
    }
```

---

## 三、复利 vs 单利：收益曲线的正确画法

```python
def compute_cumulative_return(daily_rets):
    """
    从日收益率计算累计收益曲线。
    
    注意：必须用 (1+r) 连乘，不能用 cumsum！
    cumsum 只对对数收益率有效。
    """
    return (1 + daily_rets).cumprod()

# 常见错误：对简单收益率用 cumsum
# wrong_curve = daily_rets.cumsum()  # ❌ 不考虑复利效应
```

**错误的影响**：5 年下来，cumsum 会低估收益约 2-5%，回撤也会偏小。

---

## 四、分红再投资假设

收益率计算中最容易忽视的概念差异：

```python
# 场景：股票 10 元，分红 0.5 元，除权后价格 9.5 元
# 复权后的"收益率"包含了分红再投资的假设

# 前复权收益率 = (P_adj[t] - P_adj[t-1]) / P_adj[t-1]
# 这个收益率对应着：你在 t-1 买入，经历分红后，用分红现金在除权日再买入

# 如果你在实盘中不执行分红再投资：
# 实际收益率 = (P[t] + 分红到手) / P[t-1] - 1
# 两者差异 = 分红金额在你择时下的机会成本
```

**对回测的启示**：前复权收益率假设了完美的分红再投资——这在散户实盘中几乎做不到。保守的回测应该考虑分红现金的"闲置"成本。

---

## 五、完整收益率计算工具函数

```python
import pandas as pd
import numpy as np


def compute_max_drawdown(daily_rets):
    """计算最大回撤"""
    cum = (1 + daily_rets).cumprod()
    rolling_max = cum.cummax()
    drawdown = cum / rolling_max - 1
    return drawdown.min()


def returns_summary(daily_rets, label="策略", trading_days=252, r_f=0.03):
    """
    收益率汇总表——回测报告的标准输出
    """
    daily_rets = daily_rets.dropna()
    n = len(daily_rets)
    
    cum_return = (1 + daily_rets).prod() - 1
    annual_return = (1 + cum_return) ** (trading_days / n) - 1
    annual_vol = daily_rets.std() * np.sqrt(trading_days)
    sharpe = (annual_return - r_f) / annual_vol
    max_dd = compute_max_drawdown(daily_rets)
    
    # 卡尔玛比率 = 年化收益 / 最大回撤（绝对值）
    calmar = annual_return / abs(max_dd) if max_dd != 0 else np.nan
    
    # 胜率与盈亏比
    win_rate = (daily_rets > 0).mean()
    avg_win = daily_rets[daily_rets > 0].mean() if (daily_rets > 0).any() else 0
    avg_loss = abs(daily_rets[daily_rets < 0].mean()) if (daily_rets < 0).any() else 0
    profit_loss_ratio = avg_win / avg_loss if avg_loss > 0 else np.nan
    
    return pd.Series({
        "累计收益": f"{cum_return:.2%}",
        "年化收益": f"{annual_return:.2%}",
        "年化波动": f"{annual_vol:.2%}",
        "夏普比率": f"{sharpe:.2f}",
        "最大回撤": f"{max_dd:.2%}",
        "卡尔玛比率": f"{calmar:.2f}",
        "胜率": f"{win_rate:.2%}",
        "盈亏比": f"{profit_loss_ratio:.2f}",
    })
```

---

## 六、常见收益率错误排查清单

| 症状 | 可能原因 | 修正 |
|------|---------|------|
| 年化收益 > 1000% | 用了 cumsum 而非 cumprod | 改用 `(1+r).cumprod()` |
| 夏普比率异常高 | 波动率用了交易天数而非 sqrt(trading_days) | 检查是否漏了 sqrt |
| 最大回撤 = 0 | 数据全是正的（可能用了后复权上涨趋势） | 用前复权数据重新计算 |
| 收益曲线不连续 | 除权日附近有跳空 | 确认复权方式，检查 adj_factor |
| 年化收益与直觉不符 | 交易天数 252 vs 365 混淆 | 确认市场类型 |

---

> **动手练习**：用 AKShare 拉取一只股票 5 年日线数据，分别用简单收益率 cumsum 和 (1+r).cumprod() 画收益曲线，观察两者在 5 年尺度上的差异。然后用对数收益率 verify：log_ret.cumsum() 的指数化结果应接近 (1+r).cumprod()。
