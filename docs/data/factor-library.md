# 经典因子库

> **前置要求**：已阅读 factor-intro.md，理解 IC、分层回测等评估方法。
> **内容概览**：6 大类、15+ 个经典因子的公式、Python 代码与注意事项。
> **更新日期**：2026-07-02

---

## 一、动量因子

### MOM — 过去 N 日收益

最基础的因子，也是多数人的第一个因子。

```python
def mom(close: pd.DataFrame, window: int = 20) -> pd.DataFrame:
    """动量：过去 window 个交易日的累计收益"""
    return close.pct_change(window)
```

- 学术参考：Jegadeesh & Titman (1993)
- 常见窗口：5、10、20、60、120 天
- 注意：A 股短期动量弱，**反转效应更明显**，常见做法是做空短期动量（反转因子）

### 加权动量

给近期更高权重，减少远期噪声。

```python
def weighted_momentum(close: pd.DataFrame, window: int = 20) -> pd.DataFrame:
    """加权动量：近期收益率更高权重"""
    returns = close.pct_change()
    weights = pd.Series(range(1, window + 1), index=range(window))
    weights = weights / weights.sum()
    # 滚动窗口内加权求和
    wm = returns.rolling(window).apply(
        lambda x: (x * weights.values).sum()
    )
    return wm
```

---

## 二、反转因子

### REV — 短期反转

过去几日涨幅越大，未来几日越可能回调。

```python
def reversal(close: pd.DataFrame, window: int = 5) -> pd.DataFrame:
    """反转因子：取负动量"""
    return -close.pct_change(window)
```

- 学术参考：Jegadeesh (1990)
- A 股和美股均有显著的日度反转效应
- 常与动量因子同时使用，做多长期动量 + 做空短期动量

---

## 三、价值因子

### BP — 市净率倒数

账面市值比，经典的价值因子（Fama-French 三因子之一）。

```python
def book_to_price(
    market_cap: pd.DataFrame,
    book_value: pd.DataFrame
) -> pd.DataFrame:
    """BP = 每股净资产 / 股价 = 1 / PB"""
    return book_value / market_cap
```

### EP — 市盈率倒数

比 PE 更符合因子"越大越好"的习惯。

```python
def earnings_to_price(
    market_cap: pd.DataFrame,
    net_profit: pd.DataFrame
) -> pd.DataFrame:
    """EP = 净利润 / 市值 = 1 / PE"""
    return net_profit / market_cap
```

- 学术参考：Fama & French (1993)
- 使用逻辑：买入便宜（BP/EP 高）、卖出贵（BP/EP 低）
- A 股价值因子有效，但具有明显的**周期性**（经济复苏期强，衰退期弱）

### SP — 市销率倒数

```python
def sales_to_price(
    market_cap: pd.DataFrame,
    revenue: pd.DataFrame
) -> pd.DataFrame:
    """SP = 营业收入 / 市值 = 1 / PS"""
    return revenue / market_cap
```

- 学术参考：Barbee et al. (1996)
- 适用于**盈利波动大**或**尚未盈利**的公司（此时 PE 用不了）

---

## 四、质量因子

### ROE — 净资产收益率

选股最常用的基本面指标。

```python
def roe(net_profit: pd.DataFrame, equity: pd.DataFrame) -> pd.DataFrame:
    """ROE = 净利润 / 净资产（所有者权益）"""
    return net_profit / equity
```

### GROSS_MARGIN — 毛利率

反映公司护城河。

```python
def gross_margin(revenue: pd.DataFrame, cost: pd.DataFrame) -> pd.DataFrame:
    """毛利率 = (营收 - 营业成本) / 营收"""
    return (revenue - cost) / revenue
```

### ACCRUAL — 应计利润

净利润中现金支持的部分越大，质量越高。

```python
def accrual(
    net_profit: pd.DataFrame,
    operating_cf: pd.DataFrame
) -> pd.DataFrame:
    """应计利润占比 = (净利润 - 经营现金流) / 总资产"""
    return (net_profit - operating_cf) / total_assets
```

- 学术参考：Sloan (1996)
- **应计利润越低**（现金流越好），未来收益越高

### COMPOSITE — 质量综合得分

```python
def quality_score(
    roe: pd.DataFrame,
    gross_margin: pd.DataFrame,
    leverage: pd.DataFrame,
    accrual: pd.DataFrame
) -> pd.DataFrame:
    """
    质量综合得分：
    - 高 ROE + 高毛利率 + 低负债 + 低应计利润 = 高质量
    """
    score = (
        roe.rank(axis=1, pct=True)
        + gross_margin.rank(axis=1, pct=True)
        - leverage.rank(axis=1, pct=True)      # 低杠杆得分高
        - accrual.rank(axis=1, pct=True)        # 低应计得分高
    ) / 4
    return score
```

---

## 五、波动率与风险因子

### VOL — 历史波动率

低波动异象：低波动股票长期跑赢高波动股票。

```python
def historical_volatility(close: pd.DataFrame, window: int = 60) -> pd.DataFrame:
    """历史波动率 = 过去 N 日收益率的标准差"""
    return close.pct_change().rolling(window).std()
```

### BETA — 市场贝塔

```python
def calc_beta(
    stock_returns: pd.DataFrame,
    market_returns: pd.Series,
    window: int = 252
) -> pd.DataFrame:
    """
    滚动回归计算贝塔
    stock_returns: 个股日收益率, index=date, columns=stock
    market_returns: 市场日收益率, index=date
    """
    # 协方差 / 市场方差
    cov = stock_returns.rolling(window).cov(market_returns)
    var = market_returns.rolling(window).var()
    return cov / var
```

- 学术参考：Ang et al. (2006, 2009)
- A 股同样存在低波动/低 Beta 异象
- 注意：波动率因子在**市场下跌期表现极好**，上涨期滞后

### MAX — 最大日收益

学术发现：过去一个月最大单日收益越大的股票，未来表现越差（"彩票效应"）。

```python
def max_daily_return(close: pd.DataFrame, window: int = 20) -> pd.DataFrame:
    """过去 window 天内的最大单日收益率"""
    return close.pct_change().rolling(window).max()
```

- 学术参考：Bali, Cakici & Whitelaw (2011)

---

## 六、流动性因子

### AMIHUD — 非流动性因子

衡量价格对成交量的敏感程度，越敏感说明流动性越差。

```python
def amihud_illiquidity(
    close: pd.DataFrame,
    amount: pd.DataFrame,
    window: int = 20
) -> pd.DataFrame:
    """
    Amihud 非流动性 = |日收益率| / 成交额
    越高 = 流动性越差
    """
    returns = close.pct_change().abs()
    illiq = returns / amount
    # 滚动窗口取中位数（减少极端值影响）
    return illiq.rolling(window).median()
```

- 学术参考：Amihud (2002)
- A 股小盘股流动性因子值高（流动性差），需要大市值股票时要排除这些

### TURN — 换手率变化

```python
def turnover_change(turnover: pd.DataFrame, window: int = 5) -> pd.DataFrame:
    """换手率变化 = 近期平均换手率 / 长期平均换手率"""
    short_ma = turnover.rolling(window).mean()
    long_ma = turnover.rolling(60).mean()
    return short_ma / long_ma
```

- 换手率突然放大通常伴随价格异动，是情绪因子

---

## 七、因子使用指南

### 常见窗口模板

```python
def compute_all_factors(close, volume, amount):
    """一键计算常用因子"""
    return {
        # 动量 (5 维)
        "mom_5":  mom(close, 5),
        "mom_10": mom(close, 10),
        "mom_20": mom(close, 20),
        "mom_60": mom(close, 60),

        # 反转
        "rev_5":  reversal(close, 5),

        # 波动率
        "vol_20": historical_volatility(close, 20),
        "vol_60": historical_volatility(close, 60),

        # 流动性
        "illiq":  amihud_illiquidity(close, amount),

        # 价格位置
        "pos_20": (close - close.rolling(20).min()) / (close.rolling(20).max() - close.rolling(20).min()),
    }
```

### 因子处理注意事项

1. **极值处理**：因子值上下 1% 分位截断（winsorize），避免单只股票过度影响
2. **行业中性化**：多因子模型中对行业做回归取残差，剔除行业暴露
3. **市值中性化**：类似方法剔除大小盘影响
4. **频率匹配**：日频因子预测日收益，月频基本面因子预测月收益，不要混用
5. **因子衰减检查**：定期重新评估 IC，如果 IR 从 0.5 降到 0.1，该退役了

### 极值处理与中性化代码

```python
def winsorize(factor: pd.DataFrame, limits: tuple = (0.01, 0.99)) -> pd.DataFrame:
    """截面截尾：每期去掉两端极端值"""
    lower = factor.quantile(limits[0], axis=1)
    upper = factor.quantile(limits[1], axis=1)
    return factor.clip(lower=lower, upper=upper, axis=0)

def neutralize(factor: pd.DataFrame, exposure: pd.DataFrame) -> pd.DataFrame:
    """
    中性化：回归取残差
    factor: 待中性化的因子
    exposure: 暴露变量（行业哑变量、log 市值等）
    """
    import statsmodels.api as sm
    residuals = pd.DataFrame(index=factor.index, columns=factor.columns)
    for date in factor.index:
        y = factor.loc[date].dropna()
        X = exposure.loc[date, y.index].T
        X = sm.add_constant(X)
        model = sm.OLS(y, X).fit()
        residuals.loc[date, y.index] = model.resid
    return residuals
```

---

> **动手练习**：用 `factor_intro.md` 的方法，从上述因子库中任选 3 个不同类别的因子，计算它们的 IC、IR，再做分层回测对比。思考：为什么有的因子 IC 为正，有的为负？
