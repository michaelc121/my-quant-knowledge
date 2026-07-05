# 前复权

> **来源文档**：[量化必备金融知识](../../basics/market-finance.md)
> **前置要求**：理解除权除息的基本概念
> **更新日期**：2026-07-02

## 一句话理解

前复权是把历史上所有价格按最新的股本结构重新计算，让历史 K 线看起来像是"从未发生过除权除息"。策略回测用前复权，选股研究用后复权。

## 为什么需要复权

股票发生分红送股时，价格会突然跳空（比如 10 元的股票每 10 股送 10 股，次日开盘变成 5 元）。这个跳空不反映真实涨跌，但会严重干扰任何基于价格计算的技术指标。

```python
# 不复权的危害：送股后均线"假死叉"
# 股票前一日收盘 10 元，10 送 10 后次日参考价变 5 元
# 不复权：价格从 10 → 5，看起来跌了 50%，均线瞬间向下拐头
# 实际：持仓市值不变（10 元 × 100 股 = 5 元 × 200 股），策略不该有任何变化
```

## 前复权 vs 后复权

| | 前复权 | 后复权 |
|---|---|---|
| 调整方向 | 修改**过去**的价格 | 修改**未来**的价格 |
| 最新价格 | 与实际一致 ✓ | 被放大，非真实价格 |
| 历史价格 | 可能变为负数（多次送转后） | 保持合理 |
| 适用场景 | **回测**（关心最新价和收益率） | 选股研究（关心绝对价格变化幅度） |

### 前复权公式

```python
def forward_adjust(df, adjust_factor_col="adj_factor"):
    """
    前复权：用复权因子逆向调整历史价格。
    
    adj_factor 的含义：1 元不复权的历史价格 = adj_factor 元前复权价格
    - 因子 < 1：发生过送转，历史价格被下调
    - 因子 = 1：未发生除权
    
    关键操作：当天之前的所有 OHLCV × (当前因子 / 历史因子)
    """
    df = df.sort_values("date").copy()
    latest_factor = df[adjust_factor_col].iloc[-1]
    
    for col in ["open", "high", "low", "close"]:
        df[f"{col}_adj"] = df[col] * (latest_factor / df[adjust_factor_col])
    
    return df
```

### 后复权公式

```python
def backward_adjust(df, adjust_factor_col="adj_factor"):
    """后复权：用复权因子顺向调整未来价格。"""
    df = df.sort_values("date").copy()
    first_factor = df[adjust_factor_col].iloc[0]
    
    for col in ["open", "high", "low", "close"]:
        df[f"{col}_adj"] = df[col] * (df[adjust_factor_col] / first_factor)
    
    return df
```

## 代码验证：直观感受复权效果

```python
import akshare as ak
import matplotlib.pyplot as plt

# 拉取平安银行数据（2015 年前后多次送转，适合观察复权效果）
df = ak.stock_zh_a_hist(symbol="000001", period="daily", 
                         start_date="20140101", end_date="20240701",
                         adjust="")  # 不复权

# AKShare 的 adjust 参数可直接获取不同复权模式
df_qfq = ak.stock_zh_a_hist(symbol="000001", period="daily",
                             start_date="20140101", end_date="20240701",
                             adjust="qfq")  # 前复权

# 绘制对比
fig, axes = plt.subplots(2, 1, figsize=(12, 8))
axes[0].plot(df["日期"], df["收盘"], label="不复权")
axes[0].plot(df_qfq["日期"], df_qfq["收盘"], label="前复权")
axes[0].set_title("平安银行 (000001) 不复权 vs 前复权")
axes[0].legend()

# 收益率对比
ret_raw = df["收盘"].pct_change()
ret_adj = df_qfq["收盘"].pct_change()
axes[1].plot(df["日期"], ret_raw, alpha=0.5, label="不复权收益率")
axes[1].plot(df_qfq["日期"], ret_adj, alpha=0.5, label="前复权收益率")
axes[1].set_title("日收益率对比：除权日附近不复权会出极大负收益（假信号）")
axes[1].legend()

plt.tight_layout()
plt.show()
```

## 常见误区

1. **"不复权也能做回测"**：不行。除权日附近的技术指标（均线、MACD、布林带）会全部失真。
2. **"前复权和后复权收益率一样"**：只有在考虑分红再投资时才一样。前复权假设现金分红以除权价再买入股票，后复权不改变收益率（因为分子分母同比例缩放）。
3. **"前复权价格可能变负数"**：是的，分红金额超过历史买入成本时会出现。但不影响收益率计算，因为收益率 = (P_t - P_{t-1}) / P_{t-1}，分子分母同比例缩放抵消。
4. **"所有品种都需要复权"**：ETF、期货不需要复权（不存在除权除息概念）。期货需要考虑换月移仓的跳空，但那不是复权问题。

## 回到来源文档

[← 返回量化必备金融知识](../../basics/market-finance.md)
