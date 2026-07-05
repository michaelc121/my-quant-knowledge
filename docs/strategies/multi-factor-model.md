# 多因子选股模型实战

> **前置要求**：已阅读 factor-intro.md、factor-library.md、neutralization.md，理解单因子评估方法。
> **内容概览**：Fama-French 三因子、因子去冗余、合成方法、选股回测、组合构建。
> **更新日期**：2026-07-02

---

## 一、从单因子到多因子

单因子的问题：依赖一个维度做决策，容易踩坑。

```text
动量因子：这周有效，下周失效（衰减快）
价值因子：低 PE 股可能是价值陷阱（基本面恶化）
质量因子：高 ROE 公司可能已经涨了很多（偏成长）

多因子的思路：
  多个因子各自覆盖不同的信息维度，互相补充
  即使某个因子失效了，其他因子还能撑住
```

---

## 二、因子去冗余：因子相关性检查

合成多因子前的第一步——剔除高度相关的因子。

```python
import seaborn as sns

def factor_correlation_matrix(
    factor_dict: dict[str, pd.Series]
) -> pd.DataFrame:
    """
    横截面因子相关性矩阵
    两个因子相关系数 > 0.7 → 只保留 IR 更高的那个
    """
    # 选择一个交易日作为截面
    latest_date = list(factor_dict.values())[0].index[-1]
    cross_section = pd.DataFrame({
        name: f.loc[latest_date]
        for name, f in factor_dict.items()
    }).dropna()

    corr = cross_section.corr(method="spearman")
    return corr


def remove_redundant_factors(
    factor_dict: dict[str, pd.DataFrame],
    ir_dict: dict[str, float],
    threshold: float = 0.7
) -> list[str]:
    """
    剔除高相关低 IR 的因子
    """
    corr = factor_correlation_matrix(factor_dict)
    keep = list(factor_dict.keys())
    removed = []

    for i, f1 in enumerate(keep):
        for j in range(i + 1, len(keep)):
            f2 = keep[j]
            if abs(corr.loc[f1, f2]) > threshold:
                # 两个相关度高 → 删掉 IR 更低的
                loser = f1 if ir_dict[f1] < ir_dict[f2] else f2
                if loser in keep:
                    keep.remove(loser)
                    removed.append(loser)

    print(f"原始: {len(factor_dict)} 个因子 → 保留: {len(keep)} 个")
    print(f"剔除: {removed}")
    return keep
```

---

## 三、因子合成方法

### 方法 1：等权合成（最简单）

```python
def equal_weight_composite(
    factor_dict: dict[str, pd.DataFrame]
) -> pd.DataFrame:
    """等权合成：每个因子 rank 标准化 → 取平均"""
    ranks = pd.DataFrame(index=factor_dict[list(factor_dict.keys())[0]].index)

    for name, f in factor_dict.items():
        # 截面 rank 归一化到 [0, 1]
        ranks[name] = f.rank(axis=1, pct=True)

    composite = ranks.mean(axis=1)
    return composite
```

### 方法 2：IC 加权合成（动态）

```python
def ic_weighted_composite(
    factor_dict: dict[str, pd.DataFrame],
    return_df: pd.DataFrame,
    lookback: int = 60
) -> pd.DataFrame:
    """
    IC 加权合成：滚动 IC 作为权重
    过去 60 天 IC 越高的因子，当前权重越大
    """
    from scipy.stats import spearmanr

    ic_history = {}
    for name, f in factor_dict.items():
        ic_series = []
        for t in range(lookback, len(f)):
            valid = f.iloc[t].notna() & return_df.iloc[t + 1].notna()
            ic = spearmanr(f.iloc[t][valid], return_df.iloc[t + 1][valid])[0]
            ic_series.append(ic)
        ic_history[name] = pd.Series(
            ic_series, index=f.index[lookback:]
        ).rolling(lookback).mean()

    # 权重 = max(IC, 0)，归一化
    weights = pd.DataFrame(ic_history).clip(lower=0)
    weights = weights.div(weights.sum(axis=1), axis=0)

    # 加权合成
    ranks = {name: f.rank(axis=1, pct=True) for name, f in factor_dict.items()}
    composite = pd.Series(index=weights.index, dtype=float)
    for date in weights.index:
        vals = [ranks[name].loc[date] * weights.loc[date, name] for name in factor_dict.keys()]
        composite[date] = sum(vals)

    return composite
```

### 方法 3：PCA 合成（降维）

```python
from sklearn.decomposition import PCA

def pca_composite(factor_dict: dict[str, pd.DataFrame]) -> pd.DataFrame:
    """
    PCA 降维到 1 维作为综合得分
    """
    pca = PCA(n_components=1)
    composite = pd.DataFrame(index=factor_dict[list(factor_dict.keys())[0]].index)

    for date in composite.index:
        X = pd.DataFrame({name: f.loc[date] for name, f in factor_dict.items()}).dropna()
        if len(X) > 10:
            scores = pca.fit_transform(X)
            composite.loc[date, X.index] = scores.flatten()

    return composite
```

---

## 四、完整多因子选股流程

```python
def multi_factor_selection(
    factor_dict: dict[str, pd.DataFrame],
    return_df: pd.DataFrame,
    industry: pd.DataFrame,
    market_cap: pd.DataFrame,
    top_n: int = 50,
    rebalance_freq: str = "M"
) -> pd.DataFrame:
    """
    完整多因子选股回测

    流程：
    1. 每个因子做行业+市值中性化
    2. 检查因子相关性，去冗余
    3. IC 加权合成综合得分
    4. 每月选 top_n 只股票
    5. 等权持有，计算组合收益
    """
    from datetime import timedelta

    # 1. 中性化（详见 [行业中性化与市值中性化](../data/neutralization.md)）
    neutralized = {}
    for name, f in factor_dict.items():
        from docs.data.neutralization import full_neutralize
        neutralized[name] = full_neutralize(f, industry, market_cap)

    # 2. 去冗余 + 合成
    composite = ic_weighted_composite(neutralized, return_df)

    # 3. 选股：排名前 top_n
    def select_top(df_row, n):
        return df_row.nlargest(n).index.tolist()

    # 4. 每月调仓
    positions = {}
    for date in composite.index:
        if positions:
            last_date = max(positions.keys())
            if rebalance_freq == "M" and date.month == last_date.month:
                continue

        selected = select_top(composite.loc[date], top_n)
        positions[date] = selected

    # 5. 计算组合收益
    portfolio_returns = pd.Series(index=composite.index, dtype=float)
    current_holdings = []

    for i, date in enumerate(composite.index):
        if date in positions:
            current_holdings = positions[date]

        # 当天持仓的等权收益
        valid_stocks = [s for s in current_holdings if s in return_df.columns]
        if valid_stocks:
            portfolio_returns[date] = return_df.loc[date, valid_stocks].mean()

    return portfolio_returns
```

---

## 五、Fama-French 三因子模型（经典）

Fama-French 三因子是学术界的基准模型，理解它等于理解了多因子体系的底层逻辑。

```text
E(R) - Rf = β_mkt × (Rm - Rf) + β_smb × SMB + β_hml × HML

SMB（Small Minus Big）: 小盘股收益 - 大盘股收益
HML（High Minus Low）: 高 PB 倒数（价值股）收益 - 低 PB 倒数（成长股）收益
```

### Python 构建 SMB 和 HML 因子

```python
def build_ff3_factors(market_cap, pb_ratio, returns):
    """构建 A 股 Fama-French 三因子"""
    # 市值分大小盘（中位数切分）
    size_median = market_cap.median(axis=1)
    is_big = market_cap.gt(size_median, axis=0)
    is_small = ~is_big

    # PB 倒数（EP）分高/低
    ep = 1 / pb_ratio
    ep_70 = ep.quantile(0.7, axis=1)
    ep_30 = ep.quantile(0.3, axis=1)
    is_high_bp = ep.gt(ep_70, axis=0)
    is_low_bp = ep.lt(ep_30, axis=0)

    # SMB：小盘 - 大盘
    smb = returns[is_small].mean(axis=1) - returns[is_big].mean(axis=1)

    # HML：高 EP - 低 EP
    hml = returns[is_high_bp].mean(axis=1) - returns[is_low_bp].mean(axis=1)

    return smb, hml
```

---

## 六、组合优化（简易版）

选出了 top_n 只股票后，怎么分配每只的仓位？

```python
def equal_weight(rets: pd.Series) -> pd.Series:
    """等权：每只股票 1/n"""
    return pd.Series(1 / len(rets), index=rets.index)


def composite_weight(scores: pd.Series, top_n: int = 20) -> pd.Series:
    """得分加权：得分越高权重越大"""
    top_scores = scores.nlargest(top_n)
    return top_scores / top_scores.sum()
```

---

## 七、常见陷阱

1. **因子相关性** — 用了动量(MOM_20) 和动量(MOM_60)，相关性 > 0.8，等于在加倍一个因子
2. **数据时间不一致** — 财务数据更新比股价晚 2–3 个月，用错了时间帧会产生前视偏差
3. **调仓频率** — 每天调仓成本吃掉收益，月频调仓是适合新手的起点
4. **金融股陷阱** — 银行、保险的 PB/PE 天生低，不做行业中性化一定会被它们占满持仓

---

> **动手练习**：从 factor-library.md 中选 5 个不同类别的因子（各一个），按本篇流程走通：中性化 → 去冗余 → IC 加权合成 → 选股 → 等权持有 → 回测。对比单因子（最好的那个）和多因子合成的收益曲线。
