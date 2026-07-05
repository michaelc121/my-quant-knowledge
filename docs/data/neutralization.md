# 行业中性化与市值中性化

> **前置要求**：已阅读 factor-intro.md、factor-library.md，理解因子定义与 IC 评估。
> **内容概览**：为什么需要中性化、行业与市值中性化的数学原理、Python 实现、结果验证。
> **更新日期**：2026-07-02

---

## 一、为什么需要中性化

假设你算了一个"市盈率倒数"（EP）作为选股因子，然后发现它 IC 很高。但问题是：

- 银行股天生 PE 低，消费股天生 PE 高
- 你选出来的全是银行股——这个因子本质上在**赌行业**，不是找到真正的 alpha

**中性化的目的**：去除已知的系统性暴露（行业、市值），让因子反映"纯粹的公司质量"。

```text
原始因子 = 行业暴露 + 市值暴露 + 真正的 alpha
中性化   = 把前两项减掉，只留 alpha
```

---

## 二、行业中性化

### 原理

对每个交易日，用因子值对行业哑变量做线性回归，取残差。

```text
factor_value = β₀ + β₁·D_银行 + β₂·D_消费 + ... + βₙ·D_n + ε
                                          ↑                    ↑
                                      行业暴露              残差 = 纯因子
```

### 实现

```python
import pandas as pd
import numpy as np
import statsmodels.api as sm

def industry_neutralize(
    factor: pd.DataFrame,
    industry: pd.DataFrame
) -> pd.DataFrame:
    """
    行业中性化
    factor: index=日期, columns=股票代码, values=因子值
    industry: index=日期, columns=股票代码, values=行业分类（如 '银行'、'医药'）
    返回：残差（去除行业影响后的因子值）
    """
    # 行业 → 哑变量矩阵
    industry_dummies = pd.get_dummies(industry, prefix="ind")

    residuals = pd.DataFrame(index=factor.index, columns=factor.columns, dtype=float)

    for date in factor.index:
        # 当期有效的股票
        valid_stocks = factor.loc[date].dropna().index
        if len(valid_stocks) < 10:
            continue

        y = factor.loc[date, valid_stocks].values
        X = industry_dummies.loc[date, valid_stocks].values
        X = sm.add_constant(X)      # 加截距

        try:
            model = sm.OLS(y, X).fit()
            residuals.loc[date, valid_stocks] = model.resid
        except Exception:
            continue

    return residuals
```

### 验证：中性化后的行业暴露

```python
def check_industry_exposure(
    factor: pd.DataFrame,
    industry: pd.DataFrame,
    date: str = None
) -> pd.Series:
    """
    检查某一天的因子行业暴露
    中性化后，每个行业的因子均值应接近 0
    """
    if date is None:
        date = factor.index[-1]

    f = factor.loc[date].dropna()
    ind = industry.loc[date, f.index]

    exposure = f.groupby(ind).mean()
    return exposure.sort_values()
```

---

## 三、市值中性化

### 原理

log 市值对收益有显著解释力（小盘股长期跑赢大盘股）。如果不做市值中性化，因子可能只是"小盘股效应"的代理。

```python
def size_neutralize(
    factor: pd.DataFrame,
    market_cap: pd.DataFrame,
    industry: pd.DataFrame = None
) -> pd.DataFrame:
    """
    市值中性化（可选：叠加行业中性化）
    对每个交易日：factor ~ log(market_cap) + 行业哑变量 → 取残差
    """
    log_mcap = np.log(market_cap + 1)
    residuals = pd.DataFrame(index=factor.index, columns=factor.columns, dtype=float)

    for date in factor.index:
        valid = factor.loc[date].dropna().index
        # 同时有市值和因子值的股票
        valid = valid.intersection(log_mcap.loc[date].dropna().index)
        if len(valid) < 10:
            continue

        # Y = 因子值
        y = factor.loc[date, valid].values

        # X = log 市值（+ 可选行业哑变量）
        X_df = pd.DataFrame({"log_mcap": log_mcap.loc[date, valid].values})

        if industry is not None:
            ind_valid = industry.loc[date, valid]
            ind_dummies = pd.get_dummies(ind_valid, prefix="ind", drop_first=True)
            X_df = pd.concat([X_df, ind_dummies], axis=1)

        X = sm.add_constant(X_df.values)

        try:
            model = sm.OLS(y, X).fit()
            residuals.loc[date, valid] = model.resid
        except Exception:
            continue

    return residuals
```

---

## 四、完整中性化流程

同时做行业 + 市值中性化：

```python
def full_neutralize(
    factor: pd.DataFrame,
    industry: pd.DataFrame,
    market_cap: pd.DataFrame
) -> pd.DataFrame:
    """
    完整中性化：factor ~ industry_dummies + log_mcap → 取残差
    """
    log_mcap = np.log(market_cap + 1)
    residuals = pd.DataFrame(index=factor.index, columns=factor.columns, dtype=float)

    for date in factor.index:
        valid_stocks = factor.loc[date].dropna().index
        valid_stocks = valid_stocks.intersection(
            log_mcap.loc[date].dropna().index
        )
        valid_stocks = valid_stocks.intersection(
            industry.loc[date].dropna().index
        )

        if len(valid_stocks) < 10:
            continue

        y = factor.loc[date, valid_stocks].values

        # 行业哑变量 + log 市值
        ind_dummies = pd.get_dummies(
            industry.loc[date, valid_stocks],
            prefix="ind", drop_first=True
        )
        X_df = pd.concat([
            pd.DataFrame({"log_mcap": log_mcap.loc[date, valid_stocks].values}),
            ind_dummies
        ], axis=1)
        X = sm.add_constant(X_df.values)

        try:
            model = sm.OLS(y, X).fit()
            residuals.loc[date, valid_stocks] = model.resid
        except Exception:
            continue

    return residuals
```

---

## 五、中性化效果验证

### 1. 行业暴露对比

```python
def compare_neutralization_effect(
    raw_factor: pd.Series,
    neutralized_factor: pd.Series,
    industry: pd.Series,
    name: str = ""
):
    """
    对比中性化前后，每个行业的因子均值
    中性化后各行业均值应接近 0
    """
    raw_exp = raw_factor.groupby(industry).mean()
    neu_exp = neutralized_factor.groupby(industry).mean()

    print(f"\n{name} 行业暴露:")
    print(f"  中性化前标准差: {raw_exp.std():.4f}")
    print(f"  中性化后标准差: {neu_exp.std():.4f}")
    print(f"  行业暴露降低: {(1 - neu_exp.std()/raw_exp.std())*100:.1f}%")
```

### 2. 市值相关性对比

```python
def check_size_correlation(
    factor: pd.Series,
    market_cap: pd.Series
):
    """因子与市值的相关系数——中性化后应接近 0"""
    from scipy.stats import spearmanr
    valid = factor.notna() & market_cap.notna()
    corr, pval = spearmanr(
        factor[valid].rank(),
        market_cap[valid].rank()
    )
    print(f"  因子-市值 Spearman 相关系数: {corr:.4f} (p={pval:.4f})")
    return corr
```

---

## 六、注意事项

1. **中性化不是免费的** — 每做一次回归，等价于削掉了一部分因子信息。如果 IC 从中性化前的 0.08 跌到 0.02，说明这个因子的"alpha"大部分来自行业暴露，本身价值有限
2. **行业分类要固定** — 申万一级（28 个行业）或中信一级（30 个行业）是常用标准，不要在回测中混用
3. **市值用对数** — `log(market_cap)` 比 `market_cap` 更接近正态分布，回归效果更好
4. **drop_first=True** — 行业哑变量中去掉一个（避免多重共线性），不影响残差值
5. **中性化在因子计算后、因子评估前完成** — 不要在中性化后的因子值上再算 IC，应该用中性化后的值从头评估

---

> **动手练习**：拿 factor-library.md 中的动量因子（MOM_20），取 500 只股票最近 12 个月的数据，分别做无中性化、行业中性化、行业+市值中性化三组，对比三组因子的 IC 和 IR。观察：哪个因子 IC 最高？哪个最稳定？
