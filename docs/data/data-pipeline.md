# 数据 Pipeline

> **前置要求**：已阅读 bar-conventions.md、factor-intro.md、factor-library.md。
> **内容概览**：从数据获取到入库的工程化流程，增量更新、清洗、存储、自动化。
> **更新日期**：2026-07-02

---

## 一、为什么需要 Pipeline

前期实验中你可能是手动拉数据、在 Notebook 里清洗、算因子。但当你需要：

- 每天自动更新行情
- 维护 3000+ 只股票的历史数据
- 确保每次回测使用同一份干净数据
- 团队协作时不出现"你用的数据和我用的不一样"

这时候就需要把数据流程**工程化**：写一次，跑无数次。

---

## 二、整体架构

```
数据源（AKShare / Tushare）
    │
    ▼
┌──────────────────┐
│  获取层 (fetch)   │  ← 增量更新，只拉新数据
│  重试、限速、日志 │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  原始存储层 (raw) │  ← CSV / Parquet，只写不删
│  按日期分区       │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  清洗层 (clean)   │  ← 复权、去除停牌、OHLC 校验
│  生成标准格式日线 │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  因子计算层       │  ← 从清洗后的数据算因子
│  (factor)         │
└──────────────────┘
```

**原则**：每一层的输出只依赖上一层的输出，同一层可以随时重跑而不影响下层。

---

## 三、获取层

### 增量更新

只拉缺失的日期，不重复拉已有数据：

```python
import pandas as pd
from pathlib import Path

def fetch_daily_incremental(
    symbol: str,
    data_dir: Path,
    start_date: str = "2020-01-01",
    end_date: str = None
) -> pd.DataFrame:
    """
    增量拉取日线数据
    - 检查本地已有数据的最后日期
    - 只拉缺失部分
    """
    if end_date is None:
        end_date = pd.Timestamp.today().strftime("%Y%m%d")

    file_path = data_dir / f"{symbol}.parquet"

    if file_path.exists():
        existing = pd.read_parquet(file_path)
        last_date = existing.index.max()
        fetch_start = (last_date + pd.Timedelta(days=1)).strftime("%Y%m%d")
        print(f"{symbol}: 已有数据至 {last_date.date()}, 从 {fetch_start} 起增量")
    else:
        fetch_start = start_date
        existing = pd.DataFrame()

    if fetch_start >= end_date:
        return existing

    # 调用 AKShare 拉取
    import akshare as ak
    new_data = ak.stock_zh_a_hist(
        symbol=symbol,
        period="daily",
        start_date=fetch_start,
        end_date=end_date,
        adjust=""
    )
    new_data = new_data.rename(columns={
        "日期": "date", "开盘": "open", "最高": "high",
        "最低": "low", "收盘": "close", "成交量": "volume",
        "成交额": "amount", "振幅": "amplitude",
        "涨跌幅": "pct_change", "涨跌额": "change",
        "换手率": "turnover"
    })
    new_data["date"] = pd.to_datetime(new_data["date"])
    new_data = new_data.set_index("date").sort_index()

    # 合并
    combined = pd.concat([existing, new_data])
    combined = combined[~combined.index.duplicated(keep="last")]
    combined.to_parquet(file_path)

    return combined
```

### 批量更新

```python
def batch_update(stock_list: list[str], data_dir: Path):
    """批量更新所有股票"""
    for symbol in stock_list:
        try:
            fetch_daily_incremental(symbol, data_dir)
            print(f"  ✓ {symbol}")
        except Exception as e:
            print(f"  ✗ {symbol}: {e}")
```

---

## 四、存储层

### 格式选择

| 格式 | 速度 | 磁盘 | 适用 |
|------|------|------|------|
| CSV | 慢 | 大 | 少量数据、人类可读 |
| **Parquet** | **快** | **小（压缩）** | **量化首选** |
| HDF5 | 中 | 中 | 单文件多表 |
| DuckDB | 快 | 中 | SQL 查询习惯 |

### 推荐目录结构

```
data/
├── raw/                     # 原始数据，只追加不修改
│   ├── stock_daily/
│   │   ├── 000001.parquet
│   │   ├── 000002.parquet
│   │   └── ...
│   ├── stock_minute/        # 分钟线
│   ├── index_daily/         # 指数数据
│   └── financial/           # 财务数据
├── clean/                   # 清洗后的标准数据
│   ├── stock_daily_adj.parquet     # 已复权的日线（所有股票合并）
│   └── stock_daily_features.parquet # 加入移动平均等预计算
├── factors/                 # 因子值
│   ├── mom_20.parquet
│   ├── vol_60.parquet
│   └── ...
└── metadata/
    └── update_log.csv       # 每日更新记录
```

### Parquet 的优势

```python
# Parquet 列式存储：只读需要的列
df = pd.read_parquet("data.parquet", columns=["close", "volume"])

# 压缩率高（相比 CSV 缩小 70-80%）
# 自带 schema，不会类型推断错误
# 支持分区存储
```

---

## 五、清洗层

标准化清洗流程，确保所有后续分析使用同一份干净数据：

```python
def standard_clean(
    df: pd.DataFrame, name: str = "", adj_factor: pd.Series = None
) -> pd.DataFrame:
    """
    标准数据清洗流程
    输入：原始日线数据
    输出：清洗后的标准数据
    """
    df = df.copy().sort_index()

    # 1. 基础校验
    assert df.index.is_monotonic_increasing

    # 2. 复权
    if adj_factor is not None:
        adj = adj_factor / adj_factor.iloc[-1]
        for col in ["open", "high", "low", "close"]:
            if col in df.columns:
                df[col] = df[col] * adj
        if "volume" in df.columns:
            df["volume"] = df["volume"] / adj

    # 3. 停牌标记
    df["is_suspended"] = df["volume"] == 0

    # 4. OHLC 异常标记
    ohlc_ok = (
        (df["high"] >= df["low"])
        & (df["high"] >= df["close"])
        & (df["open"] >= 0)
    )
    df["is_bad_ohlc"] = ~ohlc_ok

    # 5. 涨跌停标记
    df["is_limit_up"]   = df["close"] >= df["close"].shift(1) * 1.095
    df["is_limit_down"] = df["close"] <= df["close"].shift(1) * 0.905

    # 6. 收益率
    df["daily_return"] = df["close"].pct_change()

    return df
```

---

## 六、自动化

### 定时运行

```python
# scripts/update_data.py

def main():
    stock_list = load_stock_list("config/stock_list.csv")
    data_dir = Path("data/raw/stock_daily")

    print(f"[{pd.Timestamp.now()}] 开始更新 {len(stock_list)} 只股票")
    batch_update(stock_list, data_dir)
    print(f"[{pd.Timestamp.now()}] 更新完成")

if __name__ == "__main__":
    main()
```

配合系统定时任务：

```bash
# crontab：每个交易日 18:00 更新
# 0 18 * * 1-5 cd /path/to/project && python scripts/update_data.py >> logs/update.log 2>&1
```

### 版本记录

在每次更新后记录元数据：

```python
def log_update(symbol: str, status: str, message: str = ""):
    """记录更新日志"""
    log_file = Path("data/metadata/update_log.csv")
    entry = pd.DataFrame([{
        "timestamp": pd.Timestamp.now(),
        "symbol": symbol,
        "status": status,
        "message": message
    }])
    if log_file.exists():
        existing = pd.read_csv(log_file)
        entry = pd.concat([existing, entry])
    entry.to_csv(log_file, index=False)
```

---

## 七、完整 Pipeline 脚本模板

```python
"""
scripts/data_pipeline.py

用法：
  python scripts/data_pipeline.py --fetch     # 获取新数据
  python scripts/data_pipeline.py --clean     # 清洗全量
  python scripts/data_pipeline.py --all       # 获取 + 清洗

目录结构：
  config/stock_list.csv     # 关注的股票列表
  data/raw/                 # 原始数据
  data/clean/               # 清洗后数据
  logs/pipeline.log         # 运行日志
"""

import argparse
from pathlib import Path
import pandas as pd
from datetime import datetime


def run_pipeline(steps: list[str]):
    data_dir = Path("data")
    raw_dir = data_dir / "raw"
    clean_dir = data_dir / "clean"
    raw_dir.mkdir(parents=True, exist_ok=True)
    clean_dir.mkdir(parents=True, exist_ok=True)

    stock_list = pd.read_csv("config/stock_list.csv")["symbol"].tolist()

    if "fetch" in steps:
        print(f"[{datetime.now()}] 开始获取数据")
        for symbol in stock_list:
            try:
                fetch_daily_incremental(symbol, raw_dir / "stock_daily")
                print(f"  ✓ {symbol}")
            except Exception as e:
                print(f"  ✗ {symbol}: {e}")

    if "clean" in steps:
        print(f"[{datetime.now()}] 开始清洗数据")
        all_clean = []
        for symbol in stock_list:
            raw_path = raw_dir / "stock_daily" / f"{symbol}.parquet"
            if raw_path.exists():
                df = pd.read_parquet(raw_path)
                df_clean = standard_clean(df, name=symbol)
                df_clean["symbol"] = symbol
                all_clean.append(df_clean)
        if all_clean:
            result = pd.concat(all_clean)
            result.to_parquet(clean_dir / "stock_daily_adj.parquet")
            print(f"  清洗完成: {len(result)} 行")

    print(f"[{datetime.now()}] Pipeline 完成")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--fetch", action="store_true")
    parser.add_argument("--clean", action="store_true")
    parser.add_argument("--all", action="store_true")
    args = parser.parse_args()

    if args.all:
        run_pipeline(["fetch", "clean"])
    elif args.fetch or args.clean:
        run_pipeline([k for k in ["fetch", "clean"] if getattr(args, k)])
    else:
        parser.print_help()
```

---

> **动手练习**：创建一个 `config/stock_list.csv` 包含 10 只股票代码，用上面的 Pipeline 脚本跑一遍 `--all`。然后检查清洗后的 Parquet 文件是否正确，尝试只读取 `["date", "close", "is_limit_up"]` 三列来验证。
