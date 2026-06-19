---
name: origin-datalab
description: 用 Python 操控 Origin/OriginPro 进行实验数据作图和数据分析。支持originpro脚本和matplotlib平替两种模式。
---

# Origin 数据处理通用技能

## 使用场景

用户需要：
- 用 Origin 处理实验数据、作图、拟合
- 批量出图、标注特征点
- 导出高清图片交作业/论文

## 核心原则

```
数据处理 → Python
作图渲染 → Origin / matplotlib
样式模板 → 一次配好，重复用
导出保存 → 高清无码
```

## 两种工作模式

### 模式A：Origin + originpro（首选）
适合：正式实验报告、需要 .opju 源文件的场景
限制：需安装 Origin，破解版导出有水印

### 模式B：matplotlib 平替（备选）
适合：Origin 卡死、导出失败、没装 Origin 的场景
优点：无水印、跨平台、轻量

---

## 标准工作流

不管哪种模式，按这个顺序：

```
Step 1: 读数据  →  搞清文件格式（分隔符、跳过头几行、列含义）
Step 2: 清洗    →  单位换算、去噪、筛选
Step 3: 画图    →  选图类型、设坐标轴、叠加图例
Step 4: 标注    →  标交点、拐点、贴文字
Step 5: 导出    →  PNG/PDF，300 DPI 以上
```

---

## originpro API 速查（避坑版）

### 工作表和导入
| 正确写法 | 错误写法（踩过坑） |
|---------|-----------------|
| `wb = op.new_book('w', lname='书名')` | |
| `ws = wb[0]` | ❌ `wb.layers[0]` |
| `wb.add_sheet(name='sheet名')` | ❌ `add_layer()` |
| `op.new_sheet()` → `ws.name = '名'` | ❌ `op.new_sheet(name='名')` |
| `ws.from_list(idx, data, '列名')` | ❌ `from_file(path, options=...)` |
| `ws.cols` 返回 int（列数） | ❌ `ws.cols[0]` 当列表用 |
| `ws.cols_axis('xy', 0, 1)` | ❌ `col.type = 'X'` |
| `ws.to_list(idx)` 取列数据 | ❌ `ws.cols[idx]` |

### 作图
| 正确写法 | 错误写法 |
|---------|---------|
| `gl.add_plot(ws, coly=Y, colx=X, type='line')` | ❌ `type='scatter_line'` |
| `gl.xscale = 'log10'` | ❌ `'log'` |
| `plot.style = 'Dash'` | |
| ❌ `gl.create_text()` 不存在 → 手动标 | |

### 导出与窗口管理
| 正确写法 | 说明 |
|---------|------|
| `op.pages('g')` | 列出所有图（不是 `graph` 也不是 `graphs`） |
| `g.save_fig(path, type='png', width=800)` | 导出图（破解版可能失败） |
| `op.LT_execute('expGraph type:=png path:="...";')` | LabTalk 导出备选 |

### 多行代码运行
Python 控制台只能跑单行 → 写成 `.py` 文件，用 `exec(open("path.py").read())` 跑。

---

## 常见坑（已踩过，别重复踩）

1. **Origin 卡死** → 一次别开太多图窗口，分批出图
2. **导出没反应** → 破解版导出功能被阉割，换 matplotlib
3. **数据导入失败** → 永远用 `from_list` 手动导入，别信 `from_file`
4. **对数坐标报错** → `'log10'` 不是 `'log'`
5. **图名找不到** → `find_graph` 按短名搜，用 `pages('g')` 遍历
6. **Ctrl+J 复制有水印** → 破解版 Origin 全部导出都有水印

---

## matplotlib 平替模板

```python
import matplotlib.pyplot as plt
import numpy as np

# 中文字体
plt.rcParams['font.sans-serif'] = ['SimHei', 'Microsoft YaHei']
plt.rcParams['axes.unicode_minus'] = False

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(x_data, y_data, 'k.-', markersize=5, linewidth=1.5, label='数据')
ax.set_xlabel('X轴标签 (单位)')
ax.set_ylabel('Y轴标签 (单位)')
ax.set_title('图标题')
ax.legend()
ax.grid(True, alpha=0.3)

# 对数坐标
# ax.set_xscale('log')
# ax.set_yscale('log')

fig.tight_layout()
fig.savefig('输出路径.png', dpi=300)
```

### 多线叠加
```python
colors = ['#E60000', '#005CFF', '#00A600', '#D200D2']
labels = ['Ch1', 'Ch2', 'Ch3', 'Ch4']
for i in range(4):
    ax.plot(x, y[:, i], '.-', color=colors[i], label=labels[i])
ax.legend()
```

---

## 通用图类型模板

### 1. 单条曲线（X-Y）
```python
gl.add_plot(ws, coly=1, colx=0, type='line')
```

### 2. 多线叠加（Multi-Y）
```python
for i in range(N):
    gl.add_plot(ws, coly=2+i*2, colx=1, type='line')
```
matplotlib：用循环 `ax.plot()`

### 3. 对数坐标
```python
gl.xscale = 'log10'
```
matplotlib：`ax.set_xscale('log')`

### 4. 虚线
```python
plot.style = 'Dash'
```
matplotlib：`ax.plot(x, y, '--')`

---

## 判断流程图

```
用户说"处理数据"
    ↓
有 Origin 且能跑？──是──→ 模式A：originpro
    ↓ 否
装不了 Origin？    ──→ 模式B：matplotlib
    ↓
Origin 卡死/导出失败？──→ 模式B：matplotlib
```
