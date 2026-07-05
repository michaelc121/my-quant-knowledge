# 统计套利（Statistical Arbitrage）

> **来源文档**：[策略分类框架与案例](../../strategies/strategy-taxonomy.md)
> **前置要求**：理解均值回复策略和相关系数的概念
> **更新日期**：2026-07-03

## 一句话理解

统计套利不是赌涨跌，而是**赌两个资产之间的关系会保持稳定**。当它们的关系暂时偏离正常水平时（一个涨了、另一个没跟上），买入便宜的同时卖出贵的，等关系恢复后平仓赚差价。

## 它和其他策略有什么不同

对比三种策略的核心逻辑，差异就很清晰了：

```text
趋势跟踪：    "涨了还会涨"        → 追涨杀跌
均值回复：    "涨多了会跌"        → 单品种反向交易
统计套利：    "A涨了B没涨会补涨"  → 做多A、做空B，对冲大盘风险
```

**统计套利不关心市场涨跌，只关心 A 和 B 之间的关系是否正常。** 大盘涨 3%，你的股票 A 涨了 2%、股票 B 涨了 4%，对你来说"B 涨多了、A 涨少了"，应该卖 B 买 A。这就是统计套利的思考方式。

---

## 一、最常见的形态：配对交易（Pairs Trading）

### 核心流程

统计套利（以配对交易为例）分四步：

```text
第一步：找配对
  在同一个行业内找两只相关性高的股票（如茅台和五粮液、招行和兴业）
  → 统计上：做协整检验，确认它们的历史价格关系稳定

第二步：定义价差
  价差 = price_A - hedge_ratio × price_B
  当价差偏离均值超过 N 倍标准差时，认为出现交易机会

第三步：开仓
  价差过大（A 比 B 贵太多）→ 卖 A、买 B（赌价差缩小）
  价差过小（A 比 B 便宜太多）→ 买 A、卖 B（赌价差扩大）

第四步：平仓
  价差回归均值 → 平仓获利
  价差继续扩大超过预设止损线 → 止损（关系可能已破裂）
```

### A 股中的实际限制

配对交易需要**做空**一只股票来对冲。A 股的做空渠道有限：

| 渠道 | 是否可用 | 限制 |
|------|---------|------|
| 融资融券 | 可做空，但有限 | 融券标的只有 ~900 只，且经常无券可融 |
| 股指期货 | 可对冲大盘风险 | 只能对冲市场 beta，不能对冲个股 alpha |
| ETF 期权 | 可对冲 ETF | 标的有限 |

**实际怎么做**：A 股中更可行的方式是"不完全对冲"——用同行业多只股票的多头组合替代做空仓位的风险暴露，或者用股指期货做方向性对冲，而不是精确的一对一配对。

---

## 二、协整 vs 相关：最容易搞混的概念

统计套利最核心的概念是**协整**（Cointegration），很多人把它和相关（Correlation）混为一谈。

```python
import numpy as np
import pandas as pd

# 相关但不协整的例子
# 两只股票都是随机游走，方向相关但价差完全不均值回复
np.random.seed(42)
drift = np.random.normal(0.001, 0.02, 500)
price_a = 100 * (1 + drift).cumprod()
price_b = 100 * (1 + drift + np.random.normal(0, 0.01, 500)).cumprod()

# 相关系数可能很高（0.9+），但价差不会回归均值
corr = np.corrcoef(price_a, price_b)[0, 1]
print(f"相关性: {corr:.3f}")  # 可能 0.9+

# 价差检验→不平稳（不会回归均值）
from statsmodels.tsa.stattools import adfuller
spread = price_a - price_b
adf_stat, adf_pvalue = adfuller(spread)[:2]
print(f"价差平稳性检验 p-value: {adf_pvalue:.3f}")  # 可能 0.5+（不平稳）
```

| | 相关（Correlation） | 协整（Cointegration） |
|---|---|---|
| 定义 | 两个变量的变动方向和幅度是否一致 | 两个变量的线性组合是否稳定 |
| 是否意味着价格回归 | 不 | 是 |
| 统计检验 | Pearson / Spearman | ADF 检验、Johansen 检验 |
| 例子 | 茅台涨 10% 时五粮液大概率也涨 | 茅台和五粮液的**价差**在 ±5% 范围内波动 |
| 配对交易能用吗 | 不能——价差可能越走越远 | 能——价差会回归均值 |

**记住**：相关不等于协整。不要看到两个股票相关系数高就去做配对交易——要先做协整检验。

---

## 三、完整的配对交易回测流程

```python
import pandas as pd
import numpy as np
import statsmodels.api as sm
from statsmodels.tsa.stattools import coint, adfuller


def find_pairs(stock_prices: pd.DataFrame, p_threshold: float = 0.05):
    """
    在全市场或行业内寻找协整对的简化版本
    返回所有协整关系显著的股票对
    """
    stocks = stock_prices.columns
    pairs = []

    for i in range(len(stocks)):
        for j in range(i + 1, len(stocks)):
            score, pvalue, _ = coint(stock_prices[stocks[i]], stock_prices[stocks[j]])
            if pvalue < p_threshold:
                pairs.append({"stock_a": stocks[i], "stock_b": stocks[j],
                              "pvalue": pvalue})

    return pd.DataFrame(pairs).sort_values("pvalue")


def calc_spread(price_a: pd.Series, price_b: pd.Series):
    """
    计算对冲后的价差
    """
    X = sm.add_constant(price_b)
    model = sm.OLS(price_a, X).fit()
    hedge_ratio = model.params.iloc[1]
    spread = price_a - hedge_ratio * price_b
    return spread, hedge_ratio


def pair_trading_backtest(
    price_a: pd.Series, price_b: pd.Series,
    entry_z: float = 2.0,       # 开仓阈值：N 倍标准差
    exit_z: float = 0.5,        # 平仓阈值：回归到 N 倍标准差以内
    stop_z: float = 3.5,        # 止损阈值
    roll_window: int = 60,      # 滚动窗口：重新计算均值和标准差
):
    """
    配对交易回测

    逻辑：
    - 用过去 roll_window 天计算价差的均值和标准差
    - 当价差 Z-score > entry_z → 做空 A、做多 B
    - 当价差 Z-score < -entry_z → 做多 A、做空 B
    - 价差回归 exit_z 以内或超出 stop_z → 平仓
    """
    spread, hedge_ratio = calc_spread(price_a, price_b)
    z_score = spread.rolling(roll_window).apply(
        lambda x: (x[-1] - x[:-1].mean()) / x[:-1].std()
    )

    position = 0  # 1=做多A做空B, -1=做空A做多B, 0=空仓
    trades = []

    for i in range(roll_window, len(z_score)):
        z = z_score.iloc[i]

        if position == 0:
            if z > entry_z:
                position = -1  # A 太贵，卖 A 买 B
                trades.append({"date": z_score.index[i], "action": "卖A买B", "z": z})
            elif z < -entry_z:
                position = 1   # A 太便宜，买 A 卖 B
                trades.append({"date": z_score.index[i], "action": "买A卖B", "z": z})

        elif position == 1:   # 当前持仓：做多A做空B
            if abs(z) < exit_z or z < -stop_z:
                position = 0
                trades.append({"date": z_score.index[i], "action": "平仓", "z": z})

        elif position == -1:  # 当前持仓：做空A做多B
            if abs(z) < exit_z or z > stop_z:
                position = 0
                trades.append({"date": z_score.index[i], "action": "平仓", "z": z})

    return pd.DataFrame(trades), z_score


def half_life(spread: pd.Series):
    """
    价差回复的半衰期——衡量价差平均需要多久回归均值
    半衰期越短，策略的交易频率越高
    """
    spread = spread.dropna()
    spread_lag = spread.shift(1)
    delta = spread - spread_lag

    X = sm.add_constant(spread_lag.iloc[1:])
    y = delta.iloc[1:]
    model = sm.OLS(y, X).fit()

    # 半衰期 = ln(2) / |lambda|，其中 lambda 是回归系数
    lam = model.params.iloc[1]
    if lam >= 0:
        return float("inf")  # 不回复
    return np.log(2) / abs(lam)
```

---

## 四、统计套利的失败模式

### 最常见的死法：协整关系破裂

```text
2008 年之前：银行股和保险股高度协整 → 配对交易赚钱
2008 年之后：关系破裂，价差一去不回头

这不是统计套利的"参数调错了"——而是支撑统计关系的底层经济逻辑变了。
任何纯统计的套利策略都有这个风险。
```

### 如何预防

| 方法 | 说明 |
|------|------|
| 周期性重新检验协整 | 每月重新跑一遍协整检验，p-value < 0.05 才继续 |
| 加入基本面过滤 | 如果价差偏离是因为基本面变化（如业绩暴雷），不交易 |
| 硬止损 | 设置价差止损（如 3.5 倍标准差），关系破裂时及时退出 |
| 多对配对 | 同时持有 5-10 对配对，单对破裂对组合影响小 |

---

## 五、统计套利 vs 其他策略的关系

```text
               赌方向                    赌方向中性
              ┌────┐                    ┌────┐
  赌延续      │ 趋势跟踪 │                │ 统计套利 │
  赌回归      │ 均值回复 │                │ 配对交易  │
              └────┘                    └────┘
           单品种交易                  多品种对冲
```

统计套利和均值回复都是"赌回归"，但统计套利通过对冲排除了市场方向风险。代价是：**需要两个品种同时交易**，交易成本翻倍，而且需要做空工具。

## 回到来源文档

[← 返回策略分类框架与案例](../../strategies/strategy-taxonomy.md)
