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

---

## 两代方案对比

### 旧方案：Origin Python Console（手动模式）
写 `.py` 脚本 → 用户打开 Origin → **Tools → Python Console** → 手动执行：
```python
exec(open("path.py").read())
```
- ❌ 每次都要用户手动操作
- ❌ 不能远程自动化

### 新方案：originpro 自动化 ✅ 当前标准
安装 `originpro` 到系统 Python → 脚本直接通过 COM 控制 Origin：
```bash
pip install originpro
```
- ✅ 全程自动化，用户只需打开 Origin
- ✅ 脚本由 Agent 直接运行

---

## 三种 API 层级（必须分清！）

```
op.lt_exec('...')                       # ① originpro 的 LabTalk 封装
OriginExt.Application().LT_execute()    # ② 新建 COM 连接
gl.obj.Activate(); gl.obj.Execute()     # ③ 图层的 COM 对象
```

| 层级 | 上下文 | 适用范围 | 踩坑指数 |
|:----|:------|:---------|:--------:|
| ① `op.lt_exec()` | originpro 内部 | ⚠️ 简单命令，部分属性赋值会失败 | ⭐⭐⭐ |
| ② `OriginExt.Application().LT_execute()` | 新建 COM 连接 | ⚠️ **看不到 originpro 创建的对象** | ⭐⭐⭐⭐⭐ |
| ③ **`gl.obj.Activate()` + `gl.obj.Execute()`** | 具体图层的 COM 对象 | ✅ **最可靠，推荐** | ⭐ |

**结论：优先用 ③。** 要动哪个图/层，就用它的 `gl.obj.Execute()`。

---

## 坐标轴避坑指南（今天踩得最惨）

### ❌ 错误写法（全部无效）
```python
op.lt_exec('layer.x.axis = 1;')           # 错！layer.x.axis 不是有效属性
op.lt_exec('layer.x.axistype = 2;')       # 错！axistype 在 Origin 2024 设不了
op.lt_exec('axs -p xy 0 0;')             # 错！Origin 2024 报"未定义变量"
```

### ✅ 正确写法

**显示坐标轴：**
```python
gl.obj.Activate()
gl.obj.Execute('layer.showx = 1;')       # showx/showy 才是正确属性！
gl.obj.Execute('layer.showy = 1;')
```

**坐标轴在原点交叉：**
```python
gl.obj.Execute('layer.x.axisposition = 0;')
gl.obj.Execute('layer.y.axisposition = 0;')
```

**设轴类型：** 用 Python API（可靠），不用 LabTalk
```python
ax_x = gl.axis('x')
ax_y = gl.axis('y')
ax_x.type = 2            # 2 = 带刻度的线
ax_y.type = 4            # 4 = 带箭头
ax_x2.type = 0           # 隐藏顶轴
ax_y2.type = 0           # 隐藏右轴
ax_x.set_limits(-21000, 21000)
ax_y.set_limits(-13, 13)
```

**注意：** Python API 的 `ax_x.type` 和 LabTalk 的 `layer.x.type` 是 **两个不同属性**，互不影响。

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
| ❌ `gl.create_text()` 不存在 → 用 `gl.obj.Execute('label -a x y "text";')` | |

### 导出与窗口管理
| 正确写法 | 说明 |
|---------|------|
| `op.pages('g')` | 列出所有图 |
| `g.save_fig(path, type='png', width=800)` | Python API 导出（推荐） |
| `gl.obj.Execute('expGraph type:=png path:="...";')` | LabTalk 导出备选 |

### 标注文字
```python
gl.obj.Execute(f'label -a {x} {y} "text";')
```

---

## 常见坑（已踩过，别重复踩）

### 环境类
1. **`op.attach()` 可能启动隐藏 Origin** → 产生多个 Origin64.exe 进程，先 `taskkill` 再让用户手动开
2. **Origin 多个进程时连错** → originpro 连的 Origin 和用户看到的可能是同一个，用 `op.pages()` 验证
3. **`op.lt_exec` 部分命令弹错** → 改用 `gl.obj.Execute()`

### 坐标轴类
4. **`layer.x.axis` 不是有效属性** → 正确属性是 `layer.showx` / `layer.showy`
5. **`axs -p xy 0 0;` 在 Origin 2024 报错** → 用 `layer.x.axisposition = 0;`
6. **Python API 的 `ax_x.type` 和 LabTalk 的 `layer.x.type` 不同** → 各设各的，互不影响

### 窗口管理类
7. **`win -a HLoop;` 找不到图** → originpro 创建的图在 COM 中可能不可见，改用 `gl.obj.Activate()`
8. **`page.active` 始终为 0** → COM 和 originpro 的活动页状态不同步，用 `gl.obj.Activate()` 绕过

### 一般类
9. **Origin 卡死** → 一次别开太多图窗口，分批出图
10. **数据导入失败** → 永远用 `from_list` 手动导入，别信 `from_file`
11. **对数坐标报错** → `'log10'` 不是 `'log'`
12. **图名找不到** → `find_graph` 按短名搜，用 `pages('g')` 遍历

---

## 标准工作流（originpro 自动化版）

```python
import originpro as op
op.attach()

# 1. 数据
wb = op.new_book('w', lname='Data')
ws = wb[0]
ws.from_list(0, field, 'Field')
ws.from_list(1, moment, 'Moment')

# 2. 画图
gp = op.new_graph('Graph')
gl = gp[0]
p = gl.add_plot(ws, coly=1, colx=0, type='line')
p.color = '#000000'
gl.rescale()

# 3. 设轴（用 gl.obj 是关键）
gl.obj.Activate()
gl.obj.Execute('layer.showx = 1;')
gl.obj.Execute('layer.showy = 1;')
ax = gl.axis('x')
ax.type = 2
ax.set_limits(-21000, 21000)

# 4. 标注
gl.obj.Execute(f'label -a {x} {y} "text";')

# 5. 导出
gl.obj.Execute(f'expGraph type:=png path:="out.png" width:=1200;')
```

---

## 磁滞回线分析模板

```python
import originpro as op, os

# 连接 Origin
op.attach()

# 创建图表
wb = op.new_book('w', lname='Data')
ws = wb[0]
ws.from_list(0, field_data, 'Field_Oe')
ws.from_list(1, moment_data, 'Moment_emu')
ws.cols_axis('xy', 0, 1)

gp = op.new_graph('HLoop')
gl = gp[0]

# 主回线
p1 = gl.add_plot(ws, coly=1, colx=0, type='line')
p1.color = '#000000'; p1.width = 1.5

# 起始磁化曲线（红色虚线）
p2 = gl.add_plot(ws, coly=3, colx=2, type='line')
p2.color = '#E60000'; p2.width = 2; p2.style = 'Dash'

# 标记点（红色圆）
p3 = gl.add_plot(ws, coly=5, colx=4, type='scatter')
p3.color = '#E60000'; p3.symbol = 2; p3.size = 12

gl.rescale()

# 设轴
gl.obj.Activate()
gl.obj.Execute('layer.showx = 1;')
gl.obj.Execute('layer.showy = 1;')
gl.obj.Execute('layer.showx2 = 0;')
gl.obj.Execute('layer.showy2 = 0;')
gl.obj.Execute('layer.x.axisposition = 0;')
gl.obj.Execute('layer.y.axisposition = 0;')

ax_x = gl.axis('x'); ax_y = gl.axis('y')
ax_x.type = 2; ax_y.type = 2
ax_x.set_limits(-21500, 21500)
ax_y.set_limits(-13, 13)

gl.xlabel = 'Magnetic Field (Oe)'
gl.ylabel = 'Moment (emu)'

# 标注
gl.obj.Execute(f'label -a {x} {y} "Ms = {Ms:.3f} emu";')
gl.obj.Execute(f'label -a {x} {y} "Mr = {Mr:.3f} emu";')
gl.obj.Execute(f'label -a {x} {y} "Hc = {Hc:.0f} Oe";')

# 导出
png = 'output.png'
gl.obj.Execute(f'expGraph type:=png path:="{png}" width:=1200;')
```

---

## matplotlib 平替模板

### ⚠️ 中文显示问题
matplotlib 默认字体不含中文，必须手动指定中文字体，否则中文会变成方框 □。

```python
import matplotlib.pyplot as plt
import numpy as np

# ====== 中文字体配置（必写！不然中文变方框）======
plt.rcParams['font.sans-serif'] = [
    'DejaVu Sans',          # 放第一位！支持 ° δ ± 等特殊符号
    'Microsoft YaHei',      # 微软雅黑（中文用）
    'SimHei',               # 黑体
]
plt.rcParams['axes.unicode_minus'] = False  # 修复负号显示

# 如果上述字体还不行，手动指定路径（用你最熟悉的字体）：
# from matplotlib import font_manager
# font = font_manager.FontProperties(fname='C:/Windows/Fonts/msyh.ttc')
# ax.set_xlabel('中文标签', fontproperties=font)

# ====== 画图 ======
fig, ax = plt.subplots(figsize=(8, 6))

ax.plot(field, moment, 'k-', linewidth=1.5, label='数据线')

# 坐标轴在原点
ax.spines['left'].set_position('zero')
ax.spines['bottom'].set_position('zero')
ax.spines['right'].set_color('none')
ax.spines['top'].set_color('none')

# 箭头
ax.plot(1, 0, '>k', transform=ax.get_yaxis_transform(), clip_on=False)
ax.plot(0, 1, '^k', transform=ax.get_xaxis_transform(), clip_on=False)

ax.set_xlim(-21500, 21500)
ax.set_ylim(-13, 13)
ax.set_xlabel('Magnetic Field (Oe)')
ax.set_ylabel('Moment (emu)')

# 标注
ax.annotate(f'$M_s$ = {Ms:.3f}',
            xy=(x, y), fontsize=11, fontweight='bold',
            bbox=dict(boxstyle='round', facecolor='white', alpha=0.8))

plt.tight_layout()
plt.savefig('output.png', dpi=300, bbox_inches='tight')
```

---

## 判断流程图

```
用户说"处理数据"
    ↓
有 Origin 且能跑？──是──→ 模式A：originpro 自动化
    ↓ 否                       ↓
装不了 Origin？    ──→ 模式B：matplotlib
    ↓
Origin 卡死/导出失败？──→ 模式B：matplotlib
    ↓
LabTalk 引擎有 Bug？  ──→ 模式B：matplotlib（别硬刚）
```

## 环境配置

- Origin 安装路径：`E:\Origin2024\Origin2024\`（新安装）或 `E:\Origin\Origin\`（旧版）
- Python：Anaconda base（3.12.7），已装 originpro
- 数据路径：`G:\大学课业\材料专业实验\`
- 输出路径：实验目录下的 `Output\` 文件夹
