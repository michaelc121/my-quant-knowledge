# 策略分类框架

> **前置要求**：已掌握因子评估方法，理解 IC/IR 含义。
> **内容概览**：四大策略范式的原理、代码骨架、适用场景与风险点。
> **更新日期**：2026-07-02

---

## 一、四大范式全景

量化策略绝大多数可以归入这四类：

| 类型 | 核心假设 | 赚钱逻辑 | 典型市场 | 风险 |
|------|---------|---------|---------|------|
| **趋势跟踪** | 趋势会延续 | 吃主升/主跌段 | 期货、强趋势股 | 震荡期反复止损 |
| **均值回复** | 价格会回归均值 | 低买高卖 | 股票、商品 | 趋势来了一次亏完 |
| **统计套利** | 统计关系稳定 | 对冲后吃价差 | 期货跨期、ETF | 关系崩盘（如 2008） |
| **事件驱动** | 事件影响可预测 | 提前埋伏事件 | 股票 | 事件低于预期 |

没有万能策略，关键是**在合适的市场用对范式**。

---

## 二、趋势跟踪

### 原理

趋势跟踪不做预测，只做确认——当价格表现出方向性时，假设这个方向会持续一段时间。

### 核心组件

```python
# 趋势跟踪策略的通用骨架
def compute_trend_signal(close: pd.Series) -> pd.Series:
    """任一种趋势评判指标"""
    ma_short = close.rolling(20).mean()
    ma_long  = close.rolling(60).mean()
    signal   = (ma_short > ma_long).astype(int)   # 1 = 多头, 0 = 空头
    return signal
```

### 常见趋势判断方法

| 方法 | 原理 | 典型参数 |
|------|------|---------|
| **双均线交叉** | 短期均线上穿长期 = 买入 | MA5/MA20、MA20/MA60 |
| **ATR 通道** | 突破 N 倍 ATR 区间 | 2×ATR 通道 |
| **唐奇安通道** | 突破 N 日最高/最低 | 20 日高低点 |
| **MACD** | DIF 上穿 DEA = 买入 | 12、26、9 |
| **ADX** | ADX > 25 表示强趋势 | 14 日 ADX |

### ATR 通道示例

```python
def atr_channel(close: pd.Series, high: pd.Series, low: pd.Series, period: int = 20, multiplier: float = 2.0):
    """ATR 通道突破"""
    # ATR = 过去 N 日真实波幅的均值
    tr = pd.concat([
        high - low,
        (high - close.shift()).abs(),
        (low - close.shift()).abs()
    ], axis=1).max(axis=1)
    atr = tr.rolling(period).mean()

    center = close.rolling(period).mean()
    upper = center + multiplier * atr
    lower = center - multiplier * atr

    signal = pd.Series(0, index=close.index)
    signal[close > upper] = 1
    signal[close < lower] = -1
    return signal, atr
```

### 适用与禁忌

- **适合**：期货市场（天然有趋势）、美股长牛
- **不适合**：A 股震荡市（浪费手续费）、横盘整理期
- **关键指标**：胜率低（~40%）但盈亏比高（>2），总收益来自少数几次大行情

---

## 三、均值回复

### 原理

价格偏离均值越远，回归的概率越大。本质是在逆向交易。

### 常见方法

| 方法 | 原理 | 典型参数 |
|------|------|---------|
| **布林带** | 突破上轨做空、下轨做多 | 20 日 ± 2 倍标准差 |
| **RSI** | RSI > 70 超买做空、< 30 超卖做多 | 14 日 RSI |
| **配对交易** | 两只高度相关股票价差偏离均值 | 协整系数 |
| **乖离率** | 价格偏离均线的百分比 | 5 日乖离率 |

### 布林带示例

```python
def bollinger_band(close: pd.Series, period: int = 20, n_std: float = 2.0):
    """布林带均值回复"""
    ma = close.rolling(period).mean()
    std = close.rolling(period).std()

    upper = ma + n_std * std
    lower = ma - n_std * std

    signal = pd.Series(0, index=close.index)
    signal[close < lower] = 1    # 下轨下方 = 买入
    signal[close > upper] = -1   # 上轨上方 = 卖出
    return signal, upper, lower
```

### 适用与禁忌

- **适合**：震荡市、高流动性品种、ETF
- **不适合**：单边强趋势（会一直突破不回头的亏损）
- **关键指标**：胜率高（~60%）但盈亏比低（<1.5）

---

## 四、统计套利

> 下钻阅读：[统计套利详解](../glossary/strategies/statistical-arbitrage.md) —— 协整vs相关、完整配对交易代码、失败模式。

### 原理

利用两个或多个资产之间的统计关系（如协整），当价差偏离时做多一个、做空另一个，对冲方向性风险，赚取价差回归的收益。

### 协整配对交易（经典）

```python
def cointegration_pair_test(price_a: pd.Series, price_b: pd.Series) -> dict:
    """
    检验两只股票的价格序列是否协整
    返回协整检验结果
    """
    from statsmodels.tsa.stattools import coint, adfuller

    score, pvalue, _ = coint(price_a, price_b)

    # 残差 = 实际价差 - 拟合价差
    import statsmodels.api as sm
    X = sm.add_constant(price_b)
    model = sm.OLS(price_a, X).fit()
    residual = model.resid

    return {
        "coint_pvalue": pvalue,
        "hedge_ratio": model.params.iloc[1],
        "residual_mean": residual.mean(),
        "residual_std": residual.std(),
        "is_cointegrated": pvalue < 0.05
    }


def pair_trade_signal(
    price_a: pd.Series, price_b: pd.Series,
    hedge_ratio: float, window: int = 60, z_entry: float = 2.0
):
    """
    配对交易信号：
    当价差 > 2σ → 做空价差（卖 A 买 B）
    当价差 < -2σ → 做多价差（买 A 卖 B）
    """
    spread = price_a - hedge_ratio * price_b
    ma = spread.rolling(window).mean()
    std = spread.rolling(window).std()
    z_score = (spread - ma) / std

    signal = pd.Series(0, index=price_a.index)
    signal[z_score > z_entry]  = -1    # 做空价差
    signal[z_score < -z_entry] = 1     # 做多价差
    return signal, z_score
```

### 适用与禁忌

- **适合**：同行业股票、期货跨期套利、ETF 与净值之间
- **不适合**：低相关性品种（协整检验不通过）
- **风险**：统计关系可能崩盘（如 2008 年金融危机中大量配对关系失效）

---

## 五、事件驱动

### 原理

押注特定事件（财报、分红、IPO、政策）对价格的影响。

### 常见事件

| 事件 | 常见策略 | 特点 |
|------|---------|------|
| **财报公告** | 预期差套利 | 提前埋伏绩优股，公告后卖出 |
| **除权除息** | 填权行情 | A 股有填权效应，但幅度逐年递减 |
| **股指调仓** | 成份股调整套利 | 调入的涨、调出的跌 |
| ** ETF 申赎** | 折溢价套利 | 窗口极短，需程序化 |

### 财报公告策略骨架

```python
def earnings_surprise_strategy(
    close: pd.DataFrame,
    earnings_date: pd.Series,  # index=股票, value=财报日期
    actual_eps: pd.Series,     # 实际 EPS
    expected_eps: pd.Series,   # 预期 EPS
    pre_window: int = 5,       # 埋伏天数
    hold_window: int = 10      # 持有天数
) -> pd.Series:
    """
    财报预期差策略：
    超出预期 → 提前买入，公告后持有 N 天卖出
    低于预期 → 提前卖出/做空
    """
    surprise = (actual_eps - expected_eps) / expected_eps.abs()
    # 逻辑：超预期越多 → 预期收益越大
    return surprise
```

### 适用与禁忌

- **适合**：有明确事件日历、流动性好的品种
- **不适合**：事件结果高度不确定、容量极小的品种
- **注意**：事件驱动策略容量有限，资金大了信号会被抹平

---

## 六、策略对比与选择

### 如何选第一个策略

```
你是什么类型的市场环境？
├── 强趋势（单边上涨/下跌） → 趋势跟踪
├── 区间震荡              → 均值回复
└── 不确定                → 先做统计套利（对冲市场风险）

你偏向什么风险风格？
├── 高胜率、小亏小赚      → 均值回复
├── 低胜率、赚大亏小      → 趋势跟踪
└── 市场中性              → 统计套利
```

### 案例：T+1 短线交易——两种入场规则的冲突与拆分

这是一个 A 股短线交易非常典型的场景：T 日买入、T+1 日卖出。入场规则听起来"灵活"——既有做多趋势（追涨），也有抄底反弹（低吸）。但这里隐藏着一个核心矛盾。

#### 你的入场逻辑拆解

```text
规则一：正在上涨趋势中，明天可能还会继续涨
  → 策略范式：趋势跟踪
  → 典型信号：MA5 > MA10、MACD 金叉、价格创新高
  → 核心假设：趋势会延续

规则二：触底了，明天可能会反弹
  → 策略范式：均值回复
  → 典型信号：RSI < 30、碰布林下轨、乖离率负值
  → 核心假设：价格会回归均值
```

**这两个规则不能放在同一个策略里。** 趋势跟踪和均值回复对同一段行情的判断是相反的：一个说"涨了快买"，另一个说"涨了快卖"。放在一起会导致信号频繁冲突，哪个都赚不到钱。

#### 正确的做法：拆成两个独立的子策略

不要试图用一个策略同时兼容两种逻辑。应该拆成两个独立的策略，分开回测、分开评估、分开决定仓位。

```python
# 策略 A：追涨（趋势跟踪）
def signal_momentum(close):
    """趋势跟踪入场：价格创新高 + 短期均线上行"""
    ma5 = close.rolling(5).mean()
    ma10 = close.rolling(10).mean()
    return (close > close.shift(20).max()) & (ma5 > ma10)


# 策略 B：低吸（均值回复）
def signal_reversal(close, low):
    """均值回复入场：RSI 超卖 + 价格远离均线"""
    rsi = calc_rsi(close, 14)
    bias = (close - close.rolling(20).mean()) / close.rolling(20).mean() * 100
    return (rsi < 30) & (bias < -7)
```

**两个策略各自有各自的入场信号和出场规则，互不干扰。** 你如果同时持有了策略 A 和策略 B 的仓位，那不是"一个策略有冲突的信号"，而是"两个独立策略各自开了自己的仓位"。

#### 出场规则是这两者的共同约束

不管你是追涨还是低吸入场的，出场规则都一样：

```python
def exit_rule(close, entry_price, current_time, stop_loss_pct=0.03):
    """出场规则：止损或尾盘强制平仓"""
    # 规则 1：跌破止损线 → 立即卖出
    if close / entry_price - 1 < -stop_loss_pct:
        return True, "止损"

    # 规则 2：14:30 到了 → 强制平仓
    if current_time >= pd.Timestamp("14:30").time():
        return True, "尾盘平仓"

    return False, "持有"
```

这个出场规则其实非常好——T+1 强制尾盘平仓让回测中的持仓周期完全确定，不需要复杂的择时出场逻辑，也避免了隔夜持仓需要判断"明天是否还涨"的困境。

#### 回测方案

```
股票池：选 50-100 只流动性好的 A 股（日均成交额 > 1 亿）

回测框架：向量化回测（每日更新信号）

信号生成：
  策略 A（追涨）和策略 B（低吸）独立计算每日信号
  两个策略的持仓不共享仓位——各有各的仓位上限

出场逻辑：
  - 盘中跌破止损 → 下一根 K 线发卖出信号
  - 到达 14:30 → 发卖出信号（用当日收盘价近似）

回测输出：
  - 两个策略各自的收益曲线、夏普、胜率、交易次数
  - 对比分析：哪个策略贡献了大部分收益？哪个策略交易更稳定？
```

#### 总结

你的入场规则确实来自两个不同的策略范式。**不要尝试融合它们**——分开做、分开测、分开管。T+1 和 14:30 强制出场其实是一个很好的约束，让你的回测逻辑变得清晰。先去测测这两个子策略各自的表现，大概率会发现其中一个的夏普比率显著高于另一个——之后专注做好那个就够了。


### 新手建议

- **第一策略推荐双均线交叉**：最简单、最容易理解、回测框架最短
- **不要同时跑多个策略**：先跑通一个，理解回测的每个环节后再扩展
- **用期货测试趋势跟踪**：期货可以多空，趋势跟踪效果最好
- **用 ETF 测试均值回复**：ETF 没有停牌黑天鹅，均值回复更明显

---

## 七、策略生命周期

一个策略从发现到退役的典型路径：

```text
发现（论文/灵感）
  │
  ▼
初步验证（Notebook 快速回测）
  │
  ▼
精细化（参数扫描、样本外测试、成本估算）
  │
  ▼
模拟盘（接实时数据、观察表现）
  │
  ▼
小资金实盘（验证执行可行性）
  │
  ▼
加仓（稳定盈利后扩大资金）
  │
  ▼
监控与衰减检查（IC/IR 持续下跌 → 退役）
```

> 大多数策略在 **初步验证** 到 **精细化** 之间就被淘汰了——这是好事，说明你在实盘前就发现了问题。

---

> **动手练习**：用 bar-conventions.md 的数据，实现简单的双均线交叉策略（MA5/MA20），在 Notebook 中画出入场/出场标记。下一节 backtest-101.md 会教你如何正确评估这个策略的收益和风险。
