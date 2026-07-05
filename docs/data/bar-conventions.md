# K 线数据规范

> **前置要求**：已阅读 market-finance.md 的"除权除息与复权"章节，了解复权基本概念。
> **内容概览**：K 线形成原理、周期选择、复权深入、缺失值处理、期货换月拼接、多频率对齐。
> **更新日期**：2026-07-02

---

## 一、K 线的基本结构

K 线包含四个价格和两个附加字段：

| 字段 | 含义 | 量化中的常见问题 |
|------|------|-----------------|
| `open` | 开盘价 | 集合竞价产生，分钟线第一笔成交 |
| `high` | 最高价 | 可能被一分钟内的异常大单拉高 |
| `low`  | 最低价 | 可能被异常大单砸低，需和 high 联合校验 |
| `close` | 收盘价 | 最常用的字段，也是因子计算的主要输入 |
| `volume` | 成交量 | 停牌日为 0，复权时需要反向调整 |
| `amount` | 成交额 | 比 volume 更可靠（不受复权影响） |

**基础校验条件**：

```python
assert (df["high"] >= df["low"]).all()          # high >= low
assert (df["high"] >= df["close"]).all()        # high 是最高
assert (df["low"] <= df["open"]).all()          # low 是最低
assert (df["open"] >= 0).all()                  # 不能为负
```

---

## 二、时间周期选择

### 日线

最常用的周期，中低频策略（持有期 ≥ 1 天）用日线足够。

```python
# 日线的形成：当天 9:30–15:00 的所有成交
# open  = 1st trade
# close = last trade
# high  = max price
# low   = min price
# volume = sum of volume
```

### 分钟线

日内策略（持有期 < 1 天）才需要分钟线。A 股每分钟线在 9:31–15:00 之间落地，**9:30 的数据是集合竞价后的第一笔**，需要单独处理。

```python
# 分钟线重采样为小时线
hourly = df.resample("1h").agg({
    "open": "first",
    "high": "max",
    "low": "min",
    "close": "last",
    "volume": "sum"
})
```

### 周线 / 月线

```python
weekly = df.resample("W").agg({
    "open": "first",        # 周一开盘
    "high": "max",          # 本周最高
    "low": "min",           # 本周最低
    "close": "last",        # 周五收盘
    "volume": "sum"
})

monthly = df.resample("M").agg({
    "open": "first",
    "high": "max",
    "low": "min",
    "close": "last",
    "volume": "sum"
})
```

**周期选择原则**：能不用分钟线就不用，数据量和噪声都小一个数量级。

---

## 三、复权方法深入（核心）

### 前复权 vs 后复权的数学关系

```
前复权价格 = 原始价格 × (当前 adj_factor / 当日 adj_factor)
后复权价格 = 原始价格 × 当日 adj_factor
```

`adj_factor` 是数据源提供的累计复权因子。如果数据源没有提供，可以用除权除息记录手动计算：

```python
def build_adj_factor(dividends: pd.Series) -> pd.Series:
    """
    dividends: index=除权除息日期, values=每股分红+送转比例调整
    返回：每个交易日的累计复权因子
    """
    # 生成每日复权因子（从最早到最晚）
    adj = pd.Series(1.0, index=trade_dates)
    for date, factor in dividends.items():
        # 除权除息日之后的因子全部乘以 (1 + factor)
        adj.loc[date:] *= (1 + factor)
    return adj
```

### 成交量的复权调整

成交量也需要调整。除权除息后，由于股价发生变化，同样的成交笔数对应的金额变化，成交量需要除以调整因子：

```python
def adjust_volume(df: pd.DataFrame, adj_factor_col: str = "adj_factor"):
    """复权调整成交量"""
    df = df.copy()
    # 成交量调整方向：前复权成交量 = 原始成交量 / 复权因子
    df["volume_adj"] = df["volume"] / df[adj_factor_col]
    # 后复权成交量 = 原始成交量 × 复权因子
    df["volume_unadj"] = df["volume"]
    return df
```

### 常见复权 Bug

1. **复权因子匹配错时间索引**：除权除息日的价格已经包含了除权影响，不要对当日价格做额外调整
2. **多品种复权因子混用**：每只股票有独立的除权除息记录，不要用同一个因子
3. **前复权数据的 NaN 问题**：最早期的前复权价格可能为负（如果公司历史上分红金额超过股价），通常截掉早期数据

---

## 四、缺失值与异常值处理

### 缺失值来源

| 原因 | 表现 | 处理方式 |
|------|------|---------|
| 停牌 | volume=0，价格不变 | ffill 前先标记，或保持 NaN |
| 节假日差异 | 港股休市 A 股开市 | 用 reindex 对齐 |
| 数据源缺漏 | 某天整行缺失 | 检查日期连续性，用 ffill 或 bfill |
| IPO 前 | 上市前无数据 | 上市日起才有数据，勿填充 |

### 推荐的缺失值处理流程

```python
def clean_bars(df: pd.DataFrame) -> pd.DataFrame:
    """标准数据清洗流程"""
    df = df.copy()

    # 1. 标记停牌日
    df["is_suspended"] = df["volume"] == 0

    # 2. 价格缺失：用前向填充（停牌日不填充）
    df.loc[df["is_suspended"], ["open", "high", "low", "close"]] = df.shift(1)
    # 更严谨：只填充非停牌日，停牌日保持最后一笔价格即可

    # 3. 异常值：涨跌停日检查
    df["is_limit_up"]   = df["close"] >= df["close"].shift(1) * 1.095
    df["is_limit_down"] = df["close"] <= df["close"].shift(1) * 0.905

    # 4. OHLC 逻辑校验
    ohlc_bad = (df["high"] < df["low"]) | (df["high"] < df["close"]) | (df["low"] > df["open"])
    if ohlc_bad.any():
        print(f"WARN: {ohlc_bad.sum()} rows with invalid OHLC")

    return df
```

---

## 五、期货连续合约拼接

期货面临换月问题，一个合约到期后要切换到下一个主力合约。不同拼接方式产生不同的连续合约：

| 方式 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| **主力连续** | 选持仓量最大的合约 | 反映流动性最好的合约 | 换月日有跳空 |
| **前复权连续** | 按换月价差调整历史价格 | 无跳空 | 后期价格偏离实际 |
| **持仓量加权** | 多合约按持仓量加权平均 | 平滑 | 不反映实际交易价格 |

### Python 实现主力连续切换

```python
def build_continuous_series(
    contract_dfs: dict[str, pd.DataFrame],
    roll_method: str = "volume"
) -> pd.DataFrame:
    """
    从多个合约拼接成连续合约
    contract_dfs: {合约代码: 日线 DataFrame}
    roll_method: 'volume' 按成交量选主力, 'oi' 按持仓量
    """
    # 1. 找到每个交易日的主力合约
    all_data = []
    for code, df in contract_dfs.items():
        df = df.copy()
        df["contract"] = code
        all_data.append(df)
    combined = pd.concat(all_data)

    # 2. 每个交易日选持仓量/成交量最大的合约
    if roll_method == "oi":
        dominant = combined.loc[combined.groupby("date")["open_interest"].idxmax()]
    else:
        dominant = combined.loc[combined.groupby("date")["volume"].idxmax()]

    # 3. 按日期排序
    dominant = dominant.sort_index()

    # 4. 标记换月日
    dominant["roll_date"] = dominant["contract"] != dominant["contract"].shift(1)

    return dominant
```

---

## 六、多频率数据对齐

做实盘或因子计算时，经常需要把不同频率的数据对齐到同一时间轴：

```python
# 场景：把分钟线揉进日线，每分钟得到一个"当日累计"指标
df_daily = df_minute.resample("D").agg({
    "open": "first",
    "high": "max",
    "low": "min",
    "close": "last",
    "volume": "sum"
})

# 场景：日频因子值 forward fill 到分钟线
df_minute["factor_value"] = df_daily["factor_value"].reindex(
    df_minute.index, method="ffill"
)

# 场景：多品种对齐到同一时间轴
dates = pd.date_range("2024-01-01", "2024-12-31", freq="D")
aligned = pd.DataFrame({"date": dates})
for code in ["000001", "000002", "000003"]:
    aligned[code] = (
        df_dict[code]["close"]
        .reindex(dates, method="ffill")
        .values
    )
```

---

## 七、完整数据质量报告函数

```python
def data_quality_report(df: pd.DataFrame, name: str = "data"):
    """打印数据质量检查报告"""
    print(f"\n{'='*40}")
    print(f"  {name}")
    print(f"{'='*40}")
    print(f"  日期范围: {df.index.min()} → {df.index.max()}")
    print(f"  交易日数: {len(df)}")

    # 连续性
    date_diff = df.index.to_series().diff().dt.days
    max_gap = date_diff.max()
    print(f"  最大间隔: {max_gap} 天")
    if max_gap and max_gap > 3:
        gap_dates = df.index[date_diff > pd.Timedelta(days=3)].tolist()
        print(f"  ⚠ 长间隔日期: {gap_dates[:3]}")

    # 缺失
    na_count = df.isna().sum()
    if na_count.any():
        print(f"  缺失值:\n{na_count[na_count > 0]}")

    # OHLC 异常
    bad = (df["high"] < df["low"]) | (df["high"] < df["close"])
    if bad.any():
        print(f"  ⚠ OHLC 异常: {bad.sum()} 行")

    # 停牌比例
    suspend_ratio = (df["volume"] == 0).mean()
    if suspend_ratio > 0:
        print(f"  停牌比例: {suspend_ratio:.2%}")

    print(f"{'='*40}\n")
```

> **进阶阅读**：[收益率计算专题](../basics/returns-calculation.md) —— K 线数据拿到后，下一步就是算收益率。

---

> **动手练习**：用 AKShare 拉取一只股票一年日线，运行上面的 `data_quality_report`；再用分钟线合成日线，对比数据源直接返回的日线和你自己合成的日线差异，理解"OHLC 怎么来的"。
