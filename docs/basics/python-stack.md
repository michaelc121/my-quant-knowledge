# Python 量化工具栈速查

> **前置要求**：已了解 Python 基础语法（变量、函数、列表、字典）。
> **内容概览**：pandas / numpy / matplotlib / plotly / Jupyter 在量化场景中的核心用法。
> **更新日期**：2026-07-02

---

## 一、pandas —— 量化数据处理核心

量化中 80% 的数据操作都落在 pandas 身上，重点掌握以下模式。

### 读取与预览

```python
import pandas as pd

# 从 CSV 读取，指定日期列并解析为时间索引
df = pd.read_csv("data/000300.csv", parse_dates=["date"], index_col="date")
df.info()           # 列类型、缺失值、内存占用
df.head(3)          # 前 3 行
df.tail(3)          # 后 3 行
df.describe()       # 各列统计摘要
```

### 列操作

```python
# 选取列
prices = df["close"]                # Series
subset = df[["open", "close", "volume"]]  # DataFrame

# 新增列（向量化，不要用 for 循环）
df["daily_return"] = df["close"].pct_change()
df["log_return"]   = np.log(df["close"] / df["close"].shift(1))
df["ma_20"]        = df["close"].rolling(20).mean()
```

### 时间序列索引

```python
# 切片
df.loc["2024-01":"2024-06"]         # 按年份/月份切片
df.loc["2024-03-15":"2024-04-10"]   # 按日期区间

# 重采样（日转周、日转月）
weekly = df["close"].resample("W").last()    # 周线：取每周末尾
monthly = df["close"].resample("M").last()    # 月线
weekly_return = df["close"].resample("W").apply(lambda s: s.iloc[-1] / s.iloc[0] - 1)
```

### 滚动计算

```python
# 滚动窗口统计
df["ma_5"]   = df["close"].rolling(5).mean()
df["ma_20"]  = df["close"].rolling(20).mean()
df["std_20"] = df["close"].rolling(20).std()
df["max_20"] = df["close"].rolling(20).max()

# 滚动相关系数（两只股票）
roll_corr = df_1["close"].rolling(60).corr(df_2["close"])
```

### 合并多个品种

```python
# 按列拼接：每一列是一只股票
pivot = df.pivot_table(
    index="date", columns="stock_code", values="close"
)

# 多 DataFrame 合并
merged = pd.merge(
    left=df_a, right=df_b,
    on="date", how="inner", suffixes=("_a", "_b")
)
```

### 缺失值处理

```python
df.isna().sum()                     # 查看每列缺失数
df.fillna(method="ffill")           # 前向填充（最常用）
df.fillna(method="bfill")           # 后向填充
df.dropna()                         # 丢弃含缺失值的行
```

> **量化注意**：用 `ffill` 填充 K 线数据前务必确认停牌标记，不要把停牌期间的缺失数据直接填充成正常行情。

---

## 二、numpy —— 数值计算基础

pandas 底层依赖 numpy，独立掌握几个常用模块即可。

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5])

# 统计函数
np.mean(arr), np.std(arr), np.median(arr), np.percentile(arr, 25)

# 对数与指数（收益率计算）
returns = np.log(close / close.shift(1))

# 线性代数（多因子回归时用到）
X = np.array([[1, 2], [3, 4], [5, 6]])
y = np.array([1, 2, 3])
beta = np.linalg.lstsq(X, y, rcond=None)[0]   # 最小二乘

# 随机数（回测模拟用）
np.random.seed(42)
np.random.normal(0, 0.01, 1000)     # 正态分布模拟收益率
```

---

## 三、matplotlib / plotly —— 可视化

### matplotlib：静态图表，适合文档

```python
import matplotlib.pyplot as plt

# 设置中文字体（macOS）
plt.rcParams["font.sans-serif"] = ["Arial Unicode MS"]
plt.rcParams["axes.unicode_minus"] = False

# K 线 + 均线
plt.figure(figsize=(12, 6))
plt.plot(df.index, df["close"], label="close", linewidth=1)
plt.plot(df.index, df["ma_20"], label="ma_20", linestyle="--")
plt.title("沪深300 日线 + MA20")
plt.xlabel("date")
plt.ylabel("price")
plt.legend()
plt.grid(alpha=0.3)
plt.show()
```

### plotly：交互式图表，适合探索

```python
import plotly.graph_objects as go

fig = go.Figure()
fig.add_trace(go.Scatter(
    x=df.index, y=df["close"],
    mode="lines", name="close"
))
fig.add_trace(go.Scatter(
    x=df.index, y=df["ma_20"],
    mode="lines", name="ma_20", line=dict(dash="dash")
))
fig.update_layout(title="沪深300 日线", xaxis_title="date", yaxis_title="price")
fig.show()
```

### 副：K 线图（plotly）

```python
fig = go.Figure(data=[go.Candlestick(
    x=df.index,
    open=df["open"], high=df["high"],
    low=df["low"], close=df["close"],
    name="K 线"
)])
fig.show()
```

---

## 四、Jupyter Notebook —— 探索环境

量化研究的标准工作流是 **Jupyter 探索 → .py 脚本固化**。

### 常用快捷键

| 快捷键 | 作用 |
|--------|------|
| `Shift+Enter` | 运行当前单元格并跳到下一格 |
| `Ctrl+Enter` | 运行当前单元格 |
| `A` / `B` | 在上方/下方插入新单元格 |
| `D D` | 删除当前单元格 |
| `M` | 切换到 Markdown 模式 |
| `Tab` | 自动补全 |
| `Shift+Tab` | 查看函数文档 |

### 量化 Notebook 组织建议

```
Cell 1:  导入库、设置参数
Cell 2:  数据获取
Cell 3:  数据清洗与预处理
Cell 4:  因子/信号构建
Cell 5:  回测逻辑
Cell 6:  结果可视化
Cell 7:  总结与待办
```

> **原则**：每个 Cell 只做一件事，输出附带一句话结论。回测结果在 Notebook 中验证后，将稳定部分提取为 `.py` 模块。

> 想进一步了解 Jupyter 的设计思路与使用技巧？详见 [Jupyter 详解](../glossary/basics/jupyter.md)。

---

## 五、实战组合：数据获取 + 均线计算 + 绘图

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# 1. 读取数据
df = pd.read_csv("data/000300.csv", parse_dates=["date"], index_col="date")

# 2. 计算均线
df["ma_5"]  = df["close"].rolling(5).mean()
df["ma_20"] = df["close"].rolling(20).mean()

# 3. 生成信号：金叉买入，死叉卖出
df["signal"] = 0
df.loc[df["ma_5"] > df["ma_20"], "signal"] = 1
df["position"] = df["signal"].diff()        # 1 买入，-1 卖出
df["daily_ret"] = df["close"].pct_change()
df["strategy_ret"] = df["position"].shift(1) * df["daily_ret"]

# 4. 累计收益对比
df["cum_bench"]  = (1 + df["daily_ret"]).cumprod()
df["cum_strat"]  = (1 + df["strategy_ret"]).cumprod()

# 5. 绘图
plt.figure(figsize=(12, 6))
plt.plot(df.index, df["cum_bench"], label="基准", linewidth=1)
plt.plot(df.index, df["cum_strat"], label="策略", linewidth=1)
plt.legend()
plt.title("双均线策略 vs 基准")
plt.grid(alpha=0.3)
plt.show()
```

> 此段代码覆盖了本节大部分核心操作。在 Notebook 中逐行运行一次，比读十遍文档更有用。

---

> **下一步**：打开 Jupyter Notebook，拉取一只股票的数据重复上面的"实战组合"，体会从数据到信号的完整链路。然后进入 `docs/data/` 深入数据与因子部分。
