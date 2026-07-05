# IC 与 IR（信息系数与信息比率）

> **来源文档**：[因子定义与评估体系](../../data/factor-intro.md)
> **前置要求**：理解因子的基本概念，知道因子值与未来收益的关系
> **更新日期**：2026-07-02

## 一句话理解

**IC** 衡量因子"每个截面预测得准不准"（单次考试得分），**IR** 衡量因子"长期预测稳不稳定"（多次考试的平均分/标准差）。

---

## IC —— 信息系数

### 定义

IC 是因子值在某个截面上与未来收益的相关系数：

```python
# t 时刻截面上：
# - 因子值向量：f_1, f_2, ..., f_n（n 只股票在 t 时刻的因子值）
# - 未来收益向量：r_1, r_2, ..., r_n（同一批股票在未来一段时间的收益）
# IC_t = corr(f, r)
```

### 两种 IC

| 类型 | 计算方法 | 优点 | 缺点 |
|------|---------|------|------|
| **Pearson IC** | `np.corrcoef(f, r)` | 标准线性相关 | 对异常值敏感 |
| **Rank IC (Spearman)** | `stats.spearmanr(f, r)` | 对异常值鲁棒 | 损失数值信息 |

**实践中首选 Rank IC**，因为股票数据中总有极端值。

```python
import numpy as np
from scipy import stats

def calc_ic(factor_values, future_returns, method="rank"):
    """
    计算单期 IC
    
    参数:
    - factor_values: (N,) 某个截面的因子值
    - future_returns: (N,) 对应的未来收益
    - method: "pearson" 或 "rank"
    """
    # 去除 NaN
    mask = ~(np.isnan(factor_values) | np.isnan(future_returns))
    f = factor_values[mask]
    r = future_returns[mask]
    
    if len(f) < 30:  # 样本太少，IC 不可靠
        return np.nan
    
    if method == "pearson":
        return np.corrcoef(f, r)[0, 1]
    else:
        return stats.spearmanr(f, r)[0]
```

### IC 的评价标准

| Rank IC 均值 | 评价 |
|-------------|------|
| > 0.05 | 强因子，有实盘价值 |
| 0.03 ~ 0.05 | 中等因子，可组合使用 |
| 0.01 ~ 0.03 | 弱因子，需仔细验证 |
| < 0.01 | 基本无效 |

---

## IR —— 信息比率

### 定义

IR 是 IC 的均值除以 IC 的标准差：

```python
IR = IC_mean / IC_std
```

**直觉**：IC 均值告诉你信号有多强，IC 标准差告诉你信号有多吵。IR 综合考虑了信号强度与稳定性。

```python
def calc_ir(ic_series):
    """
    计算 IR（信息比率）
    
    参数:
    - ic_series: (T,) 各期的 IC 值时间序列
    """
    ic_series = ic_series.dropna()
    return ic_series.mean() / ic_series.std()
```

### IR 的评价标准

| IR | 评价 |
|----|------|
| > 1.0 | 优秀（Grinhod & Kahn 基准：IR=1 可产生显著 alpha） |
| 0.5 ~ 1.0 | 良好 |
| 0.25 ~ 0.5 | 一般 |
| < 0.25 | 弱 |

---

## 完整评估函数

```python
def evaluate_factor(factor_df, factor_col, return_col, date_col):
    """
    因子评估：IC 序列 + IR
    
    参数:
    - factor_df: DataFrame，含日期、股票代码、因子值、未来收益
    """
    ic_list = []
    
    for date, group in factor_df.groupby(date_col):
        ic = calc_ic(group[factor_col].values, group[return_col].values)
        ic_list.append({"date": date, "ic": ic})
    
    ic_df = pd.DataFrame(ic_list)
    ir = ic_df["ic"].mean() / ic_df["ic"].std()
    
    print(f"IC 均值: {ic_df['ic'].mean():.4f}")
    print(f"IC 标准差: {ic_df['ic'].std():.4f}")
    print(f"IC > 0 占比: {(ic_df['ic'] > 0).mean():.2%}")
    print(f"IR: {ir:.3f}")
    
    return ic_df
```

---

## IC 的两个易混淆点

1. **IC 和收益率不是一回事**：IC=0.05 不意味着"这个因子能赚 5%"，而是"因子排名和未来收益排名之间相关系数为 0.05"。实际收益取决于你如何利用排名构建多空组合。
2. **IC 是截面概念**：每一期（每个交易日）的 IC 互不依赖。你不能用"过去 20 天的平均 IC"预测明天的 IC，但可以用 IC 的稳定性（IR）判断因子质量。

## 常见误区

1. **只看 IC 均值不看标准差**：IC 均值 0.05 但标准差 0.10（IR=0.5），说明因子时灵时不灵，实盘心理压力巨大。
2. **用全样本 IC 回看因子**：因子研发阶段看到的 IC 偏高是正常的——你已经挑过了。真正可依赖的是样本外 IC。
3. **IC 符号颠倒**：负 IC 的因子和正 IC 一样有效，只要换向即可（卖高买低→卖低买高）。

## 回到来源文档

[← 返回因子定义与评估体系](../../data/factor-intro.md)
