# 海龟交易法完整拆解

> **前置要求**：已阅读 backtest-101.md，理解回测流程和评估指标。
> **内容概览**：海龟交易法的规则系统、Python 实现、回测分析、优缺点。
> **更新日期**：2026-07-02

---

## 一、背景

海龟交易法是理查德·丹尼斯（Richard Dennis）在 1980 年代训练徒弟时使用的一套完整的趋势跟踪系统。他招募了一群没有交易经验的普通人（"海龟"），用这套方法教他们在 4 年间赚了 1.75 亿美元。

**为什么海龟值得学**：它是一套**完整的交易系统**，不是孤立的入场信号。它包含了入场、止损、加仓、退出、仓位管理全套规则。

---

## 二、核心规则

### 1. 入场规则：唐奇安通道突破

```python
def entry_signal(close: pd.Series, high: pd.Series, low: pd.Series, period: int = 20) -> pd.Series:
    """
    唐奇安通道突破入场
    - 系统 1（短线）：突破 20 日最高点做多，突破 20 日最低点做空
    - 系统 2（长线）：突破 55 日最高点做多，突破 55 日最低点做空
    
    返回：1=做多, -1=做空, 0=无信号
    """
    high_20 = high.rolling(20).max().shift(1)  # shift(1): 用前一天的突破位
    low_20  = low.rolling(20).min().shift(1)

    signal = pd.Series(0, index=close.index)
    signal[close > high_20] = 1    # 突破高点做多
    signal[close < low_20]  = -1   # 跌破低点做空
    return signal
```

**关键细节**：
- 用历史最高/最低点作为突破位，不是均值
- **失效过滤**：如果入场后跌破过去 10 日低点，该次入场算失败，不再用该系统入场一次
- 系统 1 和系统 2 独立运行，两个信号可以同时存在

### 2. 止损规则：ATR 跟踪止损

```python
def stop_loss_price(entry_price: float, atr: float, multiplier: float = 2.0) -> float:
    """2倍 ATR 止损：入场价 - 2 × ATR"""
    return entry_price * (1 - multiplier * atr / entry_price)
```

海龟不使用固定比例止损，而是用 ATR（平均真实波幅）动态止损。波动越大，止损越宽——避免被正常波动震出去。

### 3. 加仓规则：金字塔加仓

海龟实行**分批加仓**，不是全仓一次性买入。

```python
def pyramid_positions(entry_price: float, atr: float, unit_size: int, step: float = 0.5):
    """
    海龟加仓规则：
    - 每上涨 0.5 ATR 加仓一次
    - 最大加至 4 单位
    """
    positions = []
    for i in range(4):
        price = entry_price * (1 + i * step * atr / entry_price)
        positions.append({"unit": i + 1, "price": price, "size": unit_size})
    return positions
```

### 4. 退出规则：反向突破退出

海龟的退出不是固定止盈，而是等待反向信号。

```python
def exit_signal(close: pd.Series, low: pd.Series, period: int = 10) -> pd.Series:
    """
    海龟退出规则（系统 1）：
    多头：跌破过去 10 日最低点 → 平多
    空头：涨破过去 10 日最高点 → 平空
    """
    low_10  = low.rolling(10).min().shift(1)
    high_10 = low.rolling(10).max().shift(1)

    exit_long  = close < low_10
    exit_short = close > high_10
    return exit_long.astype(int) - exit_short.astype(int)
```

---

## 三、资金管理：用 ATR 确定仓位大小

> 海龟用 ATR 定仓位，凯利公式是另一套思路——详见 [资金管理与风控框架](../trading/risk-management.md)。两者可以互补使用。

海龟最有价值的部分可能不是入场规则，而是**基于波动率的仓位管理**。

```python
def unit_size(
    account_value: float, atr: float, risk_ratio: float = 0.01
) -> int:
    """
    计算每单位的交易数量
    1 单位 = 账户的 1% ÷ ATR
    """
    risk_per_unit = atr  # 每手波动一个 ATR 的金额
    total_risk = account_value * risk_ratio  # 单笔最多亏 1%
    return max(1, int(total_risk / risk_per_unit))
```

**含义**：ATR 越小（波动越小）→ 单位越大，反之亦然。市场波动低时多买，波动高时少买。

**完整仓位**：最多 4 个单位，分布在不同的加仓价位上。总风险 = 4 × 1% = 每次最多承受 4% 的账户风险。

---

## 四、完整策略代码

```python
import pandas as pd
import numpy as np

class TurtleStrategy:
    """海龟交易法实现（简化版）"""

    def __init__(
        self,
        entry_period: int = 20,
        exit_period: int = 10,
        atr_period: int = 20,
        atr_multiplier: float = 2.0,
        risk_ratio: float = 0.01,
        cost_rate: float = 0.001
    ):
        self.entry_period = entry_period
        self.exit_period = exit_period
        self.atr_period = atr_period
        self.atr_multiplier = atr_multiplier
        self.risk_ratio = risk_ratio
        self.cost_rate = cost_rate

    def run(self, df: pd.DataFrame, account_value: float = 1_000_000):
        df = df.copy()

        # 1. ATR
        tr = pd.concat([
            df["high"] - df["low"],
            (df["high"] - df["close"].shift()).abs(),
            (df["low"] - df["close"].shift()).abs()
        ], axis=1).max(axis=1)
        df["atr"] = tr.rolling(self.atr_period).mean()

        # 2. 唐奇安入场信号
        high_break = df["high"].rolling(self.entry_period).max().shift(1)
        low_break  = df["low"].rolling(self.entry_period).min().shift(1)
        df["entry_signal"] = 0
        df.loc[df["close"] > high_break, "entry_signal"] = 1
        df.loc[df["close"] < low_break, "entry_signal"] = -1

        # 3. 退出信号
        high_exit = df["high"].rolling(self.exit_period).max().shift(1)
        low_exit  = df["low"].rolling(self.exit_period).min().shift(1)
        df["exit_signal"] = 0
        df.loc[df["close"] < low_exit, "exit_signal"] = 1   # 平多
        df.loc[df["close"] > high_exit, "exit_signal"] = -1  # 平空

        # 4. 仓位管理
        unit = account_value * self.risk_ratio / df["atr"]
        df["position"] = 0
        position = 0

        for i in range(1, len(df)):
            # 退出
            if df["exit_signal"].iloc[i] != 0 and position != 0:
                position = 0

            # 入场
            if position == 0 and df["entry_signal"].iloc[i] != 0:
                position = df["entry_signal"].iloc[i]

            df.loc[df.index[i], "position"] = position

        # 5. 回测
        df["daily_ret"] = df["close"].pct_change()
        df["strategy_ret"] = df["position"].shift(1) * df["daily_ret"]
        df["trade"] = (df["position"] != df["position"].shift(1)).astype(int)
        df["cost"] = df["trade"] * self.cost_rate
        df["strategy_ret_net"] = df["strategy_ret"] - df["cost"]
        df["cum_strat"] = (1 + df["strategy_ret_net"]).cumprod()

        return df
```

---

## 五、海龟在 A 股的适配

海龟是为期货设计的（可以做空、有杠杆、趋势性强），在 A 股需要调整：

| 原版规则 | A 股适配 | 原因 |
|---------|---------|------|
| 双向交易（多+空） | 只做多 | A 股融券成本高、券源少 |
| 4 单位加仓 | 2–3 单位 | A 股波动率低于商品期货 |
| 1% 风险比例 | 0.5% | A 股连续跌停下可能无法止损 |
| 唐奇安 20/55 日 | 缩短为 10/30 日 | A 股趋势持续时间短 |

**适配后的建议组合**：

```python
# 海龟 A 股版参数
turtle_a = TurtleStrategy(
    entry_period=10,         # 10 日突破（原版 20）
    exit_period=5,           # 5 日退出（原版 10）
    atr_multiplier=1.5,      # 1.5 倍 ATR 止损（原版 2）
    risk_ratio=0.005,        # 0.5% 风险（原版 1%）
)
```

---

## 六、海龟交易法的优缺点

### 优势

1. **系统完整性** — 不是仅一个信号，而是入场 + 止损 + 加仓 + 仓位管理全套
2. **波动率自适应** — ATR 动态调整仓位，市场波动大时自动减仓
3. **长期有效** — 趋势跟踪是最古老的策略类型之一，逻辑不会失效（只要市场有趋势）
4. **可复制** — 规则明确，没有主观判断空间

### 缺陷

1. **震荡期表现极差** — 可能在震荡市中连续止损 10+ 次，需要极强的心理承受力
2. **需要大资金** — 分散到多个品种才能平滑曲线，小资金很难复制
3. **高回撤** — 最大回撤通常在 30–50%，不是每个人都能扛住
4. **A 股适配性有限** — 没有双向交易 + T+1 限制，收益打折扣

### 海龟适合谁

```
你是否有足够的本金分散到至少 5 个品种？  → 否 → 不适合海龟
你是否能接受 30% 以上的最大回撤？         → 否 → 不适合海龟
你是否相信"趋势终将到来"？               → 否 → 不适合海龟
你的持仓周期是否以月/年为单位？          → 否 → 不适合海龟
```

如果以上全 ✓，海龟可能是最适合你入门的系统。

---

> **动手练习**：用回测 101 的 `vectorized_backtest` 和本篇的 `TurtleStrategy` 类，在某个期货品种（如螺纹钢指数）上运行海龟回测。比较 A 股版和原版的收益差异。观察：最大回撤出现在什么时候？连续止损最长的次数是多少？
