# 因子挖掘方法论

> **前置要求**：已阅读 [factor-intro.md](factor-intro.md) 和 [factor-library.md](factor-library.md)，理解因子的基本概念和经典公式。
> **内容概览**：因子灵感的来源、假设驱动的挖掘流程、因子工程的常用技巧、数据挖掘偏差的规避、一个完整挖掘案例。
> **更新日期**：2026-07-02

---

因子库里的经典因子是"鱼"，真正的能力是学会"钓鱼"。这篇文档讲的是钓鱼的方法。

## 一、因子灵感从哪里来

### 1. 学术论文

量化因子的主要来源。重点关注以下期刊和数据库：

| 来源 | 说明 |
|------|------|
| Journal of Finance / Journal of Financial Economics | 顶级期刊，因子来源最权威 |
| SSRN（论文预印本） | 新因子发布最快，但未经过同行评审 |
| AQR Capital 研究论文 | 业界最活跃的量化研究机构之一 |
| Alpha Architect / Quantpedia | 将学术因子翻译为可交易策略 |

**读论文的技巧**：不要从头读到尾。直接跳到方法论章节看因子公式，然后用 Python 在 A 股上复制。如果 IC 方向和原文一致，再仔细读全文；如果不一致，先检查数据处理是否有误。

### 2. 市场直觉

从交易经验中抽象出可量化的规则：

```
直觉："财报超预期的股票会涨"
  → 量化：定义"超预期" = (实际 EPS - 预期 EPS) / 预期 EPS
  → 因子：标准化意外盈利（Standardized Unexpected Earnings, SUE）

直觉："跌多了总会反弹"
  → 量化：过去 N 日跌幅排序
  → 因子：短期反转因子（1 个月反转）
```

### 3. 数据探索——"先看分布，再看相关"

```python
import pandas as pd
import numpy as np

# 数据探索的四个标准步骤
# 1. 看分布：因子值的直方图/密度图
# 2. 看截面：每期分组后各组未来收益的单调性
# 3. 看 IC：IC 时间序列的均值和稳定性
# 4. 看相关性：新因子与已有因子的相关性矩阵
```

### 4. 另类数据（Alternative Data）

| 数据类型 | 具体来源 | 潜在因子方向 |
|---------|---------|------------|
| 卫星图像 | 停车场车辆密度、油罐储量 | 零售/能源公司营收预测 |
| 社交媒体 | 微博/推特情绪、搜索量 | 舆情因子、关注度因子 |
| 供应链 | 海关进出口、航运数据 | 产业链景气度 |
| 另类金融 | 分析师一致预期、融资融券 | 预期修正、杠杆情绪 |

---

## 二、假设驱动的挖掘流程

随意组合数据直到撞到一个高 IC 的信号——这是数据挖掘，大概率过拟合。正确的方式是**先有假设，后有验证**。

### 标准流程

```
① 提出假设 → ② 定义因子公式 → ③ 计算因子值 → ④ 单因子检验
                                            ↓
⑧ 样本外验证 ← ⑦ 因子组合 ← ⑥ 稳健性检验 ← ⑤ 多因子检验（相关性检查）
      ↓
⑨ 实盘模拟
```

### 每一步的检查清单

```python
# ③ 计算因子值：确保数据质量
# - 缺失值占比是否 < 20%？
# - 因子值分布是否合理（无极端异常值）？
# - 因子值的时间序列是否有结构性断点？

# ④ 单因子检验
# - Rank IC 均值 > 0.03？
# - IC > 0 的月份占比 > 60%？
# - 分层回测：多头组合收益 > 空头组合收益？（单调性）
# - 因子在市值子样本中的效果是否一致？（稳健性）

# ⑤ 多因子检验
# - 新因子与已有因子的相关性 < 0.5？
# - Fama-MacBeth 回归：控制已有因子后新因子仍有显著溢价？
```

### 完整挖掘脚本框架

```python
def factor_discovery_workflow(data, factor_formula, factor_name):
    """
    因子挖掘标准工作流
    返回：该因子是否值得进入因子库的判断
    """
    results = {}
    
    # Step 1: 计算因子值
    data[factor_name] = factor_formula(data)
    
    # Step 2: 数据质量检查
    coverage = 1 - data[factor_name].isna().mean()
    results["数据覆盖率"] = coverage
    if coverage < 0.8:
        print(f"⚠ {factor_name}: 覆盖率仅 {coverage:.1%}，建议扩大股票池或调整公式")
    
    # Step 3: 单因子 IC 检验
    ic_series = []
    for date, group in data.groupby("date"):
        ic = group[factor_name].rank().corr(group["future_ret_1m"].rank())
        ic_series.append(ic)
    
    ic_mean = np.nanmean(ic_series)
    ic_std = np.nanstd(ic_series)
    ir = ic_mean / ic_std if ic_std > 0 else 0
    
    results["IC均值"] = ic_mean
    results["IR"] = ir
    results["IC>0占比"] = (pd.Series(ic_series) > 0).mean()
    
    # Step 4: 分层回测单调性
    data["factor_quantile"] = data.groupby("date")[factor_name].transform(
        lambda x: pd.qcut(x, 5, labels=False, duplicates="drop")
    )
    quantile_rets = data.groupby(["date", "factor_quantile"])["future_ret_1m"].mean().unstack()
    monotonic = quantile_rets.iloc[:, -1].mean() > quantile_rets.iloc[:, 0].mean()
    results["单调性"] = "✓" if monotonic else "✗"
    
    # Step 5: 与已有因子的相关性
    existing_factors = ["momentum_1m", "volatility_1m", "turnover_1m"]
    correlations = {}
    for ef in existing_factors:
        if ef in data.columns:
            corr = data.groupby("date").apply(
                lambda g: g[factor_name].corr(g[ef])
            ).mean()
            correlations[ef] = corr
    results["因子相关性"] = correlations
    
    # 最终判断
    passed = (
        coverage >= 0.8 and
        ic_mean > 0.02 and
        ir > 0.3 and
        monotonic and
        all(abs(c) < 0.5 for c in correlations.values())
    )
    results["通过筛选"] = "✓ 进入因子库" if passed else "✗ 需要优化或放弃"
    
    return pd.Series(results)
```

---

## 三、因子工程的常用技巧

### 1. 时序平滑

原始因子值噪声大，平滑是第一步。

```python
# 方法一：移动平均
factor_smooth = raw_factor.rolling(5).mean()

# 方法二：EMA（指数移动平均）——近期权重更高
factor_ema = raw_factor.ewm(span=5).mean()

# 方法三：Hull 移动平均——减少滞后
# WMA(n) = 加权移动平均
# Hull = WMA(2*WMA(n/2) - WMA(n), sqrt(n))

# 什么时候平滑？
# - 换手率约束时用平滑降低交易频率
# - 高频因子（分钟级）比日频更需要平滑
```

### 2. 截面标准化

```python
# 每期将因子值转为 Z-score：减去均值，除以标准差
data["factor_z"] = data.groupby("date")["raw_factor"].transform(
    lambda x: (x - x.mean()) / x.std()
)

# 或转为分位数：映射到 [0, 1] 区间
data["factor_rank"] = data.groupby("date")["raw_factor"].transform(
    lambda x: x.rank(pct=True)
)
```

### 3. 窗口选择

```python
# 同一个公式，不同窗口是不同的因子
# 例：过去 N 日收益率
momentum_1m = close.pct_change(21)   # 动量因子
reversal_1w = close.pct_change(5)    # 反转因子（短期负相关）

# 窗口选择的经验法则
# - 日频因子：5, 10, 20, 60, 120 日是常用窗口
# - 月频因子：1, 3, 6, 12 个月
# - 窗口不宜过密（如 1-20 全部试一遍）→ 数据挖掘嫌疑
```

### 4. 因子变形

```python
# 同一基础数据的多种表达
raw_factor = close.pct_change(20)  # 原始：20 日动量

# 变形一：动量加速度（动量的一阶差分）
accel = raw_factor.diff()

# 变形二：动量的波动率（因子稳定性）
vol_of_momentum = raw_factor.rolling(60).std()

# 变形三：极端值截断
capped = raw_factor.clip(lower=raw_factor.quantile(0.01),
                          upper=raw_factor.quantile(0.99))
```

---

## 四、如何避免数据挖掘

因子挖掘最大的敌人是自己——你总会发现一些"看起来很好"的信号。

### 多重检验校正

如果你同时测试 100 个因子，即使全是噪声，也有约 5 个会出现 |t| > 2 的统计显著。这时需要校正显著性阈值。

```python
from statsmodels.stats.multitest import multipletests

# 100 个因子各自的 p 值
p_values = [0.01, 0.03, 0.05, 0.001, ...]  # 100 个

# Bonferroni 校正：阈值 = 0.05 / 100 = 0.0005
# 只有 p < 0.0005 才算显著
reject_bonferroni, pvals_corrected, _, _ = multipletests(
    p_values, alpha=0.05, method="bonferroni"
)

# BH 校正（更宽松）
reject_bh, pvals_corrected, _, _ = multipletests(
    p_values, alpha=0.05, method="fdr_bh"
)
```

### 经济逻辑约束

一个因子除了统计上显著，还必须有**合理的经济解释**。没有经济逻辑的因子几乎一定会衰减。

```
✓ 好理由："低波动率股票长期跑赢高波动率，因为散户偏好彩票型股票推高了价格"
✗ 坏理由："倒数第 7 天的收益率和未来 13 天收益的秩相关系数为 0.04"
```

---

## 五、完整案例：挖掘一个"机构关注度"因子

### 假设

机构持股比例高的股票，公司治理更好、信息不对称更低，长期收益应该更高。

### 实现

```python
import akshare as ak
import pandas as pd
import numpy as np

# 1. 获取机构持股数据（季度）
# 用 AKShare 获取基金重仓股数据作为代理变量
fund_holdings = ak.fund_portfolio_hold_detail_em(date="2024-03-31")

# 2. 构建因子：机构持股家数（标准化后）
# 实际中需要多季度数据，这里简化
def construct_institutional_factor(stock_list, date_range):
    """
    机构关注度因子：被多少只基金持有
    每月更新（用最近一期季报数据）
    """
    # 数据获取逻辑...
    # 因子值 = log(1 + 持股基金数)
    # 然后用行业中性化处理
    pass

# 3. 验证流程
# ic_series = rolling_ic(factor, future_ret_1m)
# 预期 IC 均值 > 0.03，IR > 0.5
```

### 验证结果（示例）

| 指标 | 原始因子 | 行业中性化后 |
|------|---------|------------|
| IC 均值 | 0.035 | 0.028 |
| IR | 0.62 | 0.55 |
| IC>0 占比 | 72% | 68% |
| 单调性 | ✓ | ✓ |

行业中性化后 IC 下降是正常的——说明机构偏好某些行业。只要去除行业因素后因子仍然有效，就说明它提供了行业之外的增量信息。

---

## 六、因子发现检查清单

在将一个新因子加入因子库之前，逐项检查：

- [ ] 有明确的经济逻辑支撑
- [ ] 数据覆盖率 > 80%
- [ ] Rank IC 均值 > 0.03，IR > 0.5
- [ ] IC 在时间上均匀分布，不存在"仅某几年有效"的情况
- [ ] 分层回测单调性成立
- [ ] 与已有因子的相关性 < 0.5
- [ ] 在市值中性化后仍有显著性
- [ ] 在样本外（最近 1-2 年）的 IC 不衰减
- [ ] 有合理的换手率（因子值不会每天剧烈跳变）

---

> **动手练习**：从自己的直觉中提炼一个假设（如"最近有龙虎榜买入的股票短期会涨"），按本篇流程走通：假设定义 → 数据获取 → 计算因子 → IC 检验 → 稳健性检验。重点体会"先有假设，后有验证"和"随便试出来的因子"之间的区别。
