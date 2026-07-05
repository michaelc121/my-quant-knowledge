# 策略评价与归因诊断

> **前置要求**：已阅读 backtest-101.md，理解夏普、最大回撤等基础回测指标。
> **内容概览**：从"策略赚了多少钱"到"钱从哪来、是否可持续"的完整分析框架。
> **更新日期**：2026-07-03

---

回测跑完后你拿到一份绩效报告：年化 20%，夏普 1.2，最大回撤 15%。这些数字只回答了"历史表现如何"。这篇文档覆盖你接下来要回答的三个问题：

```
① 靠谱吗？  → 收益拆解，排除运气和暴露风险
② 哪来的？  → Brinson 归因、选股 vs 择时、因子暴露
③ 能持续吗？→ 滚动评估、策略诊断、衰减检测
```

---

## 一、收益拆解：排除"虚假"收益

一个年化 20% 的策略，真实原因可能完全不同：

```text
A: 市场普涨贡献了 18%（就是买了 beta）
B: 碰巧超配了小盘股（风格暴露）
C: 真正的超额 alpha（2%）

A 和 B 赚钱的策略，换一个市场环境就失效了。
```

### 用回归拆解收益

```python
import pandas as pd
import numpy as np
import statsmodels.api as sm


def decompose_returns(
    strategy_returns: pd.Series,
    benchmark_returns: pd.Series,
    factor_returns: pd.DataFrame = None
) -> dict:
    """
    收益分解：市场收益 + 因子暴露贡献 + alpha

    参数:
    - strategy_returns: 策略日收益率
    - benchmark_returns: 基准日收益率（如沪深 300）
    - factor_returns: 因子收益率 DataFrame（如市值、价值、动量）

    返回:
    - alpha: 扣除市场和因子后的残差，才是真正的选股能力
    - beta: 对市场的暴露
    - factor_exposures: 对各因子的暴露
    """
    aligned = pd.concat([strategy_returns, benchmark_returns], axis=1, join="inner")
    aligned.columns = ["策略", "基准"]

    if factor_returns is not None:
        aligned = aligned.join(factor_returns, how="inner")
        X = sm.add_constant(aligned[["基准"] + list(factor_returns.columns)])
    else:
        X = sm.add_constant(aligned[["基准"]])

    y = aligned["策略"]
    model = sm.OLS(y, X).fit()

    return {
        "alpha": model.params.iloc[0],            # 截距项：市场无关的超额
        "beta": model.params.iloc[1],              # 市场暴露
        "factor_exposures": model.params.iloc[2:] if factor_returns is not None else {},
        "r_squared": model.rsquared,               # 收益中有多少能被市场和因子解释
    }
```

**关注 `R²` 和 `alpha` 的显著性**：
- R² > 0.8 → 策略大部分收益来自市场暴露，alpha 很小
- R² < 0.5 → 策略和市场相关性低，可能有独立的 alpha
- alpha p-value > 0.05 → alpha 在统计上不显著，可能是运气

---

## 二、选股 vs 择时：超额收益的两个来源

超额收益（跑赢基准的部分）来自两种能力：

```text
选股能力（Stock Picking）：
  在同一个行业内选到了比行业平均好的股票

择时能力（Market Timing）：
  在市场上行时加仓、下行时减仓
```

### 实操区分

```python
def evaluate_skill(
    portfolio_returns: pd.Series,
    benchmark_returns: pd.Series,
    industry_returns: pd.DataFrame,
    window: int = 60
) -> dict:
    """
    评估选股能力和择时能力

    核心思路：
    - 选股能力 = 组合收益 - 行业平均收益（剥离行业影响后）
    - 择时能力 = 在市场上涨时是否持有更多仓位
    """
    # 选股能力：组合 vs 行业等权组合
    industry_avg = industry_returns.mean(axis=1)
    stock_picking = portfolio_returns - industry_avg

    # 择时能力：用滚动 beta 看是否在市场上涨前增加了暴露
    rolling_beta = portfolio_returns.rolling(window).cov(benchmark_returns) / benchmark_returns.rolling(window).var()
    timing_corr = rolling_beta.corr(benchmark_returns)  # beta 和市场收益的相关性

    return {
        "选股超额年化": float(stock_picking.mean() * 252),
        "择时能力得分": float(timing_corr),  # 正数 = 加仓时市场在涨
        "beta_稳定性": float(rolling_beta.std()),
    }


# 实际解读：
# 选股超额年化 > 3% → 选股能力显著
# 择时能力得分 > 0.2 → 有择时能力
# beta_稳定性 < 0.3 → beta 稳定，不是靠频繁调仓
```

---

## 三、Brinson 归因：收益来自配置还是选股

Brinson 归因是业界标准方法。它把超额收益拆成三部分：

```
超额收益 = 配置效应 + 选择效应 + 交互效应

配置效应：超配了涨得好的行业（行业轮动能力）
选择效应：在同一个行业里选的股票比同行好（选股能力）
交互效应：配置和选择交叉的部分（通常忽略不计）
```

### 单期归因

```python
def brinson_attribution(
    portfolio_weights: pd.Series,    # 组合中各行业的权重
    benchmark_weights: pd.Series,    # 基准中各行业的权重
    portfolio_returns: pd.Series,    # 组合中各行业的收益
    benchmark_returns: pd.Series,    # 基准中各行业的收益
) -> pd.DataFrame:
    """
    Brinson 单期归因

    输入：一个截面上（某个月）的行业数据
    """
    # 配置效应：超配/低配 * 基准收益
    allocation = (portfolio_weights - benchmark_weights) * benchmark_returns

    # 选择效应：行业权重基准 * 行业内超额
    selection = benchmark_weights * (portfolio_returns - benchmark_returns)

    # 交互效应（一般很小）
    interaction = (portfolio_weights - benchmark_weights) * (portfolio_returns - benchmark_returns)

    return pd.DataFrame({
        "行业": portfolio_weights.index,
        "配置效应": allocation.values,
        "选择效应": selection.values,
        "交互效应": interaction.values,
        "总超额": allocation.values + selection.values + interaction.values,
    }).set_index("行业")


# 示例
portfolio_w = pd.Series({"银行": 0.30, "医药": 0.20, "科技": 0.50})
benchmark_w = pd.Series({"银行": 0.15, "医药": 0.35, "科技": 0.50})
portfolio_r = pd.Series({"银行": 0.05, "医药": 0.10, "科技": 0.15})
benchmark_r = pd.Series({"银行": 0.03, "医药": 0.08, "科技": 0.12})

attr = brinson_attribution(portfolio_w, benchmark_w, portfolio_r, benchmark_r)
print(attr)

# 结果解读：
# - 配置效应为正 → 你超配了该行业，而该行业涨了
# - 选择效应为正 → 你在这个行业里选的股票比行业平均好
# - 哪个效应大，你的超额就主要来自哪个能力
```

### 多期累积归因

把每期的 Brinson 结果累积起来看趋势：

```python
def multi_period_brinson(
    portfolio_weights: pd.DataFrame,   # index=日期, columns=行业
    benchmark_weights: pd.DataFrame,
    portfolio_returns: pd.DataFrame,
    benchmark_returns: pd.DataFrame,
) -> pd.DataFrame:
    """
    多期 Brinson 归因：累积配置效应和选择效应
    """
    cumulative = {"配置效应": [], "选择效应": []}

    for date in portfolio_weights.index:
        attr = brinson_attribution(
            portfolio_weights.loc[date],
            benchmark_weights.loc[date],
            portfolio_returns.loc[date],
            benchmark_returns.loc[date],
        )
        cumulative["配置效应"].append(attr["配置效应"].sum())
        cumulative["选择效应"].append(attr["选择效应"].sum())

    return pd.DataFrame(cumulative, index=portfolio_weights.index).cumsum()
```

---

## 四、风险归因：风险从哪来

收益归因告诉你"钱从哪来"，风险归因告诉你"爆仓风险从哪来"。

```python
def risk_attribution(weights: np.ndarray, cov_matrix: np.ndarray) -> pd.Series:
    """
    风险归因：每个品种对组合总风险的贡献比例

    例子：银行仓位 15% 但风险贡献 40%
    → 银行波动太大，需要减仓
    """
    port_vol = np.sqrt(weights @ cov_matrix @ weights)
    marginal_risk = cov_matrix @ weights / port_vol
    risk_contrib = weights * marginal_risk
    return pd.Series(risk_contrib / risk_contrib.sum(), index=weights.index)
```

---

## 五、滚动评估：检测策略衰减

策略不是一成不变的。用滚动窗口监控关键指标的变化：

```python
def rolling_evaluation(
    strategy_returns: pd.Series,
    benchmark_returns: pd.Series,
    window: int = 60
) -> pd.DataFrame:
    """
    滚动窗口评估——看策略是否在衰减

    关键观察点：
    - 滚动夏普持续下降半年以上 → 策略在衰减
    - 滚动超额从正变负 → 策略已失效
    """
    results = []
    for i in range(window, len(strategy_returns) + 1):
        rets = strategy_returns.iloc[i - window:i]
        bench = benchmark_returns.iloc[i - window:i]

        ann_ret = (1 + rets.mean()) ** 252 - 1
        sharpe = rets.mean() / rets.std() * np.sqrt(252) if rets.std() > 0 else 0
        excess = (rets - bench).mean() * 252

        results.append({"日期": strategy_returns.index[i - 1],
                        "年化收益": ann_ret, "夏普比率": sharpe, "超额收益": excess})

    return pd.DataFrame(results).set_index("日期")
```

---

## 六、策略诊断流程

当策略表现不如预期时，按顺序排查，不要跳步：

```text
1. 基准对吗？
   → 沪深 300？中证 500？万得全 A？基准错了后面全错

2. 市场异常？
   → 检查是否有熔断、停市、极端行情

3. 执行偏差？
   → 实盘持仓 vs 回测信号对比（QMT/XTP 查持仓）

4. 超额在哪一层衰减？
   → Brinson 归因 → 是选股不行了还是行业偏了

5. 因子失效？
   → 滚动 IC 持续下降？因子收益率归零？

6. 成本被低估？
   → 实盘滑点 vs 回测滑点对比

7. 过拟合？
   → 样本外表现显著差于样本内
```

---

## 七、归因报告模板

```python
def attribution_report(
    strategy_returns: pd.Series,
    benchmark_returns: pd.Series,
    brinson_result: pd.DataFrame = None,
    name: str = "策略",
):
    """
    一站式归因报告
    输入：回测的收益序列 + Brinson 归因结果（可选）
    """
    excess = strategy_returns - benchmark_returns

    print(f"\n{'=' * 52}")
    print(f"  {name} — 绩效归因报告")
    print(f"{'=' * 52}")

    # 总体统计
    ann_ret = (1 + strategy_returns.mean()) ** 252 - 1
    ann_excess = excess.mean() * 252
    sharpe = strategy_returns.mean() / strategy_returns.std() * np.sqrt(252)
    rolling_excess_std = excess.rolling(60).mean().std() * 252

    print(f"\n  总体表现")
    print(f"    年化收益:       {ann_ret:.2%}")
    print(f"    年化超额:       {ann_excess:.2%}")
    print(f"    夏普比率:       {sharpe:.2f}")
    print(f"    超额波动率:     {rolling_excess_std:.2%}")

    # Brinson 归因
    if brinson_result is not None:
        alloc = brinson_result["配置效应"].sum()
        select = brinson_result["选择效应"].sum()
        print(f"\n  超额来源")
        print(f"    行业配置贡献:  {alloc:.2%}")
        print(f"    选股贡献:      {select:.2%}")

    print(f"\n{'=' * 52}\n")
```

---

## 八、常见绩效问题快速诊断

| 现象 | 可能原因 | 排查步骤 |
|------|---------|---------|
| 夏普 > 2 但超额持续下降 | 过拟合，样本外不行 | 缩短样本内，拉长样本外重测 |
| 收益曲线平稳但突现大回撤 | 市场结构变了 | 检查回撤期的因子暴露和行业分布 |
| 超额集中某几个月 | 运气驱动 | bootstrap 随机模拟验证显著性 |
| 多头收益好，多空收益差 | 做空端贡献小 | 因子只在做多端有效，考虑单向策略 |
| 回测好、实盘差 | 成本低估 + 执行偏差 | 对比实盘持仓、滑点、成交率 |
| 连续亏损超过历史最大回撤 | 策略已经失效 | 暂停实盘，重新从样本外测试开始 |

---

> **动手练习**：拿 multi-factor-model.md 的回测结果，跑一遍 `attribution_report`。回答三个问题：① 超额收益主要来自市场还是因子？② 选股和择时哪个贡献更大？③ 滚动夏普最近半年趋势如何？
