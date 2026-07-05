# 实盘部署与接口选型

> **前置要求**：已阅读 risk-management.md，理解仓位管理与风控框架。
> **内容概览**：交易接口对比、实盘架构、日志监控、灰度上线流程。
> **更新日期**：2026-07-03

---

## 一、交易接口选型

| 接口 | 适用品种 | 协议 | 难度 | 费用 |
|------|---------|------|------|------|
| **CTP** | 期货（中金所、上期所等） | TCP Socket | 中 | 期货公司提供 |
| **XTP** | A 股 | TCP Socket | 中 | 券商提供 |
| **REST API** | 加密货币（Binance、OKX） | HTTP/WebSocket | 低 | 交易所免费 |
| **[QMT](../glossary/trading/qmt.md)** / PTrade | A 股（券商自带） | 图形化/脚本 | 低 | 券商提供 |
| **掘金量化** | A 股、期货 | SDK | 中 | 部分免费 |
| **vn.py** | A 股、期货**全套** | 框架 | 高 | 开源 |

### 新手选型建议

```text
你的量化方向？
├── A 股、资金 < 50 万       → 先用 QMT / 掘金量化（券商自带，省去接口开发）
├── A 股、资金 > 50 万       → XTP（速度最快，适合中高频）
├── 期货                     → CTP（行业标准，没有替代）
└── 加密货币                 → Binance REST API + WebSocket
```

**对于这个知识库的学习路径**：找一家支持 **QMT** 的券商开户，用 A 股本地环境跑通实盘。数据端用 **AKShare**（免费行情），执行端用 QMT（自动下单）。从这个路径开始理解实盘流程，因为它是 A 股量化入门的标准入口，文档和社区资源最完善。

---

## 二、A 股实盘数据与接口

A 股和加密货币的一个核心差异：A 股没有统一的公开 REST API，交易需要通过**券商提供的渠道**接入。最常见的方式是 **QMT**（本地脚本执行）和 **XTP**（底层 API 接口）。

### 数据端：AKShare

无论你用哪种执行方式，行情数据都可以用 AKShare 统一获取：

```python
import akshare as ak
import pandas as pd

def get_stock_data(symbol: str = "000001", start: str = "2025-01-01"):
    """用 AKShare 获取 A 股日线数据"""
    df = ak.stock_zh_a_hist(
        symbol=symbol,
        period="daily",
        start_date=start,
        adjust="qfq"  # 前复权
    )
    df["日期"] = pd.to_datetime(df["日期"])
    df = df.sort_values("日期").reset_index(drop=True)
    return df
```

### 执行端方案一：QMT（入门最快）

QMT 不通过 HTTP API 调用，而是在 **QMT 客户端中直接运行本地 Python 脚本**。你写好策略 .py 文件放入 QMT 的策略目录，QMT 自动调度：

```python
# QMT 策略骨架（放在 QMT 策略目录下运行）
# 不需要写网络请求、不需要 API Key——QMT 已自动连接券商柜台

def init(context):
    """策略初始化——QMT 启动时自动调用"""
    context.stock = "000001.SZ"          # 平安银行
    context.ma_short = 5
    context.ma_long = 20


def handlebar(context):
    """每根 K 线回调一次——QMT 自动传入实时行情"""
    # 获取历史收盘价
    close = context.get_close(context.stock, context.ma_long)
    if len(close) < context.ma_long:
        return

    # 均线信号
    ma_short = close[-context.ma_short:].mean()
    ma_long = close.mean()

    current_pos = context.get_position(context.stock)

    if ma_short > ma_long and current_pos == 0:
        context.order_target_percent(context.stock, 0.9, order_type="market")
        log_trade(f"买入 {context.stock}，MA5={ma_short:.2f} > MA{context.ma_long}={ma_long:.2f}")
    elif ma_short < ma_long and current_pos > 0:
        context.order_target_percent(context.stock, 0, order_type="market")
        log_trade(f"卖出 {context.stock}，MA5={ma_short:.2f} < MA{context.ma_long}={ma_long:.2f}")
```

### 执行端方案二：XTP（中高频）

XTP（中泰证券极速交易平台）是目前 A 股最快的交易接口。使用时需要通过 `xtp_client` SDK 连接：

```python
# XTP 示例骨架——需要券商开通 XTP 权限

def xtp_demo():
    """XTP 连接示范（伪代码，实际需要券商 SDK）"""
    # 1. 创建交易客户端
    client = XTPTradeClient()

    # 2. 登录（需券商提供的软证书或账号密码）
    client.login(ip="192.168.x.x", port=6001, user="your_id", password="your_pwd")

    # 3. 下单
    order_id = client.insert_order(
        symbol="000001.SZ",
        price=12.50,
        quantity=1000,
        side=ORDER_BUY,
        order_type=ORDER_LIMIT,
    )
    print(f"订单已提交, ID: {order_id}")

    # 4. 查询持仓
    positions = client.query_all_positions()
    print(positions)
```

> XTP 的连接流程比 Binance REST 复杂，涉及 TCP 长连接、心跳包、重连逻辑等。初学者建议先用 QMT 跑通流程，后期再迁移到 XTP。

### 安全规范

A 股交易安全的关键点与加密货币不同——QMT/XTP 不需要在代码中存储 API Key，但需要注意以下几点：

```python
# 1. QMT 策略文件中不要存储交易密码——QMT 在客户端登录时已建立会话
#
# 2. 环境变量依然有用——用于风控参数和通知渠道配置
import os
from dotenv import load_dotenv
load_dotenv(".env")

RISK_CONFIG = {
    "max_position_pct": float(os.getenv("MAX_POSITION", "0.9")),
    "stop_loss_pct": float(os.getenv("STOP_LOSS", "0.05")),
    "max_daily_loss": float(os.getenv("MAX_DAILY_LOSS", "0.03")),
}

# 3. 通知渠道 Webhook（通过环境变量避免硬编码）
WEBHOOK_URL = os.getenv("DINGTALK_WEBHOOK_URL", "")

# 4. QMT 本地 .py 文件的安全
#    - 策略源码以明文存储，不要在代码中写密码
#    - 如果共享电脑，考虑策略源码加密
```

### 查询持仓与成交

一个需要提前说清楚的差异：**AKShare 能免费查行情，但行情是公开数据。你的持仓和成交是个人账户数据，不存在免费的公开 API。**

就像你不可能用 Google 搜到别人的银行流水一样，没有免费的公开接口能查询"某账户的持仓"。只能通过**你自己的券商认证**来查。

### 方案一：QMT 内置接口（最方便）

如果你已经有 QMT 环境，它提供了查持仓和查成交的内置函数：

```python
# QMT 策略中查询持仓
def query_position(context):
    """查询当前持仓"""
    all_positions = context.get_total_positions()
    for pos in all_positions:
        print(f"{pos.m_strCode}: {pos.m_nVolume} 股, 成本 {pos.m_dOpenPrice:.2f}")

    # 查询指定品种
    stock_pos = context.get_position("000001.SZ")
    if stock_pos:
        print(f"持有 {stock_pos.m_nVolume} 股")
    else:
        print("无持仓")


def query_trade_history(context):
    """查询当日成交记录"""
    trades = context.get_trade_records()
    for t in trades:
        print(f"{t.m_strCode} | {'买入' if t.m_direction == 0 else '卖出'} "
              f"| {t.m_nVolume} 股 @ {t.m_dPrice:.2f}")
```

### 方案二：XTP 接口（中高频）

```python
# XTP 查询持仓和成交（伪代码）
positions = client.query_all_positions()
for pos in positions:
    print(f"{pos.ticker}: 持仓 {pos.quantity} 股, 浮动盈亏 {pos.float_pnl:.2f}")

# 查询历史成交
trades = client.query_trades(begin_time="2026-01-01", end_time="2026-07-03")
```

### 方案三：券商 APP 导出（无编程接口时）

没有 QMT/XTP 的情况下，最实际的方案：

```python
# 大多数券商 APP 支持在 PC 端导出成交记录为 Excel/CSV
# 路径示例：券商 APP → 交易 → 查询 → 历史成交 → 导出
# 然后用 pandas 读取：
import pandas as pd

# 导出的 CSV 文件路径
df_trades = pd.read_csv("成交记录.csv", encoding="gbk")
df_positions = pd.read_csv("持仓查询.csv", encoding="gbk")
```

每次交易后手动导出一次，在 Notebook 中做持仓监控。做不到实时，但足够做日终对账。

### 方案四：Selenium 自动化券商网页端（不推荐）

```python
# 技术上行得通但风险较高：
# 1. 违反大多数券商的服务条款
# 2. 券商网页端改版后需要重写爬虫
# 3. 增加了账号信息泄露的风险
# 4. 验证码/二次认证可能阻断自动化

# 如果你仍然有这个需求，搜索关键词：selenium + 华泰/平安/银河 + 自动登录
# 但后果自负
```

### 四种方案对比

| 方案 | 实时性 | 需要券商 | 实现难度 | 推荐场景 |
|------|--------|---------|---------|---------|
| QMT 内置接口 | 实时 | 是 | 低（已内置） | **最推荐**，有 QMT 就用它 |
| XTP 查询 | 实时 | 是 | 中 | 高频策略需要实时持仓 |
| 券商 APP 导出 | T+1 | 是 | 低 | 研究阶段，无 QMT 可用 |
| Selenium 爬虫 | 准实时 | 是 | 高 | 不推荐（风险 > 收益） |

**总结**：不存在免费的"查持仓 API"——持仓是你的个人数据，只有你的券商知道。最实际的路径是：**开一个支持 QMT 的券商账户**，然后用 `context.get_position()` 免费、实时地查。


## 三、实盘架构设计

一个最小的实盘系统需要三个模块：

```
                  ┌─────────────────────┐
                  │    策略模块          │
                  │ 接收行情 → 生成信号   │
                  └────────┬────────────┘
                           │ 信号（买入/卖出/数量）
                           ▼
                  ┌─────────────────────┐
                  │    执行模块          │
                  │ 连接交易所 → 下单     │
                  │ 查持仓 → 查成交      │
                  └────────┬────────────┘
                           │ 成交回报
                           ▼
                  ┌─────────────────────┐
                  │    监控模块          │
                  │ 记录日志 → 异常报警   │
                  │ 资金曲线 → 风控检查  │
                  └─────────────────────┘
```

### 最小实现骨架

```python
class LiveTradingSystem:
    """
    最小实盘系统骨架（A 股版本）
    数据端用 AKShare，信号和日志本地计算，执行端通过 QMT/XTP
    """

    def __init__(self, symbol: str = "000001"):
        self.symbol = symbol
        self.position = 0
        self.trades = []  # 交易日志

    def fetch_market_data(self) -> pd.DataFrame:
        """用 AKShare 获取 A 股日线数据"""
        import akshare as ak
        df = ak.stock_zh_a_hist(
            symbol=self.symbol,
            period="daily",
            start_date="2025-01-01",
            adjust="qfq"
        )
        df["收盘"] = df["收盘"].astype(float)
        df["日期"] = pd.to_datetime(df["日期"])
        return df

    def generate_signal(self, df: pd.DataFrame) -> int:
        """策略信号（双均线）"""
        close = df["收盘"]
        ma_short = close.rolling(5).mean().iloc[-1]
        ma_long  = close.rolling(20).mean().iloc[-1]
        return 1 if ma_short > ma_long else 0

    def execute_signal(self, signal: int):
        """执行信号：记录日志（QMT 环境中由 handlebar 自动执行下单）"""
        df = self.fetch_market_data()
        price = float(df["收盘"].iloc[-1])
        date = df["日期"].iloc[-1]

        if signal == 1 and self.position == 0:
            self.position = 1
            self.trades.append({
                "time": str(date),
                "symbol": self.symbol,
                "action": "BUY",
                "price": price,
                "quantity": 1000
            })
            send_alert(f"买入信号: {self.symbol} @ {price:.2f}", "INFO")
        elif signal == 0 and self.position == 1:
            self.position = 0
            self.trades.append({
                "time": str(date),
                "symbol": self.symbol,
                "action": "SELL",
                "price": price,
                "quantity": 1000
            })
            send_alert(f"卖出信号: {self.symbol} @ {price:.2f}", "INFO")

    def run_once(self):
        """运行一个周期"""
        df = self.fetch_market_data()
        signal = self.generate_signal(df)
        self.execute_signal(signal)
```

---

## 四、日志与监控

实盘系统最重要的不是策略收益，而是**出了问题能第一时间知道**。

### 交易记录

```python
import json
from pathlib import Path

def log_trade(trade: dict, log_dir: Path = Path("logs/trades")):
    """记录每笔交易到 JSON 文件"""
    log_dir.mkdir(parents=True, exist_ok=True)
    date = pd.Timestamp.now().strftime("%Y%m%d")
    file_path = log_dir / f"{date}.jsonl"
    with open(file_path, "a") as f:
        f.write(json.dumps(trade, default=str) + "\n")
```

### 异常报警

```python
def send_alert(message: str, level: str = "WARN"):
    """发送告警（示例：打印到终端 + 写入日志）"""
    timestamp = pd.Timestamp.now()
    print(f"[{timestamp}] [{level}] {message}")
    # 实际可以对接：
    # - 钉钉/企业微信 Webhook
    # - Telegram Bot
    # - 邮件
```

---

## 五、灰度上线流程

不要做完回测就直接上实盘，走完三步：

```text
第一步：模拟盘（Paper Trading）
  - 接实时行情，但不下真实单
  - 验证策略在真实市场环境下的行为
  - 时长：至少 1 个月
  ↓

第二步：小资金实盘
  - 投入资金的 5-10%
  - 只跑策略的 50% 仓位
  - 验证执行环节（接口、滑点、延迟）
  - 时长：至少 1 个月
  ↓

第三步：逐步加仓
  - 每 2 周加 10-20%
  - 持续监控回撤和异常
  - 到 100% 仓位后再观察 3 个月
```

### 模拟盘实现

```python
class PaperTrading(LiveTradingSystem):
    """模拟盘：不下真实单，记录信号（A 股版本）"""

    def execute_signal(self, signal: int):
        df = self.fetch_market_data()
        price = float(df["收盘"].iloc[-1])
        date = df["日期"].iloc[-1] if "日期" in df.columns else str(pd.Timestamp.now())[:10]

        if signal == 1 and self.position == 0:
            self.position = 1
            self.trades.append({"time": str(date), "action": "BUY", "price": price})
            send_alert(f"模拟买入 {self.symbol} @ {price:.2f} ({date})", "INFO")
        elif signal == 0 and self.position == 1:
            self.position = 0
            self.trades.append({"time": str(date), "action": "SELL", "price": price})
            send_alert(f"模拟卖出 {self.symbol} @ {price:.2f} ({date})", "INFO")
```

---

## 六、实盘常见坑

| 坑 | 表现 | 预防 |
|----|------|------|
| **API Key 泄露** | 账户被盗刷 | 环境变量 + 只启用交易权限 |
| **交易所宕机** | 无法下单、无法撤单 | 准备备用接口或经纪商 |
| **网络延迟** | 滑点放大、撤单不及时 | 使用同机房服务器 |
| **代码异常中断** | 持仓暴露无风控 | try/except 兜底 + 异常报警 |
| **交易所规则变更** | 下单失败、费率变化 | 订阅官方公告渠道 |
| **T+1 交易规则** | 当天买入无法卖出，止损策略失效 | 策略必须至少隔夜；考虑日末持仓风险 |
| **涨跌停限制** | 跌停时无法止损，涨停时无法买入 | 回测中加入涨跌停过滤；实盘中设定替代方案 |
| **ST / 退市风险** | 持仓股被 ST 后连续跌停无法平仓 | 策略中加入 ST 排除逻辑；设置品种白名单 |
| **分红除权跳空** | 除权日技术指标失真 | 回测和实盘都使用前复权数据 |
| **节假日休市** | 非预期休市导致持仓暴露 | 交易日历订阅；策略中加入节前减仓规则 |

---

## 七、上线前检查清单

- [ ] API Key 用环境变量，没硬编码
- [ ] API Key 只开了交易权限（没开提现）
- [ ] 每日风控限额已配置（最大亏损、最大回撤）
- [ ] 异常报警渠道已测试（能收到消息）
- [ ] 模拟盘已运行 ≥ 1 个月，策略表现符合预期
- [ ] 交易日志能正常写入和查询
- [ ] 回测中的交易成本与实盘基本一致
- [ ] 小资金实盘已跑通一次完整的买入→持有→卖出流程

---

> **动手练习**：打开 Jupyter Notebook，用 AKShare 获取平安银行（000001）近 1 年日线数据。在 Notebook 上实现双均线策略的信号生成，并用 `PaperTrading` 类跑一次模拟盘——不下真实单，只记录每天的买卖信号和模拟持仓。核心目标是验证策略的触发频率是否合理（信号过于频繁 --> 可能过拟合）。
