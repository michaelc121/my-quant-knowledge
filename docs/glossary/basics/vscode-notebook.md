# VS Code 内置 Notebook

> **来源文档**：[Jupyter](../jupyter.md)
> **前置要求**：已安装 VS Code、Python、pip
> **更新日期**：2026-07-02

## 一句话理解

VS Code 内置了和 JupyterLab 几乎一样的 Notebook 体验，但少开一个浏览器窗口，代码和 Notebook 可以在同一个编辑器里切换，适合已经习惯用 VS Code 写代码的人。

---

## 设置步骤

### 1. 确认环境

确保 VS Code 已安装 **Python 扩展**（微软官方，id: `ms-python.python`），它自带 Notebook 支持。在扩展面板搜 `Python` 安装即可。

### 2. 创建或打开 Notebook

三种方式：

```
方式 A：在 VS Code 中新建
  Ctrl+Shift+P → "Create: New Jupyter Notebook" → 保存为 .ipynb

方式 B：直接打开已有文件
  双击任意 .ipynb 文件，VS Code 自动以 Notebook 模式打开

方式 C：从命令面板
  Ctrl+Shift+P → "Notebook: Create New Notebook"
```

### 3. 选择 Kernel

新建 Notebook 后，右上角会显示 **"选择内核"** 按钮：

- 点它 → 选择你装好 `pandas`、`numpy` 的 Python 环境
- 如果用的是 conda 环境，VS Code 会自动扫描所有环境列出来

```text
✓ 选中正确内核后，状态栏会显示 Python 版本和环境名
  例如: Python 3.11.8 ('base': conda)
```

### 4. 开始写 Cell

和 JupyterLab 操作一致：`Shift+Enter` 运行，`A` / `B` 插入 Cell，`M` 切换 Markdown。

---

## VS Code Notebook vs JupyterLab

| 维度 | VS Code Notebook | JupyterLab |
|------|-----------------|------------|
| 安装 | 装 VS Code + Python 扩展即可 | 需单独 `pip install jupyterlab` |
| 启动 | 打开 .ipynb 文件即用 | 终端运行 `jupyter lab`，等浏览器打开 |
| 多标签 | 依赖 VS Code 的 Tab 管理 | 原生多标签面板 |
| 变量查看 | 有专门的变量面板（左栏 Debug） | 没有内置变量浏览器 |
| 代码补全 | VS Code 的 Pylance 补全，非常强 | 自带补全，但不如 VS Code |
| Git 集成 | 侧边栏直接做 Git 操作 | 需要切换到终端或 Git GUI |
| 插件生态 | VS Code 海量插件 | Jupyter 扩展相对少 |
| 运行速度 | 底层同样是 Jupyter Kernel，速度无差异 | 同 |

---

## 量化实战演示

以下步骤在 VS Code Notebook 中跟着操作一遍：

### Cell 1：导入与参数

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# 这里的输出会直接显示在 Cell 下方
print("pandas:", pd.__version__)
print("numpy:", np.__version__)
```

### Cell 2：快速验证数据可用

```python
# 用随机数据模拟拿到的日线数据
dates = pd.date_range("2024-01-01", "2024-12-31", freq="D")
np.random.seed(42)
df = pd.DataFrame({
    "close": 100 + np.cumsum(np.random.randn(len(dates)) * 0.5)
}, index=dates)

# 内联绘图 —— 图直接显示在 Cell 下方
df["close"].plot(title="模拟日线", figsize=(10, 4))
plt.grid(alpha=0.3)
plt.show()
```

### Cell 3：调参 —— 不改前面 Cell，只改这一个

```python
# 先试试 5 日均线
ma = df["close"].rolling(5).mean()
ma.plot(label="ma5")
df["close"].plot(label="close")
plt.legend()
plt.show()

# 不满意？改上面的 5 → 20，重新运行这个 Cell 就好
```

### Cell 4：导出为 .py

VS Code 提供了**一键导出为 Python 脚本**的功能：

```
右击 Notebook 任意空白处 → "Export" → "Python Script"
```

所有代码 Cell 会被合并成一个 `.py` 文件，Markdown 转为注释。这就是"Notebook 探索 → .py 固化"的标准转换路径。

---

## 什么时候选 VS Code Notebook

| 建议选 VS Code | 建议选 JupyterLab |
|---------------|------------------|
| 你已经在用 VS Code 写策略脚本 | 你习惯浏览器环境 |
| 经常需要在 Notebook 和 .py 文件之间切换 | 经常同时打开多个 Notebook 对比 |
| 需要 VS Code 的 Git / 终端 / 调试器 | 需要 Jupyter 的扩展插件（如 nbdime）|
| 电脑配置一般，不想多开一个浏览器进程 | 机器性能足够，浏览器不是负担 |

对于量化初学者，两个选哪个都可以，**核心体验没有本质差别**。

---

## 注意事项

- VS Code Notebook 依赖 Python 扩展，装完后**记得重启 VS Code**
- 如果 Kernel 连不上：`Ctrl+Shift+P` → "Python: Select Interpreter" 手动指定
- `.ipynb` 文件是 JSON 格式，Git 合并冲突不好处理。团队协作时可以用 `jupytext` 把 Notebook 转成 `.py` 格式提交
- VS Code 的 Notebook 变量浏览器在 Debug 面板下面，不是自动刷新的，需要点一下刷新按钮

---

## 回到来源文档

[← 返回 Jupyter 详解](../jupyter.md)
