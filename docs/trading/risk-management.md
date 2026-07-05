# 资金管理与风险控制框架

> **前置要求**：已阅读 backtest-101.md，理解夏普、最大回撤等指标。
> **内容概览**：凯利公式、固定比例仓位、多级风控、黑天鹅应对、压力测试。
> **更新日期**：2026-07-02

---

## 一、资金管理的优先序

一个被无数实盘验证过的排序：

```text
风险控制 > 资金管理 > 策略逻辑 > 选品择时
```

意思是：仓位下重了，再好的策略也会爆仓；仓位合理了，平庸的策略也能赚钱。

---

## 二、[凯利公式](../glossary/trading/kelly-criterion.md)

凯利公式告诉你"最优下注比例"是多少，使得长期复利最大化。

### 单次下注

```python
def kelly_criterion(win_rate: float, profit_loss_ratio: float) -> float:
    """
    凯利公式：f = (bp - q) / b
    b = 盈亏比（平均赚 / 平均亏）
    p = 胜率
    q = 1 - p
    """
    b = profit_loss_ratio
    p = win_rate
    q = 1 - p
    f = (b * p - q) / b
    return max(0, min(f, 1))  # 截断到 [0, 1]
```

**举例**：

| 胜率 | 盈亏比 | 凯利比例 | 含义 |
|------|--------|---------|------|
| 40% | 2.0 | 10% | 用 10% 仓位 |
| 50% | 1.5 | 16.7% | 用 16.7% 仓位 |
| 60% | 1.0 | 20% | 用 20% 仓位 |

### 半凯利

```python
def half_kelly(win_rate: float, profit_loss_ratio: float) -> float:
    """半凯利 = 凯利的一半，更保守"""
    return kelly_criterion(win_rate, profit_loss_ratio) * 0.5
```

**建议**：
- 新手先用 **半凯利**：复利速度慢了，但回撤大幅降低
- 凯利公式的前提是胜率和盈亏比是稳定的——但实际上它们会变，所以算出来打个折更安全

### 多品种场景

多个独立策略时：

```python
def multi_kelly(metrics: list[dict]) -> list[float]:
    """
    多策略凯利分配
    metrics: [{"win_rate": 0.4, "profit_loss_ratio": 2.0}, ...]
    返回每个策略的权重
    """
    kellys = [kelly_criterion(m["win_rate"], m["profit_loss_ratio"]) for m in metrics]
    total = sum(kellys)
    if total <= 1:
        return kellys
    else:
        # 超过 100% 时按比例缩回来
        return [k / total for k in kellys]
```

### 从回测指标反算凯利比例

回测报告中的胜率、盈亏比、夏普比率等指标，本身就是凯利公式的输入参数。回测跑完后可以直接用这些指标反算凯利比例，不需要额外计算。

#### 方法一：从逐笔交易记录（最精确）

如果你的回测系统输出了逐笔交易的 PnL 记录，直接算：

```python
def kelly_from_trade_log(trade_log: pd.DataFrame) -> dict:
    """
    从回测的逐笔交易记录计算凯利
    
    参数:
    - trade_log: DataFrame，每行一笔交易，含 pnl 列
    """
    wins = trade_log[trade_log["pnl"] > 0]
    losses = trade_log[trade_log["pnl"] < 0]
    
    if len(wins) == 0 or len(losses) == 0:
        return {"胜率": "N/A", "盈亏比": "N/A", "全凯利": "无亏损/盈利样本"}
    
    win_rate = len(wins) / len(trade_log)
    avg_win = wins["pnl"].mean()
    avg_loss = abs(losses["pnl"].mean())
    b = avg_win / avg_loss if avg_loss > 0 else float("inf")
    
    f_full = (b * win_rate - (1 - win_rate)) / b
    f_half = f_full / 2
    f_quarter = f_full / 4
    
    return {
        "交易次数": len(trade_log),
        "胜率": f"{win_rate:.1%}",
        "盈亏比": f"{b:.2f}",
        "全凯利": f"{f_full:.1%}",
        "半凯利": f"{f_half:.1%}",
        "四分之一凯利": f"{f_quarter:.1%}",
        "建议": "半凯利" if f_full > 0.1 else "全凯利",
    }
```

#### 方法二：从收益率序列推导（基于夏普）

当策略不是"开平仓"的离散交易模式（如指数增强、因子选股），方法一不适用。此时用夏普比率推导凯利的**最优杠杆倍数**：

```text
理论公式（连续复利条件下）：
  f* = μ / σ²
  
其中：
  μ = 年化超额收益率（策略年化收益 - 无风险利率）
  σ = 年化波动率

推导：最大化 E[log(1 + f × r)] ≈ fμ - f²σ²/2
      一阶条件：μ - fσ² = 0 → f = μ/σ²
```

```python
def kelly_from_sharpe(
    annual_return: float, 
    annual_vol: float, 
    risk_free: float = 0.03
) -> float:
    """
    从回测的年化收益和波动率推算凯利最优杠杆
    
    返回：f* = μ / σ²（最优杠杆倍数）
    
    注意：这个 f* 是"杠杆倍数"而非"仓位比例"
    - f* = 1.0 → 不用杠杆
    - f* = 1.5 → 加 50% 杠杆
    - f* = 0.5 → 只用一半资金
    """
    mu = annual_return - risk_free  # 超额收益
    sigma = annual_vol
    return mu / (sigma ** 2) if sigma > 0 else 0


def kelly_from_sharpe_simple(sharpe_ratio: float, annual_vol: float) -> float:
    """用夏普比率直接推导：f* = Sharpe / σ"""
    return sharpe_ratio / annual_vol if annual_vol > 0 else 0
```

#### 两种方法的对比

| | 方法一：逐笔交易 | 方法二：夏普推导 |
|---|---|---|
| **适用场景** | 离散开平仓策略（CTA、趋势跟踪、配对交易） | 连续持仓策略（因子选股、指数增强） |
| **含义** | 每笔交易下注账户的百分之多少 | 整个策略可以加多少倍杠杆 |
| **单位** | 仓位比例（0 ~ 1） | 杠杆倍数（通常 0 ~ 100 倍） |
| **需要的数据** | 逐笔交易 PnL 日志 | 日收益率序列 |
| **截断** | 天然在 [0, 1] 之间 | 不截断，1 是"不用杠杆"基准 |

#### 实战例子

一个 CTA 趋势跟踪策略的回测结果：

| 指标 | 值 |
|------|------|
| 总交易次数 | 158 |
| 胜率 | 42% |
| 平均盈利 | +2.3% |
| 平均亏损 | -1.4% |
| 年化收益 | 18.5% |
| 年化波动率 | 22% |
| 夏普比率 | 0.70 |

```python
# 方法一：从逐笔交易记录
# win_rate = 0.42, b = 2.3 / 1.4 = 1.64
# f = (1.64 × 0.42 - 0.58) / 1.64 = 0.068 → 全凯利 6.8%
# 半凯利 = 3.4%
# → 每笔交易用 3.4% 仓位

# 方法二：从夏普推导
# μ = 0.185 - 0.03 = 0.155, σ = 0.22
# f* = 0.155 / (0.22²) = 3.20 → 最优杠杆 3.2 倍
# 半凯利 = 1.6 倍
# → 整个策略可以加 1.6 倍杠杆
```

**两个方法不是二选一，是互补的**：方法一告诉你每笔下多少，方法二告诉你总资产最多用多少倍杠杆。组合使用：每笔用 3.4% 仓位，同时总杠杆不超过 1.6 倍。


---

## 三、固定风险比例

比凯利更简单、更适合新手的方法是**固定比例风险**——每笔交易只承担账户总资金的固定比例风险。

```python
def position_size_by_risk(
    account_value: float,
    entry_price: float,
    stop_loss_price: float,
    risk_ratio: float = 0.01   # 每笔最多亏 1%
) -> int:
    """
    基于风险的仓位计算
    """
    risk_per_unit = abs(entry_price - stop_loss_price)
    total_risk = account_value * risk_ratio
    units = total_risk / risk_per_unit
    return max(1, int(units))
```

**典型风险比例参考**：

| 风险风格 | 单笔风险 | 同时持仓 | 日最大风险 |
|---------|---------|---------|-----------|
| 保守 | 0.5% | 5–10 | 2.5–5% |
| 稳健 | 1.0% | 5–10 | 5–10% |
| 激进 | 2.0% | 5–10 | 10–20% |

> **量化建议**：从保守开始，连续 3 个月稳定盈利后再上调一个档位。

---

## 四、多层级风控框架

不要只在单笔上做风控，应该构建**三层防线**：

```python
class RiskManager:
    """
    三层风控：
    1. 单笔止损（固定金额 / 固定比例）
    2. 日级风控（每日最大亏损上限）
    3. 账户级风控（最大回撤限制 + 强制减仓）
    """

    def __init__(
        self,
        account_value: float,
        per_trade_risk: float = 0.01,     # 单笔 1%
        daily_loss_limit: float = 0.03,    # 日亏损上限 3%
        max_drawdown_limit: float = 0.20,  # 最大回撤上限 20%
        position_limit: int = 10           # 最大同时持仓数
    ):
        self.account_value = account_value
        self.peak_value = account_value
        self.per_trade_risk = per_trade_risk
        self.daily_loss_limit = daily_loss_limit
        self.max_drawdown_limit = max_drawdown_limit
        self.position_limit = position_limit
        self.daily_pnl = 0
        self.current_positions = 0

    def check_entry(self, trade_value: float, stop_loss: float) -> bool:
        """检查是否允许入场"""
        # 1. 持仓上限
        if self.current_positions >= self.position_limit:
            return False

        # 2. 单笔风险
        risk = abs(trade_value - stop_loss) / self.account_value
        if risk > self.per_trade_risk:
            return False

        # 3. 当天已经亏太多了
        if self.daily_pnl <= -self.daily_loss_limit * self.account_value:
            return False

        # 4. 账户总回撤超过上限
        drawdown = (self.peak_value - self.account_value) / self.peak_value
        if drawdown >= self.max_drawdown_limit:
            return False

        return True

    def update_pnl(self, pnl: float):
        """更新当日盈亏和峰值"""
        self.daily_pnl += pnl
        self.account_value += pnl
        self.peak_value = max(self.peak_value, self.account_value)

    def get_status(self) -> dict:
        """获取当前风控状态"""
        drawdown = (self.peak_value - self.account_value) / self.peak_value
        return {
            "account_value": self.account_value,
            "drawdown": f"{drawdown:.2%}",
            "daily_pnl": f"{self.daily_pnl:.0f}",
            "positions": self.current_positions,
            "trading_allowed": drawdown < self.max_drawdown_limit
        }
```

---

## 五、回撤控制策略

### 策略级

回撤控制的核心难点不是"什么时候停"——停很容易，难的是**什么时候恢复**。恢复太早会被反复打脸（whipsaw），恢复太晚会错过策略的回血期。

下面提供四种恢复策略，按风险从低到高排列。

#### 策略 A：恢复阈值法（默认保守）

当前实现。当回撤超过阈值后，必须等回撤**恢复到阈值的一半**才恢复：

```
暂停触发：回撤 < -10%
恢复触发：回撤 > -5%（即亏到-10%后涨回到只亏5%）
```

```python
def drawdown_control_recovery(
    drawdown_series: pd.Series,
    pause_threshold: float = 0.10,
    resume_ratio: float = 0.5,
) -> pd.Series:
    """
    回撤控制：恢复阈值法
    
    参数:
    - pause_threshold: 触发暂停的回撤阈值（如 0.10 = 10%）
    - resume_ratio: 恢复阈值 = pause_threshold × resume_ratio
                    默认 0.5 → 恢复阈值为 -5%
    """
    paused = False
    signal = pd.Series(1, index=drawdown_series.index)
    resume_level = -pause_threshold * resume_ratio

    for i in range(1, len(drawdown_series)):
        if not paused and drawdown_series.iloc[i] < -pause_threshold:
            paused = True
            signal.iloc[i] = 0
        elif paused and drawdown_series.iloc[i] > resume_level:
            paused = False
        elif paused:
            signal.iloc[i] = 0

    return signal
```

**优点**：实现简单，一次性恢复，适合大多数策略。
**缺点**：如果策略在恢复后不久再次达到暂停阈值，说明策略可能失效了。

#### 策略 B：冷却期法

暂停后，不仅要等回撤恢复，还要等一个**最低冷却期**（cooldown days），防止当天 V 反后就立即入场：

```python
def drawdown_control_cooldown(
    drawdown_series: pd.Series,
    pause_threshold: float = 0.10,
    resume_ratio: float = 0.5,
    cooldown_days: int = 5,
) -> pd.Series:
    """
    回撤控制：恢复阈值 + 冷却期法
    
    参数:
    - cooldown_days: 满足恢复条件后至少再等 N 天才恢复
    """
    paused = False
    cooldown_counter = 0
    signal = pd.Series(1, index=drawdown_series.index)
    resume_level = -pause_threshold * resume_ratio

    for i in range(1, len(drawdown_series)):
        if not paused and drawdown_series.iloc[i] < -pause_threshold:
            paused = True
            cooldown_counter = 0
            signal.iloc[i] = 0
        elif paused:
            if drawdown_series.iloc[i] > resume_level:
                cooldown_counter += 1
                if cooldown_counter >= cooldown_days:
                    paused = False  # 冷却期满，恢复
                else:
                    signal.iloc[i] = 0  # 还在冷却期
            else:
                cooldown_counter = 0  # 又跌回去了，重置冷却
                # 回撤未恢复到阈值时暂停，所以这里不应该再重置暂停状态
                # 但已在这里停滞，不需要额外操作
                signal.iloc[i] = 0
        # 如果 paused 为 False，signal 保持默认值 1

    return signal
```

**优点**：防止 V 反后立即再次止损的 whipsaw 效应。
**缺点**：冷却期内如果策略大爆发，你会完全错过。

#### 策略 C：信号确认法

恢复必须有策略本身的**入场信号配合**。简单的回撤恢复不够，你必须确认趋势或因子方向仍然有效：

```python
def drawdown_control_signal(
    drawdown_series: pd.Series,
    strategy_signal: pd.Series,
    pause_threshold: float = 0.10,
    resume_ratio: float = 0.5,
) -> pd.Series:
    """
    回撤控制：信号确认法
    
    恢复条件：
    1. 回撤恢复到 resume_level 以上
    2. 策略自身信号方向与持仓一致（多/空信号至少不是反的）
    
    参数:
    - strategy_signal: 策略的入场信号（正=多头，负=空头，0=无信号）
    """
    paused = False
    signal = pd.Series(1, index=drawdown_series.index)
    resume_level = -pause_threshold * resume_ratio

    for i in range(1, len(drawdown_series)):
        if not paused and drawdown_series.iloc[i] < -pause_threshold:
            paused = True
            signal.iloc[i] = 0
        elif paused:
            drawdown_ok = drawdown_series.iloc[i] > resume_level
            signal_ok = abs(strategy_signal.iloc[i]) > 0  # 策略有明确方向信号
            if drawdown_ok and signal_ok:
                paused = False
            else:
                signal.iloc[i] = 0
        # 如果 paused 为 False，signal 保持默认值 1

    return signal
```

**优点**：不会在趋势已经反转后重新入场（这是最简单的死法）。
**缺点**：在长线 CTA 中可能错过趋势的中间恢复段。

#### 策略 D：多级渐进法（最推荐）

不是一次性全仓恢复，而是根据恢复程度逐步加仓。回撤越大，恢复仓位越慢：

```python
def drawdown_control_gradual(
    drawdown_series: pd.Series,
    strategy_signal: pd.Series,
    pause_threshold: float = 0.10,
) -> pd.Series:
    """
    回撤控制：多级渐进法
    
    返回：仓位比例（0 ~ 1），而非 0/1 开关
    - 回撤 < -10%：清仓（仓位 = 0）
    - 回撤 -10% ~ -5%：仓位 = 50%
    - 回撤 -5% ~ -3%：仓位 = 75%
    - 回撤 > -3% 且信号确认：仓位 = 100%
    """
    levels = [
        (-pause_threshold, 0.0),       # 回撤 > 10% → 空仓
        (-pause_threshold * 0.5, 0.5), # 回撤恢复到 -5% → 半仓
        (-pause_threshold * 0.3, 0.75), # 回撤恢复到 -3% → 75%
        (0.0, 1.0),                    # 回撤归零 → 满仓
    ]

    position = pd.Series(1.0, index=drawdown_series.index)

    for i in range(1, len(drawdown_series)):
        dd = drawdown_series.iloc[i]

        # 检查是否有有效信号
        has_signal = abs(strategy_signal.iloc[i]) > 0

        # 按回撤幅度决定仓位
        pct = 1.0
        for (level, target_pct) in levels:
            if dd < level:
                pct = target_pct
                break

        # 如果没有信号，即使在恢复期间也只保留最低仓位
        if not has_signal:
            pct = min(pct, 0.25)

        position.iloc[i] = pct

    return position
```

**优点**：回撤越深，恢复越谨慎；信号缺失时自动降仓。实战中最灵活。
**缺点**：比简单的 0/1 开关多一个参数，但带来的回撤控制效果值得。

#### 四种策略对比

| 策略 | 恢复条件 | whipsaw 防护 | 胜率恢复 | 实现的复杂度 |
|------|---------|-------------|---------|------------|
| A：恢复阈值法 | 回撤恢复 50% | 中 | 中 | 最低 |
| B：冷却期法 | 恢复 50% + 等 N 天 | 高 | 低（会错过早期） | 低 |
| C：信号确认法 | 恢复 50% + 信号方向 OK | 高 | 高 | 中 |
| D：多级渐进法 | 逐级恢复 + 信号确认 | 高 | 中（但波动最低） | 中 |

**建议**：
- 实盘前 3 个月 → 用 **A**（简单，先跑通流程）
- 实盘 3-6 个月后 → 升级为 **D**（多级渐进，实盘最适用）
- 信号可靠、胜率高的策略 → **C**（信号确认法，不错过反转）
- 趋势跟踪（CTA）类策略 → **D**（趋势回撤后需要时间确认恢复）

### 组合级

- 单一品种仓位不超过总资金 20%
- 同一行业不超过总资金 40%
- 相关性 > 0.7 的品种视为同一类，合并计算上限

---

## 六、黑天鹅应对

黑天鹅（极端行情）无法预测，但可以**提前准备**。

### 极端行情模拟

```python
def stress_test(returns: pd.Series, scenarios: list[tuple]) -> dict:
    """
    压力测试
    scenarios: [("2008 金融海啸", -0.05), ("2020 新冠", -0.10), ...]
    返回：每种场景下的最大回撤
    """
    current_value = 1.0
    results = {}
    for name, shock in scenarios:
        shocked_returns = returns.copy()
        shocked_returns.iloc[0] = shock  # 第一天暴跌
        cum = (1 + shocked_returns).cumprod()
        drawdown = (cum.cummax() - cum) / cum.cummax()
        results[name] = drawdown.max()
    return results
```

### 应对措施（按严重程度排序）

```text
轻度（回撤 10–15%）：
  → 降仓位至正常水平的 70%
  → 关闭新开仓（只平仓不减仓）

中度（回撤 15–25%）：
  → 降仓位至 50%
  → 关闭新开仓
  → 对冲（买入看跌期权 / 做空股指期货）

重度（回撤 > 25%）：
  → 清仓所有多头仓位
  → 转为现金或货币基金
  → 停止交易至少一周，复盘后再决策
```

---

## 七、风控检查清单

每日实盘前检查：

- [ ] 当前总仓位是否超过预设上限？
- [ ] 单笔风险是否在 1% 以内？
- [ ] 今日已亏损是否超过日限？
- [ ] 账户总回撤是否在安全线内？
- [ ] 持仓品种间相关性是否过高？
- [ ] 是否有未处理的异常日志？
- [ ] 策略信号与实盘仓位是否一致？

---

> **动手练习**：拿 backtest-101.md 的回测结果，算一下这个策略的凯利比例。然后用半凯利作为仓位，重新回测一次，对比全仓和半凯利下的最大回撤差异。
