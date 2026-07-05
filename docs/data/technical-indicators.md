# 常用技术指标速查

> **前置要求**：已阅读 bar-conventions.md，理解 K 线结构和复权逻辑。
> **内容概览**：ATR、ADX、MACD、布林带、RSI、乖离率的公式、Python 实现与量化用法。
> **更新日期**：2026-07-03

---

技术指标在量化中用于构建信号、过滤噪声和风险度量。**核心原则**：理解指标计算的数学逻辑，比记住公式更重要——这样才能判断它是否适合你的策略场景。

---

## 一、ATR（Average True Range，平均真实波幅）

### 一句话

衡量**波动幅度**，不指示方向。ATR 大 = 价格波动剧烈，ATR 小 = 价格平缓。

### 公式

```text
TR = max(当日最高 - 当日最低, |当日最高 - 前日收盘|, |当日最低 - 前日收盘|)
ATR = TR 的 N 日移动平均（常用 N=14 或 N=20）
```

### Python 实现

```python
def calc_atr(high, low, close, period=14):
    """
    计算 ATR
    high/low/close: pd.Series
    """
    tr = pd.DataFrame({
        "hl": high - low,
        "hc": (high - close.shift(1)).abs(),
        "lc": (low - close.shift(1)).abs(),
    }).max(axis=1)
    return tr.rolling(period).mean()
```

### 量化中的用途

| 用途 | 说明 |
|------|------|
| 仓位管理 | 海龟交易法用 ATR 定仓位：波动大时自动减仓 |
| 止损线 | 入场价 ± N × ATR（如 2×ATR）作为动态止损 |
| 过滤信号 | 只在高波动（市况活跃）时交易 |
| 通道突破 | 价格突破前高 + ATR 倍数时确认突破有效 |

---

## 二、ADX（Average Directional Index，平均趋向指数）

### 一句话

衡量**趋势的强度**（而不是方向）。ADX > 25 趋势强，ADX < 20 震荡市。

### 公式

```text
先计算方向指标（DI+ 和 DI-），再平滑得到 ADX：
ADX 高 → 趋势明显（不论涨跌）
ADX 低 → 横盘震荡，不适合趋势策略
```

### Python 实现

```python
def calc_adx(high, low, close, period=14):
    """
    计算 ADX 和 DI+/DI-
    ADX 只判断趋势强度，DI+ > DI- 表示多头趋势
    """
    tr = pd.DataFrame({
        "hl": high - low,
        "hc": (high - close.shift(1)).abs(),
        "lc": (low - close.shift(1)).abs(),
    }).max(axis=1)
    atr = tr.rolling(period).mean()

    up_move = high - high.shift(1)
    down_move = low.shift(1) - low

    dm_plus = np.where((up_move > down_move) & (up_move > 0), up_move, 0)
    dm_minus = np.where((down_move > up_move) & (down_move > 0), down_move, 0)

    di_plus = 100 * pd.Series(dm_plus).rolling(period).mean() / atr
    di_minus = 100 * pd.Series(dm_minus).rolling(period).mean() / atr

    dx = 100 * (di_plus - di_minus).abs() / (di_plus + di_minus)
    adx = dx.rolling(period).mean()
    return adx, di_plus, di_minus
```

### 量化中的用途

| 用途 | 说明 |
|------|------|
| 趋势过滤 | 只用 ADX > 25 时交易，排除震荡期 |
| 方向确认 | DI+ > DI- 且 ADX 上升 → 多头趋势延续 |
| 反转预警 | ADX 从高位下降 → 趋势可能结束 |

---

## 三、MACD（Moving Average Convergence Divergence）

### 一句话

用两条均线的位置关系判断趋势方向和动能变化。最通用的趋势指标。

### 公式

```text
EMA12 = 12 日指数移动平均
EMA26 = 26 日指数移动平均
DIF = EMA12 - EMA26
DEA = DIF 的 9 日 EMA
MACD 柱 = (DIF - DEA) × 2
```

### Python 实现

```python
def calc_macd(close, fast=12, slow=26, signal=9):
    """
    计算 MACD
    - DIF: 快慢线之差（正值 = 短期比长期强）
    - DEA: DIF 的平滑线
    - 柱状图: DIF - DEA
    """
    ema_fast = close.ewm(span=fast, adjust=False).mean()
    ema_slow = close.ewm(span=slow, adjust=False).mean()
    dif = ema_fast - ema_slow
    dea = dif.ewm(span=signal, adjust=False).mean()
    macd_bar = 2 * (dif - dea)
    return dif, dea, macd_bar
```

### 量化中的用途

| 用途 | 说明 |
|------|------|
| 趋势方向 | DIF > 0 多头，DIF < 0 空头 |
| 金叉/死叉 | DIF 上穿 DEA → 买入信号（滞后但稳定） |
| 背离 | 价格创新高但 MACD 柱没创新高 → 顶背离，反转可能 |
| 零轴突破 | DIF 从负转正 → 趋势转多，确认性高 |

---

## 四、布林带（Bollinger Bands）

### 一句话

在移动均线上下各加 N 倍标准差形成的通道。价格到上轨 = 短期超买，到下轨 = 短期超卖。

### 公式

```text
中轨 = N 日均线（常用 N=20）
上轨 = 中轨 + K × 标准差（常用 K=2）
下轨 = 中轨 - K × 标准差
```

### Python 实现

```python
def calc_bollinger(close, period=20, std_mult=2):
    """
    计算布林带
    """
    mid = close.rolling(period).mean()
    std = close.rolling(period).std()
    upper = mid + std_mult * std
    lower = mid - std_mult * std
    return mid, upper, lower
```

### 量化中的用途

| 用途 | 说明 |
|------|------|
| 超买超卖 | 价格到上轨→可能回调，不追高 |
| 均值回复 | 布林带宽度稳定时，价格到上/下轨后回归中轨 |
| 突破信号 | 布林带开口扩大 + 价格沿上轨走→趋势延续 |
| 波动率变化 | 带宽 = (上轨 - 下轨) / 中轨，带宽急缩后通常有大行情 |

**注意**：布林带是**均值回复指标**，但价格突破上轨后可能继续上涨（趋势情况）。需要区分震荡市和趋势市。

---

## 五、RSI（Relative Strength Index，相对强弱指标）

### 一句话

衡量近期涨跌幅的比例，判断超买还是超卖。数值范围 0-100。

### 公式

```text
RS = N 日内平均涨幅 / N 日内平均跌幅（绝对值）
RSI = 100 - 100 / (1 + RS)
常用 N=14，超买线 70，超卖线 30
```

### Python 实现

```python
def calc_rsi(close, period=14):
    """
    计算 RSI
    """
    delta = close.diff()
    gain = delta.clip(lower=0)
    loss = (-delta).clip(lower=0)
    avg_gain = gain.rolling(period).mean()
    avg_loss = loss.rolling(period).mean()
    rs = avg_gain / avg_loss
    rsi = 100 - 100 / (1 + rs)
    return rsi
```

### 量化中的用途

| 用途 | 说明 |
|------|------|
| 超买超卖 | RSI > 70 超买，RSI < 30 超卖（参数可调） |
| 背离信号 | 价格新高但 RSI 没新高 → 顶背离，反转概率高 |
| 趋势过滤 | 多头趋势中 RSI 回落到 40-50 可能是买入机会 |
| 动量确认 | RSI > 50 多头动能，RSI < 50 空头动能 |

---

## 六、乖离率（BIAS）

### 一句话

价格偏离均线的百分比。正乖离 = 价格远高于均线，负乖离 = 远低于均线。

### 公式

```text
BIAS(N) = (当前价格 - N 日均线) / N 日均线 × 100%
```

### Python 实现

```python
def calc_bias(close, period=6):
    """
    计算乖离率
    BIAS 常用周期: 6 日、12 日、24 日
    """
    ma = close.rolling(period).mean()
    bias = (close - ma) / ma * 100
    return bias
```

### 量化中的用途

| 用途 | 说明 |
|------|------|
| 均值回复信号 | BIAS 绝对值大时 → 价格向均线回归的概率大 |
| 分批建仓参考 | 负乖离达阈值时分批买入（如 BIAS24 < -8%） |
| 极值判断 | BIAS6 > 8% 短期超买，BIAS24 < -12% 深度超卖 |

### A 股常用参考阈值

```python
# A 股常用的乖离率阈值（仅供参考，需根据具体品种调整）
bias_thresholds = {
    "BIAS6": {"超买": 8, "超卖": -7},
    "BIAS12": {"超买": 10, "超卖": -9},
    "BIAS24": {"超买": 14, "超卖": -12},
}
```

> **与布林带的区别**：布林带用**标准差**衡量偏离（通道宽度随波动率自动调整），乖离率用**百分比**衡量偏离（阈值固定）。高波动行情中乖离率会频繁触发超买超卖，布林带则自动放宽通道过滤掉这些虚假信号。低波动行情中乖离率可能长时间达不到阈值，布林带则收紧通道及时捕捉到偏离。布林带的自适应是其优势，乖离率的固定阈值是它的简单优势。

---

## 七、指标对比与选型

| 指标 | 本质 | 适用于 | 不适用于 | 默认参数 |
|------|------|-------|---------|---------|
| ATR | 波动率 | 仓位管理、止损 | 方向判断 | 14/20 |
| ADX | 趋势强度 | 过滤震荡市 | 方向选择 | 14 |
| MACD | 趋势(含方向) | 趋势跟踪、背离 | 横盘震荡 | 12, 26, 9 |
| 布林带 | 均值回复（标准差，自适应波动率） | 均值回归、波动率带宽 | 强趋势中价格沿上轨持续走 | 20, 2 |
| RSI | 动量 | 超买超卖、背离 | 长期趋势 | 14, 70/30 |
| 乖离率 | 偏离度（百分比，固定阈值） | 极值交易、品种间横向对比 | 波动率突变期 | 6/12/24 |

**选择原则**：同一策略中不要堆砌太多指标。趋势策略 + 过滤指标（如 MACD + ADX）或均值回复策略 + 偏离指标（如布林带 + RSI）是合理的组合。趋势指标和均值回复指标混用会互相冲突——一个指标说"突破进场"，另一个说"超买准备回调"。

## 八、组合示例：ADX + MACD 确认趋势方向

### 问题的本质

ADX 只回答"趋势有多强"，不回答"往哪个方向"。趋势强但方向不明，等于没有信号。

### 方式一：用 ADX 自带的 DI+ 和 DI- 判断方向

ADX 公式中已经包含了 DI+ 和 DI-，它们分别衡量向上和向下的方向性运动：

```text
DI+ > DI- → 上涨趋势占主导
DI- > DI+ → 下跌趋势占主导
ADX  上升  → 当前方向在加强（不是反转）
```

### 方式二：用 MACD 判断方向（你想问的）

MACD 的 DIF 或 MACD 柱直接输出方向：

```text
DIF > 0 且 DIF 上穿 DEA → 多头
DIF < 0 且 DIF 下穿 DEA → 空头
```

### 两种方式对比

```python
import pandas as pd
import numpy as np


def trend_signal_adx_di(adx, di_plus, di_minus, adx_threshold=25):
    """
    用 ADX + DI+/DI- 得到方向信号
    返回: 1=多头, -1=空头, 0=无信号（趋势不够强）
    """
    signal = pd.Series(0, index=adx.index)
    strong_trend = adx > adx_threshold
    signal.loc[strong_trend & (di_plus > di_minus)] = 1   # 上涨趋势
    signal.loc[strong_trend & (di_minus > di_plus)] = -1  # 下跌趋势
    return signal


def trend_signal_adx_macd(adx, dif, dea, adx_threshold=25):
    """
    用 ADX + MACD 得到方向信号（你想问的组合）
    返回: 1=多头, -1=空头, 0=无信号
    """
    signal = pd.Series(0, index=adx.index)
    strong_trend = adx > adx_threshold
    macd_bullish = (dif > dea) & (dif > 0)   # MACD 确认多头
    macd_bearish = (dif < dea) & (dif < 0)   # MACD 确认空头
    signal.loc[strong_trend & macd_bullish] = 1
    signal.loc[strong_trend & macd_bearish] = -1
    return signal


def compare_signals(high, low, close):
    """
    对比两种信号在同一个品种上的表现
    """
    adx, di_plus, di_minus = calc_adx(high, low, close)
    dif, dea, _ = calc_macd(close)

    signal_di = trend_signal_adx_di(adx, di_plus, di_minus)
    signal_macd = trend_signal_adx_macd(adx, dif, dea)

    comparison = pd.DataFrame({
        "ADX": adx,
        "DI+": di_plus,
        "DI-": di_minus,
        "DIF": dif,
        "DEA": dea,
        "信号(DI法)": signal_di,
        "信号(MACD法)": signal_macd,
    })

    # 统计一致率
    agreement = (signal_di == signal_macd).mean()
    print(f"两种信号一致率: {agreement:.1%}")
    print("大部分情况下一致，差异集中在趋势转折初期")
    print("- DI+/- 比 MACD 对价格变化更敏感，转折时信号出现更早")
    print("- MACD 更平滑，信号更少但更稳定")
    return comparison


# 实际使用建议：
# - 趋势跟踪策略 → 用 ADX + DI+/DI-（信号早 1-2 根 K 线）
# - 中长线策略   → 用 ADX + MACD（信号更稳定，减少假信号）
# - 两种都行     → 选择你更熟悉、更信任的那个，不要同时用
```

### 结论

ADX + DI+/DI- **已经能同时告诉你趋势强度和方向**，不需要额外依赖 MACD。但如果你已经习惯了 MACD，用 ADX（强度过滤）+ MACD（方向确认）也是一个完全合理的组合，因为 MACD 的信号比 DI+/- 更平滑、更少假信号。

```text
你的问题"ADX 和 MACD 搭配是否能确认涨跌"→ 能。
但 ADX 直接自带的方向指标 DI+/- 就能完成这件事——不需要额外引入一个指标。
用哪个取决于你的策略周期和噪声容忍度。
```



## 常见误区

1. **"指标越多越好"**：三个兼容的指标比十个冲突的指标有用。信号叠加反而降低信噪比。
2. **"默认参数适合所有品种"**：ATR 14 在 A 股日线和美股小时线效果完全不同。参数应该根据品种和周期调整。
3. **"背离信号百分百赚钱"**：背离信号的概率优势只有 55-65%，准确率远低于直觉印象。背离只是一个信号，不是圣杯。
4. **"布林带突破上轨就该卖"**：在强趋势中，价格会沿着上轨持续运行。用量化方法先确认是否是趋势市，再用布林带。

---

> **动手练习**：用 AKShare 获取一只股票 3 年日线，在上述 6 个指标中选 2 个你觉得逻辑兼容的（如 MACD + ADX），写一个简单的双信号策略。对比用 2 个信号 vs 只用 1 个信号的夏普比率差异。重点观察：多了另一个指标后，是改善了收益还是降低了交易次数。
