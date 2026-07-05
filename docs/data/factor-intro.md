# 因子定义与评估体系

> **前置要求**：已阅读 bar-conventions.md，理解 K 线数据规范。
> **内容概览**：因子的定义、分类、计算流程、评估指标、组合方法。
> **更新日期**：2026-07-02

---

## 一、什么是因子

因子是一个**能预测股票未来收益的量化特征**。

数学上，因子就是每只股票在每个时间点上的一个数值：

```
因子值 f(i, t) = 股票 i 在时间 t 的某个特征值
               = 过去 20 天的累计收益
               = 市净率的倒数
               = 过去 60 天的波动率
               = ...
```

**核心公式**：因子选股的本质是相信因子值与未来收益之间存在相关性。

```python
# 因子研究的逻辑链条：
# 因子值 (t)  →  排序分组  →  多空组合  →  未来收益 (t+1, t+n)
#                ↑ 如果高因子值的组未来收益 > 低因子值的组
#                ↑ 说明该因子有效
```

---

## 二、因子分类体系

### 按驱动因素

| 因子类别 | 逻辑 | 例子 |
|---------|------|------|
| **动量** | 强者恒强 / 反转 | 过去 N 日收益率、RSI |
| **价值** | 估值回归 | PE、PB、PS、股息率 |
| **质量** | 基本面强劲 | ROE、毛利率、资产负债率 |
| **波动率** | 低波异象 | 历史波动率、最大回撤 |
| **情绪** | 市场过度反应 | 换手率、融资余额变化 |
| **成长** | 盈利增长 | 营收增速、利润增速 |
| **流动性** | 流动性溢价 | Amihud 非流动性指标、换手率 |

> 具体因子公式与代码实现见 [因子库](../data/factor-library.md)。

### 按数据来源

| 来源 | 优势 | 劣势 |
|------|------|------|
| **价格/成交量**（量价因子） | 频率高、更新快 | 信号衰减快 |
| **财务报表**（基本面因子） | 逻辑清晰、持久 | 更新慢（季度） |
| **另类数据** | 信息优势 | 获取成本高 |

---

## 三、因子计算流程

### 单因子函数模板

```python
import pandas as pd
import numpy as np

def compute_momentum_factor(
    close: pd.DataFrame,
    window: int = 20
) -> pd.DataFrame:
    """
    动量因子：过去 N 日累计收益
    close: DataFrame, index=日期, columns=股票代码
    返回: DataFrame, 同形状, 因子值
    """
    return close.pct_change(periods=window)
```

### 多因子同时计算

```python
def compute_factor_library(close: pd.DataFrame, volume: pd.DataFrame) -> dict:
    """计算常用因子库"""
    factors = {}

    # 动量
    factors["mom_5"]   = close.pct_change(5)
    factors["mom_20"]  = close.pct_change(20)
    factors["mom_60"]  = close.pct_change(60)

    # 反转（负动量）
    factors["rev_1"]   = -close.pct_change(1)   # 隔夜反转

    # 波动率
    factors["vol_20"]  = close.pct_change().rolling(20).std()

    # 成交量变化
    factors["vol_ratio_5"] = volume.rolling(5).mean() / volume.rolling(20).mean()

    # 价格位置（当前在 N 日高低点的位置）
    high_20 = close.rolling(20).max()
    low_20  = close.rolling(20).min()
    factors["pos_20"] = (close - low_20) / (high_20 - low_20)

    return factors
```

### 数据对齐与缺失处理

因子计算前必须对齐：

```python
# 对齐到同一日历
all_dates = pd.date_range("2024-01-01", "2024-12-31", freq="D")
prices = prices.reindex(all_dates)

# 上市首日之前的因子值设为 NaN
# 停牌日的收益率设为 0（或 NaN）
returns = prices.pct_change()
returns[prices.isna()] = np.nan
```

---

## 四、因子评价指标

因子计算出来后，怎么知道它有没有用？

### 1. [IC（信息系数）](../glossary/data/ic-ir.md)

Spearman 秩相关系数：因子值与下期收益的排序相关性。

```python
def calc_ic(factor: pd.Series, forward_return: pd.Series) -> float:
    """
    计算单期 IC
    factor: 截面因子值（同一天所有股票的因子值）
    forward_return: 下期收益
    """
    from scipy.stats import spearmanr
    valid = factor.notna() & forward_return.notna()
    return spearmanr(factor[valid], forward_return[valid])[0]


def calc_ic_series(
    factor_df: pd.DataFrame,
    return_df: pd.DataFrame
) -> pd.Series:
    """逐期计算 IC，返回 IC 时间序列"""
    dates = factor_df.index
    ic_values = []
    for t in range(len(dates) - 1):
        ic = calc_ic(
            factor_df.iloc[t],
            return_df.iloc[t + 1]   # t+1 期收益
        )
        ic_values.append(ic)
    return pd.Series(ic_values, index=dates[:-1])
```

**IC 的含义**：

| IC 值 | 含义 |
|-------|------|
| > 0.05 | 因子有效，正向预测 |
| < -0.05 | 因子有效，反向预测 |
| ≈ 0 | 因子无效 |
| > 0.10 | 强因子（少见） |

### 2. IR（信息比率）

IR = IC 的均值 / IC 的标准差

```python
def calc_ir(ic_series: pd.Series) -> float:
    """信息比率 = mean(IC) / std(IC)"""
    return ic_series.mean() / ic_series.std()
```

- IR > 0.5：优秀因子
- IR > 1.0：极强因子（很少见）
- 正 IR 说明因子**持续稳定**地预测收益，不只是某一段时间有效

### 3. ICIR 累积图

```python
def plot_cumulative_ic(ic_series: pd.Series, name: str = "factor"):
    """绘制累计 IC 曲线，看因子是否持续有效"""
    cum_ic = ic_series.cumsum()
    plt.figure(figsize=(10, 4))
    plt.plot(cum_ic.index, cum_ic.values, label=name)
    plt.axhline(y=0, color="gray", linestyle="--")
    plt.title(f"Cumulative IC — {name}  (IR={calc_ir(ic_series):.2f})")
    plt.legend()
    plt.grid(alpha=0.3)
    plt.show()
```

> 如果累计 IC 曲线先上升后下跌（或者过山车），说明因子可能已经失效，或者被过拟合了。

---

## 五、分层回测

把股票按因子值排序分成 5 组或 10 组，观察每组的未来收益是否单调。

```python
def factor_quantile_backtest(
    factor: pd.DataFrame,
    forward_return: pd.DataFrame,
    n_quantiles: int = 5
) -> pd.DataFrame:
    """
    分层回测：每天按因子值分成 n 组，计算每组平均下期收益
    返回：每组的累计收益序列
    """
    group_returns = []
    for t in range(len(factor) - 1):
        # 当天因子值
        f_t = factor.iloc[t]
        # 下期收益
        r_t1 = forward_return.iloc[t + 1]

        # 分成 n 组
        labels = pd.qcut(f_t.rank(method="first"), n_quantiles, labels=False)
        labels = labels + 1   # group 1..n

        # 计算每组平均收益
        grp = r_t1.groupby(labels).mean()
        group_returns.append(grp)

    result = pd.DataFrame(group_returns, index=factor.index[:-1])
    result.columns = [f"Q{i+1}" for i in range(n_quantiles)]

    # 多空组合（最高组 - 最低组）
    result["long_short"] = result[f"Q{n_quantiles}"] - result["Q1"]

    return result
```

**有效的因子**：Q5（高因子组）收益 > Q4 > ... > Q1（低因子组），单调递减或递增。

---

## 六、因子组合

单因子风险高，通常把多个因子合成一个综合分数。

### 等权合成

```python
def equal_weight_combine(factor_dict: dict[str, pd.DataFrame]) -> pd.DataFrame:
    """等权合成：每个因子先 rank 标准化，再取平均"""
    ranks = {}
    for name, f in factor_dict.items():
        # 截面 rank 归一化到 [0, 1]
        ranks[name] = f.rank(axis=1, pct=True)
    # 等权平均
    combined = pd.concat(ranks, axis=0).groupby(level=1).mean()
    return combined
```

### IC 加权

```python
def ic_weight_combine(
    factor_dict: dict[str, pd.DataFrame],
    return_df: pd.DataFrame
) -> pd.DataFrame:
    """按滚动 IC 加权合成"""
    # 计算每个因子的滚动 IC（过去 60 天）
    ic_hist = {}
    for name, f in factor_dict.items():
        ic_series = calc_ic_series(f, return_df)
        ic_hist[name] = ic_series.rolling(60).mean()

    # 权重 = max(IC, 0)，归一化
    weights = pd.DataFrame(ic_hist).clip(lower=0)
    weights = weights.div(weights.sum(axis=1), axis=0)

    # 加权合成
    ranks = {name: f.rank(axis=1, pct=True) for name, f in factor_dict.items()}
    combined = sum(weights[name].values[:, np.newaxis] * ranks[name] for name in factor_dict)
    return combined
```

---

## 七、常见因子陷阱

1. **幸存者偏差** — 只用了还活着的股票做因子测试。因子值依赖未来信息。修正：回测时必须使用当时的成份股列表
2. **前视偏差** — 用当天的收盘价算因子，但收盘价是交易结束后才知道的。修正：因子值计算时 price 要用 `close.shift(1)` 对齐到已知信息
3. **过拟合** — 从 1000 个因子中挑出回测最好的 5 个。修正：样本外测试、交叉验证
4. **因子衰减** — 一个因子被发现后很快失效。修正：持续监控 IC 滚动均值，衰减了就换
5. **拥挤** — 太多资金跟踪同一个因子，因子收益被抹平。修正：关注因子拥挤度指标

---

## 八、完整工作流示例

```python
import pandas as pd
import numpy as np
from scipy.stats import spearmanr

# 1. 数据准备
close = pd.read_parquet("data/close.parquet")   # index=date, columns=stock
volume = pd.read_parquet("data/volume.parquet")

# 2. 计算因子
factor = close.pct_change(20)   # 动量因子

# 3. 计算下期收益（t+1 的收益）
forward_ret = close.pct_change().shift(-1)

# 4. 对齐
valid = factor.notna() & forward_ret.notna()
factor = factor[valid]
forward_ret = forward_ret[valid]

# 5. 评估
ic_series = calc_ic_series(factor, forward_ret)
ir = calc_ir(ic_series)
print(f"IC 均值: {ic_series.mean():.4f}")
print(f"IR:      {ir:.4f}")

# 6. 分层回测
result = factor_quantile_backtest(factor, forward_ret, n_quantiles=5)

# 7. 累积收益
(result + 1).cumprod().plot()
plt.title(f"分层回测 — IR={ir:.2f}")
plt.show()
```

---

> **进阶阅读**：[因子挖掘方法论](factor-discovery.md) —— 学会自己造因子，而不只是用现成的因子库。

> **动手练习**：用 AKShare 拉取 50 只股票一年日线，计算 20 日动量因子，按上面的流程跑一遍 IC 计算和分层回测。观察：动量因子的 IC 是正还是负？IR 有多少？
