# Level 2 行情

> **来源文档**：[QMT（迅投量化交易平台）](../trading/qmt.md)、[量化必备金融知识](../../basics/market-finance.md)
> **前置要求**：理解订单簿的基本结构（买一卖一、盘口）
> **更新日期**：2026-07-03

## 一句话理解

Level 2（L2）行情就是比基础行情**更细、更深、更快**的市场数据。基础行情（Level 1）只告诉你最好的一档报价和前 5 档挂单，Level 2 告诉你 10 档挂单、每一笔真实的成交和委托队列的变化。

## Level 1（基础行情） vs Level 2 行情

在 A 股，Level 1 是所有人免费能看到的，Level 2 需要额外付费或通过特定券商开通：

| 数据维度 | Level 1（基础行情） | Level 2（深度行情） |
|---------|-------------------|-------------------|
| 盘口深度 | 5 档（买一至买五，卖一至卖五） | 10 档（买一至买十，卖一至卖十） |
| 逐笔成交 | 每 3 秒聚合一次的"快照"，非真实成交顺序 | **每一笔真实成交**的逐笔记录 |
| 委托总量 | 只有买一卖一的挂单量 | 买一和卖一的**前 50 笔**委托详情 |
| 更新频率 | 3 秒一张快照 | 0.5-1 秒（更高频的实时推送） |
| 费用 | 免费 | 通常 200-400 元/年（券商渠道） |

### 核心区别一：十档 vs 五档

```
Level 1 只告诉你：
卖⑤ 10.05  1000股
卖④ 10.04  1500股
卖③ 10.03  2000股
卖② 10.02  3000股
卖① 10.01   500股
────────────────
买① 10.00  1000股
买②  9.99  1500股
买③  9.98  2000股
买④  9.97  5000股
买⑤  9.96  8000股

Level 2 额外多出卖⑥~卖⑩和买⑥~买⑩：
卖⑩ 10.09  20000股  ← 大单卖压藏在第 8-10 档，Level 1 完全看不到
...
卖⑥ 10.06  15000股
```

### 核心区别二：逐笔成交（Tick 数据）

Level 1 告诉你的 "成交明细" 实际上是每 3 秒聚合一次的加权快照。Level 2 逐笔数据告诉你：**什么时间、以什么价格、谁主动吃单、成交了多少**。

```python
# Level 2 逐笔成交数据样例
# 每一行代表一笔真实成交
tick_data = [
    {"time": "09:30:01.250", "price": 10.01, "volume": 500,  "side": "buy"},   # 主动买入
    {"time": "09:30:01.320", "price": 10.01, "volume": 200,  "side": "sell"},  # 主动卖出
    {"time": "09:30:01.410", "price": 10.02, "volume": 1000, "side": "buy"},   # 主动买入
    {"time": "09:30:01.500", "price": 10.02, "volume": 3000, "side": "buy"},   # 大单买入
]
```

## 量化中如何使用 Level 2

### 1. 订单簿压力分析

```python
def order_book_imbalance(bid_volumes, ask_volumes):
    """
    订单簿不平衡率（Order Book Imbalance）
    
    正数 = 买方力量更强，负数 = 卖方力量更强
    常用于高频信号：OB 极端时预示着短期价格方向
    """
    total_bid = sum(bid_volumes)
    total_ask = sum(ask_volumes)
    return (total_bid - total_ask) / (total_bid + total_ask)


def weighted_mid_price(bid_prices, bid_volumes, ask_prices, ask_volumes):
    """
    加权中间价——比简单(买一+卖一)/2 更能反映真实交易成本
    """
    weighted_bid = sum(p * v for p, v in zip(bid_prices, bid_volumes)) / sum(bid_volumes)
    weighted_ask = sum(p * v for p, v in zip(ask_prices, ask_volumes)) / sum(ask_volumes)
    return (weighted_bid + weighted_ask) / 2
```

### 2. 大单识别（逐笔数据）

```python
def detect_large_trades(tick_data, volume_threshold=10000):
    """
    从逐笔成交中识别大单交易
    
    A 股市场主力常把大单拆成多笔，逐笔数据能在一定程度上
    通过"同一毫秒、同一方向"来合并识别拆分大单
    """
    large_trades = []
    for tick in tick_data:
        if tick["volume"] >= volume_threshold:
            large_trades.append(tick)
    
    # 大单买入/卖出比例
    buy_volume = sum(t["volume"] for t in large_trades if t["side"] == "buy")
    sell_volume = sum(t["volume"] for t in large_trades if t["side"] == "sell")
    
    return {
        "total_large": len(large_trades),
        "buy_volume": buy_volume,
        "sell_volume": sell_volume,
        "ratio": buy_volume / sell_volume if sell_volume > 0 else float("inf"),
    }
```

### 3. 委托队列分析（Level 2 独有）

Level 2 提供了买一和卖一位置的前 50 笔委托明细。这可以帮你判断：

- 买一上那 1000 股是**1 个人挂的**还是**100 个人各挂 10 股**的？
- 如果是集中挂单，可能是主力伪装买盘；如果是分散挂单，说明自然买盘充足

```python
def analyze_order_queue(buy_queue, sell_queue):
    """
    委托队列分析
    
    集中挂单：少数人挂大单（可能是假支撑）
    分散挂单：多数人挂小单（真实需求）
    """
    buy_orders = len(buy_queue)      # 委托笔数
    buy_total = sum(buy_queue)        # 总挂单量
    buy_concentration = max(buy_queue) / buy_total  # 集中度
    
    sell_orders = len(sell_queue)
    sell_total = sum(sell_queue)
    sell_concentration = max(sell_queue) / sell_total
    
    return {
        "买盘集中度": buy_concentration,
        "卖盘集中度": sell_concentration,
        # 集中度高（>50%）说明某一个人的挂单占了大部分，不可信
    }
```

## 上交所 vs 深交所的 Level 2 差异

A 股的两家交易所 Level 2 数据格式不同：

| 维度 | 上交所（SSE） | 深交所（SZSE） |
|------|-------------|-------------|
| 逐笔成交 | 全量逐笔（包含所有成交） | 全量逐笔 |
| 逐笔委托 | 有（可以重建订单簿） | 无（只有快照） |
| 盘口 | 10 档 | 10 档 |
| 委托队列 | 买一卖一前 50 笔 | 买一卖一前 50 笔 |
| 数据频率 | 实时推送 | 约 0.5-1 秒快照 |

## 实盘获取方式

| 途径 | 费用 | 延迟 | 适合 |
|------|------|------|------|
| QMT / 迅投 Level 2 | 包含在 QMT 服务中 | 低 | 已用 QMT 的用户 |
| 券商 APP 付费 | 约 200-400 元/年 | 中 | 散户看盘 |
| 万得（Wind） | 贵（机构授权） | 低 | 机构研究 |
| 通达信 Level 2 | 约 200 元/年 | 中 | 看盘 + 公式选股 |
| 自研对接交易所 | 数据费 + 机房费 | 最低 | 高频交易机构 |

## 常见误区

1. **"有了 Level 2 就能赚钱"**：不是。Level 2 提供了更多信息，但能从这些信息中提炼出什么信号，取决于你的策略。大多数人开了 Level 2 后也只是"看看盘"。
2. **"Level 2 数据频率越高越好"**：对于日频或小时级策略，Level 2 毫无意义——信号周期比数据频率短反而会引入噪声。
3. **"Level 1 就够了，Level 2 没用"**：对于因子选股（多因子周频调仓），Level 1 确实够了。对于日内策略、算法执行、订单簿策略，没有 Level 2 等于闭着眼交易。
4. **"QMT 的 Level 2 就是全市场都看得见"**：QMT 的 Level 2 只覆盖你开通了的品种，并非全市场。需要逐一开通权限。

## 判断是否需要 Level 2

```text
你的策略类型 → 是否需要 Level 2？
├── 日频 / 周频因子选股 → 不需要，Level 1 足够
├── 分钟级 CTA      → 不一定，分钟 K 线的 OHLCV 够了
├── 日内高频策略    → 需要，尤其是逐笔成交和订单簿
├── 算法交易（TWAP/VWAP）→ 需要，用来优化执行成本
└── 做市策略        → 必需，没有 Level 2 无法做市
```

## 回到来源文档

[← 返回 QMT（迅投量化交易平台）](../trading/qmt.md)
