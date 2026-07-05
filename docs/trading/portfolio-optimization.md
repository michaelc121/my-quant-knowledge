# 组合优化

> **前置要求**：已阅读 risk-management.md、multi-factor-model.md，理解仓位管理和多因子选股。
> **内容概览**：均值方差模型、风险平价、约束条件、Python 实现。
> **更新日期**：2026-07-02

---

## 一、组合优化的目的

选出了 50 只股票后，每只买多少？等权最简单但不一定最好。组合优化的目标：

```text
在给定的预期收益目标下，使组合风险最小
或者在给定的风险预算下，使预期收益最大
```

---

## 二、均值方差模型（Markowitz 经典）

### 原理

```text
最小化： w^T Σ w           （组合方差）
约束条件： w^T μ ≥ target  （收益目标）
         Σ w = 1           （满仓）
         w ≥ 0             （不允许做空）
```

### 实现

```python
import numpy as np
import pandas as pd
from scipy.optimize import minimize

def mean_variance_optimize(
    returns: pd.DataFrame,
    target_return: float = None
) -> np.ndarray:
    """
    均值方差优化
    returns: index=日期, columns=股票, values=收益率
    target_return: 目标日收益率，None = 最小方差组合
    """
    n_assets = len(returns.columns)
    mean_ret = returns.mean().values
    cov_matrix = returns.cov().values

    # 目标函数：组合方差
    def portfolio_variance(weights):
        return weights @ cov_matrix @ weights

    # 约束条件
    constraints = [{"type": "eq", "fun": lambda w: np.sum(w) - 1}]  # 满仓
    if target_return is not None:
        constraints.append({
            "type": "eq",
            "fun": lambda w: w @ mean_ret - target_return
        })

    # 边界：每个资产权重 ∈ [0, 0.2]
    bounds = tuple((0, 0.2) for _ in range(n_assets))

    # 初始猜测：等权
    init_guess = np.ones(n_assets) / n_assets

    result = minimize(
        portfolio_variance,
        init_guess,
        method="SLSQP",
        bounds=bounds,
        constraints=constraints
    )

    return result.x
```

### 使用示例

```python
# 输入：50 只股票过去 12 个月的日收益率
weights = mean_variance_optimize(returns, target_return=0.001)

# 打印权重
weight_df = pd.DataFrame({
    "stock": returns.columns,
    "weight": weights
}).sort_values("weight", ascending=False)

# 过滤掉权重 < 0.1% 的
weight_df = weight_df[weight_df["weight"] > 0.001]
print(weight_df)
```

---

## 三、有效前沿

通过调整目标收益率，可以画出风险-收益的最优曲线（有效前沿）：

```python
def efficient_frontier(returns: pd.DataFrame, n_points: int = 20) -> pd.DataFrame:
    """计算有效前沿"""
    min_ret = returns.mean().min()
    max_ret = returns.mean().max()
    target_returns = np.linspace(min_ret, max_ret, n_points)

    frontier = []
    for target in target_returns:
        w = mean_variance_optimize(returns, target_return=target)
        port_ret = w @ returns.mean().values
        port_vol = np.sqrt(w @ returns.cov().values @ w)
        frontier.append({
            "收益": port_ret,
            "波动率": port_vol,
            "夏普": port_ret / port_vol if port_vol > 0 else 0
        })

    return pd.DataFrame(frontier)
```

---

## 四、风险平价

均值方差对预期收益估计极其敏感（收益率预测不准，权重就偏了）。风险平价是更好的替代。

```python
def risk_parity_optimize(returns: pd.DataFrame) -> np.ndarray:
    """
    风险平价：每项资产对组合风险的贡献相等
    """
    n_assets = len(returns.columns)
    cov = returns.cov().values

    # 边际风险贡献: MRC_i = (Σ w) / σ_p  * (Σw)_i
    def risk_contrib(w):
        port_vol = np.sqrt(w @ cov @ w)
        marginal_contrib = cov @ w / port_vol
        total_contrib = w * marginal_contrib
        return total_contrib / total_contrib.sum()  # 归一化

    # 目标：每项资产贡献 ≈ 1/n
    def objective(w):
        rc = risk_contrib(w)
        target = np.ones(n_assets) / n_assets
        return np.sum((rc - target) ** 2)

    constraints = [{"type": "eq", "fun": lambda w: np.sum(w) - 1}]
    bounds = tuple((0.01, 0.25) for _ in range(n_assets))
    init = np.ones(n_assets) / n_assets

    result = minimize(objective, init, method="SLSQP", bounds=bounds, constraints=constraints)
    return result.x
```

---

## 五、实用约束

```python
# 行业集中度限制
def industry_constraint(weights, industry_labels, max_industry_weight=0.30):
    """检查是否有行业超过 30%"""
    exposure = pd.Series(weights).groupby(industry_labels).sum()
    return (exposure <= max_industry_weight).all()

# 最小权重过滤
weights[weights < 0.005] = 0      # 权重 < 0.5% 清零
weights = weights / weights.sum()  # 重新归一化
```

---

## 六、优化 vs 简单方法的对比

| 方法 | 收益 | 风险控制 | 稳定性 | 适用 |
|------|------|---------|--------|------|
| 等权 | 中等 | 无 | 极高 | 新手默认选择 |
| 均值方差 | 理论最优 | 有 | 低（对输入敏感） | 需要精确的预期收益 |
| 风险平价 | 中等 | 有 | 高 | 多资产组合首选 |
| 最大多元化 | 中等 | 有 | 高 | 因子组合 |

> **建议**：量化新手从**等权**或**风险平价**开始。均值方差看起来很美，但预期收益的小误差会导致权重剧烈震荡。

---

> **动手练习**：取 multi-factor-model.md 选出的 top 20 股票，用等权、均值方差、风险平价三种方法分配权重，持有 3 个月，比较三组的收益曲线和最大回撤。观察：哪种方法的回撤最小？
