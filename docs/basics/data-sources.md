# 数据源对比与使用

> **前置要求**：已阅读 market-finance.md 和 python-stack.md，熟悉 pandas 基本操作。
> **内容概览**：主流数据源的注册、调用、数据格式对比、适用场景。
> **更新日期**：2026-07-02

---

## 一、数据源总览

| 数据源 | 品种覆盖 | 费用 | 难度 | 推荐场景 |
|--------|---------|------|------|---------|
| **AKShare** | A 股、期货、基金、宏观 | 免费 | 低 | 入门学习、快速原型 |
| **Tushare** | A 股、期货、基金、宏观 | 积分制（注册免费） | 中 | 个人研究、因子库 |
| **Yahoo Finance** | 全球股票、ETF、期货 | 免费 | 低 | 美股/港股回测 |
| **Wind** | A 股全量 + 机构级数据 | 付费（万元级/年） | 低 | 专业研究、机构 |
| **JoinQuant / RiceQuant** | A 股、期货 | 平台免费 | 低 | 在线回测平台自带数据 |
| **交易所 API**（Binance 等） | 加密货币 | 免费 | 中 | 高频策略 |
| **iFind / 聚宽** | A 股全量 | 付费 | 低 | 专业研究 |

---

## 二、AKShare（免费首选）

国产开源数据源，**无需注册**即可使用，最适合入门。

### 安装

```bash
pip install akshare
```

### 常用接口速查

```python
import akshare as ak

# A 股日线
df = ak.stock_zh_a_hist(
    symbol="000001",        # 股票代码
    period="daily",          # daily / weekly / monthly
    start_date="20240101",
    end_date="20240630",
    adjust="qfq"             # qfq 前复权，hfq 后复权，空=不复权
)

# A 股实时行情
df = ak.stock_zh_a_spot_em()

# 期货日线
df = ak.futures_main_sina(
    symbol="RB0"             # RB0=螺纹钢主力连续
)

# 指数成份股
df = ak.index_stock_cons(
    symbol="000300"          # 沪深 300
)

# 龙虎榜、分红送配、财报日期等
df = ak.stock_dividents_cninfo(symbol="000001")
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 零注册，pip 即用 | 接口偶尔变动，需关注版本更新 |
| 覆盖品种全 | 数据质量偶有瑕疵（需交叉验证） |
| 社区活跃，文档较全 | 高频数据（分钟级）支持有限 |

---

## 三、Tushare（进阶免费）

需要注册获取 token，按积分获取不同等级的数据。免费额度对个人研究足够。

### 安装与初始化

```bash
pip install tushare
```

```python
import tushare as ts

# 注册后获取 token：https://tushare.pro
ts.set_token("your_token_here")
pro = ts.pro_api()

# A 股日线
df = pro.daily(
    ts_code="000001.SZ",
    start_date="20240101",
    end_date="20240630"
)

# 财务指标
df = pro.fina_indicator(
    ts_code="000001.SZ",
    start_date="20240101"
)

# 每日涨跌停统计
df = pro.limit_list(
    trade_date="20240601"
)
```

### 积分门槛

| 积分 | 获取方式 | 可用数据 |
|------|---------|---------|
| 100（基础） | 注册即送 | 日线、基础信息 |
| 500+ | 注册 + 邀请 | 财务数据、分钟线 |
| 2000+ | 注册 + 贡献 | 高频数据、另类数据 |

> **量化建议**：免费的 100 积分足够跑日频策略。先注册拿到 token 存到环境变量里，别硬编码在代码中。

---

## 四、Yahoo Finance（美股/港股）

```bash
pip install yfinance
```

```python
import yfinance as yf

# 港股：腾讯
df = yf.download("0700.HK", start="2024-01-01", end="2024-06-30")

# 美股：标普 500 ETF
df = yf.download("SPY", start="2024-01-01", end="2024-06-30")

# 多品种同时下载
tickers = ["0700.HK", "SPY", "TSLA"]
data = yf.download(tickers, start="2024-01-01")
```

**局限**：延迟约 15–30 分钟（非实时），部分 A 股数据不完整。

---

## 五、数据质量检查清单

拿到数据后不要直接跑策略，先做质量检查：

```python
# 1. 缺失值
print(df.isna().sum())

# 2. 价格异常（0 或负数）
print((df[["open", "high", "low", "close"]] <= 0).sum())

# 3. OHLC 逻辑检查
bad = df[(df["high"] < df["low"]) | (df["high"] < df["close"]) | (df["low"] > df["open"])]
print(f"OHLC 异常行数: {len(bad)}")

# 4. 成交量异常（停牌可能成交量为 0）
no_trade = df[df["volume"] == 0]
print(f"零成交量天数: {len(no_trade)}")

# 5. 日期连续性（跳过的交易需要确认是否真的是休市）
date_diff = df.index.to_series().diff()
print(f"最大间隔: {date_diff.max()}")
```

---

## 六、数据存储建议

不重复拉取，把原始数据落盘：

```python
# 原始数据存 CSV（日频可用，分钟频推荐 Parquet）
df.to_csv("data/raw/000001_daily.csv")

# 多品种、多年份数据推荐分文件存储
# data/raw/
# ├── stock_daily/
# │   ├── 000001.csv
# │   ├── 000002.csv
# │   └── ...
# └── futures_daily/
#     ├── RB0.csv
#     └── ...
```

> **原则**：原始数据只拉一次不做修改，清洗逻辑放在独立脚本中。这样哪天发现清洗有 bug，不需要重新拉数据。

---

## 七、数据源选择建议

| 你的情况 | 推荐 |
|---------|------|
| 刚入门，练手用 | **AKShare** — 零门槛，pip 即用 |
| 做 A 股多因子、需要财务数据 | **Tushare** — 财务数据最全的免费源 |
| 做美股/港股 | **Yahoo Finance** — 最方便 |
| 做加密货币 | **Binance API** — 官方接口，文档完善 |
| 需要分钟级或 tick 级 | 需要付费数据（Wind，Tushare 高积分，或找券商） |

对于这个知识库的学习路径，**先掌握 AKShare 就够了**，它能覆盖 80% 的练手场景。等需要更专业的数据时再切换到 Tushare 或付费源。

---

> **动手练习**：打开 Jupyter Notebook（VS Code 或 JupyterLab 都行），用 AKShare 拉取沪深 300 成份股（`index_stock_cons`），再循环下载每只股票一年日线、用上面"数据质量检查清单"检查一遍。跑通这个流程后，数据获取这一关就算过了。
