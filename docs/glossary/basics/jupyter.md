# Jupyter

> **来源文档**：[Python 量化工具栈速查](../../docs/basics/python-stack.md)
> **前置要求**：已阅读 python-stack.md 第四节"Jupyter Notebook"
> **更新日期**：2026-07-02

## 一句话理解

Jupyter 是一个**交互式编程环境**，让你能边写代码边看结果，中间可以随时插入文字和图表。它是最适合量化探索阶段的工作台，但不适合用来跑实盘。

---

## 两个名字的关系

| 名称 | 说明 |
|------|------|
| **Jupyter Notebook**（经典版） | 老牌产品，每个 `.ipynb` 文件是一个文档，在浏览器中打开。简单直接。 |
| **JupyterLab**（下一代） | 功能更强的 IDE 式界面，支持多标签页、拖拽单元格、文件浏览器、终端。**新用户建议直接用这个。** |

JupyterLab 向下兼容 Notebook 的所有功能，两者打开的是同一种 `.ipynb` 文件。

---

## 为什么量化离不开 Jupyter

量化的研究流程天然是**探索式**的：拉数据 → 看形态 → 算因子 → 画图 → 调参数 → 再看。每一步都需要即时反馈。

```text
传统脚本模式：
  改代码 → 跑整个脚本 → 等输出 → 啊不对 → 改代码 → 又跑一遍...

Jupyter 模式：
  Cell 1: 拉数据（只跑一次）
  Cell 2: 画图看形态（反复调参数重跑）
  Cell 3: 算因子（基于 Cell 1 的结果，不用重复拉数据）
  Cell 4: 回测（基于前序结果，增量执行）
```

**核心价值**：每个 Cell 可以独立执行，不浪费前面已经算好的结果。这在调参和做 EDA 时节约大量时间。

---

## 量化中的典型用法

### 1. 数据探查（EDA）

```python
# Cell 1: 拉数据
df = pd.read_csv("data/000300.csv", parse_dates=["date"], index_col="date")

# Cell 2: 看一眼
df.head()
df.describe()
df.isna().sum()

# Cell 3: 画图看分布
df["close"].plot()

# Cell 4: 发现异常，回来清洗
df = df[df["volume"] > 0]
```

### 2. 策略原型验证

```python
# Cell 5: 算信号
ma_short = df["close"].rolling(5).mean()
ma_long  = df["close"].rolling(20).mean()

# Cell 6: 看一眼信号质量
fig, ax = plt.subplots(2, 1)
ax[0].plot(df["close"])
ax[0].plot(ma_short, label="ma5")
ax[0].plot(ma_long, label="ma20")
ax[1].plot(ma_short - ma_long, label="diff")
plt.show()

# Cell 7: 不行，换参数
ma_short = df["close"].rolling(10).mean()
```

### 3. 组织研究笔记

Jupyter 的 Markdown Cell 可以写公式、贴图片、做笔记。一份 Notebook 本身就是一份**可执行的实验报告**，结果和代码在一起，不会出现"图在 PPT 里，数据去哪儿了"的情况。

---

## 重要注意事项

### 变量状态管理

Notebook 的变量是**全局共享**的，Cell 的执行顺序决定了当前变量值。

**原则**：每次重启 Kernel 后，**从上到下依次执行所有 Cell**，不要跳跃。

### 不要用来跑实盘

- Notebook 是同步执行，翻车时会卡死整个界面
- 依赖浏览器，网络或进程意外中断会导致未保存的工作丢失
- `.ipynb` 文件是 JSON 格式，Git diff 不友好

**正确分工**：

| 阶段 | 工具 |
|------|------|
| 探索、调参、可视化 | Jupyter（Notebook / Lab） |
| 固化策略逻辑 | `.py` 脚本 + 函数封装 |
| 定时回测、实盘运行 | `.py` 脚本 + 定时任务 |

---

## 不是只有 Jupyter

| 替代方案 | 适合场景 |
|----------|---------|
| [VS Code 内置 Notebook](vscode-notebook.md) | 如果你已经在用 VS Code 写代码，少开一个浏览器窗口，详见 [使用指南](vscode-notebook.md) |
| Quarto | 需要生成更专业的研究报告（支持导出 PDF/Word） |
| 纯 `.py` + 终端 | 确定性的批处理流程，不需要交互 |

对于量化初学阶段，**JupyterLab 是性价比最高的选择**，学一个够用很久。

---

## 回到来源文档

[← 返回 Python 量化工具栈速查](../../docs/basics/python-stack.md)
