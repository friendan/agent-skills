---
name: ezui-spinner
description: 指导如何使用EZUI框架的Spinner加载控件，包括htm属性设置、C++ API调用、绘制逻辑、踩坑记录等。
---

# EZUI Spinner 加载控件指南

## 概述

Spinner 是一个旋转加载动画控件，类似 Bootstrap 的 `spinner-border`，用于指示组件或页面的加载状态。纯 GDI 绘制，不依赖 CSS 动画。

只实现了 **Border 模式（边框转圈）**，Grow 模式未实现。

## 实现原理

- `Spinner` 继承 `Label`，不绘制文字
- 用 D2D `Geometry` 构建 270° 弧线 path（HOLLOW 模式），`DrawGeometry` 绘制
- 用框架的 `Animation` 类驱动角度从 0→360 无限循环（设置 100 圈 75000ms 避免提前结束，`OnForePaint` 中检测 `IsStopped` 重启）
- 颜色复用全局 `g_controlKindColors` 体系，取 `Normal.BackColor` 作为圆环色
- 背景透明、无边框、禁用鼠标事件

## HTM 中使用

### 标签属性方式

```html
<spinner id="sp1" kind="primary"></spinner>
```

### 支持的属性

| 属性 | 说明 | 默认值 | 示例 |
|------|------|--------|------|
| `kind` | 颜色风格（复用 ControlKind） | `primary` | `kind="danger"` |
| `spinner-size` | 尺寸（像素） | `32` | `spinner-size="48"` |
| `spinner-sm` | 是否小尺寸（16px） | `false` | `spinner-sm="true"` |

### 完整示例

```html
<!-- 各种颜色 -->
<spinner kind="primary"></spinner>
<spinner kind="danger"></spinner>
<spinner kind="success"></spinner>
<spinner kind="warning"></spinner>
<spinner kind="info"></spinner>

<!-- 小尺寸 -->
<spinner kind="primary" spinner-sm="true"></spinner>
<spinner kind="danger" spinner-sm="true"></spinner>

<!-- 自定义尺寸 -->
<spinner kind="primary" spinner-size="48"></spinner>
<spinner kind="danger" spinner-size="64"></spinner>
```

## C++ API

```cpp
// 创建 Spinner
auto* sp = new Spinner(parent);

// 设置尺寸（像素）
sp->SetSpinnerSize(48);
int size = sp->GetSpinnerSize();

// 设置颜色风格
sp->SetKind(ControlKind::Primary);

// 控制动画
sp->Start();   // 开始旋转
sp->Stop();    // 停止旋转
bool spinning = sp->IsSpinning();
```

## 绘制逻辑

在 `OnForePaint` 中用 D2D `Geometry` 构建弧线 path：

1. `ApplySpinnerColor()` 从 `g_controlKindColors[kind].Normal.BackColor` 取主题色，存入 `Style.ForeColor`
2. 强制清除背景色（透明）、四边边框（0）、禁用 hit test
3. 计算居中绘制区域
4. 从当前角度 `m_angle` 到 `m_angle + 270°` 构建弧线 path
5. 弧线宽度 = 尺寸的 1/8（最小 1px）

```cpp
Geometry arcPath;
arcPath.BeginFigure(startPt, D2D1_FIGURE_BEGIN_HOLLOW);
arcPath.AddAcr(arcSeg);
arcPath.CloseFigure(D2D1_FIGURE_END_OPEN);
g.SetColor(spinnerColor);
g.DrawGeometry(&arcPath, m_borderWidth);
```

注意：270° > 180°，`D2D1_ARC_SEGMENT` 的 arcSize 必须设为 `D2D1_ARC_SIZE_LARGE`。

## 动画驱动

```
Animation 周期 750ms，设置 100 圈（75000ms），避免提前停止
OnForePaint 中检测 m_anim->IsStopped() 时重新 StartAnim()
ValueChanged → m_angle = value % 360 → Invalidate()
```

- 动画默认自动启动（`m_spinning = true`）
- 控件隐藏时自动停止动画（重写 `SetVisible`）
- 控件显示且 `m_spinning = true` 时自动恢复动画

## ApplySpinnerColor 要点

`kind` 属性处理流程：

1. 先让基类 `Control::SetAttribute` 处理 `kind` 解析 → 调用 `SetKind` → 调用 `ApplyControlKindColors`（设置了背景色、边框、前景色）
2. 然后 `ApplySpinnerColor()` 覆盖：
   - `Style.ForeColor` = kind 的 `Normal.BackColor`（主题色）
   - `Style.BackColor` = 透明（`Color(0)`）
   - 四边边框 = 0，`Style`/`HoverStyle`/`ActiveStyle` 的 `Border.Style` = `None`
3. `Init()` 中设置 `SetHitTestVisible(false)` 禁用鼠标事件

## 关联代码

| 文件 | 内容 |
|------|------|
| `src/include/EzUI/Spinner.h` | Spinner 类声明 |
| `src/sources/Spinner.cpp` | Spinner 类实现 |
| `src/sources/UIManager.cpp` | `RegisterControl<Spinner>("spinner")` 注册 |
| `src/include/EzUI/EzUI.h` | `class Spinner;` 前向声明 |

## 踩坑记录

### 坑1：`min` 宏冲突

`std::min` 被 Windows 的 `min` 宏展开导致编译错误。

**解决**：用 `(std::min)`（括号包裹防止宏展开）或在文件开头 `#undef min`。

### 坑2：`DrawArc` 未实现

`DXRender::DrawArc` 是空函数（注释标注"未实现"）。

**解决**：改用 D2D `Geometry` 构建弧线 path，用 `DrawGeometry` 绘制。注意 270° 弧要用 `D2D1_ARC_SIZE_LARGE`。

### 坑3：颜色获取

`Graphics` 没有 `GetColor()` 方法，只有 `SetColor()`。

**解决**：直接设置颜色而不保存恢复（每次绘制前设一次即可）。

### 坑4：`UIString` 没有 `utf8()` 方法

`ui_text::String` 继承 `std::string`，本身就是 UTF-8 编码。

**解决**：直接用 `value.c_str()` 获取 C 字符串。

### 坑5：新文件未被 CMake 编译

`CMakeLists.txt` 使用 `file(GLOB)`，但 GLOB 只在 cmake 配置时执行，添加新文件后需要重新运行 cmake 才能被识别。

**解决**：在 `build_clang_ninja_debug` 目录重新执行 cmake 配置，或删除已存在的 build 目录重新执行 `build_clang_ninja_debug.bat`。

### 坑6：`ApplyControlKindColors` 自动设置 1px 边框

`ApplyControlKindColors` 当 `BackColor` 或 `BorderColor` 非0时会设置四边边框为1px。

**解决**：`ApplySpinnerColor()` 中强制清零 `Style.Border.Left/Top/Right/Bottom` 并设置 `Border.Style = None`，同时清掉 `HoverStyle` 和 `ActiveStyle` 的边框。

### 坑7：Animation 到达终点后停止

`Animation` 到达 `endValue` 后自动调用 `Timer::Stop()`，不再继续。

**解决**：设置足够大的圈数（100圈 × 750ms = 75000ms），同时在 `OnForePaint` 中检测 `IsStopped()` 时重新启动。

### 坑8：Spinner 需要禁用鼠标事件

Spinner 是装饰性控件，不需要鼠标交互。默认的 `SetKind` 会设置 `HoverStyle` 和 `ActiveStyle`，鼠标移入时会出现 hover 效果。

**解决**：`Init()` 中调用 `SetHitTestVisible(false)`，彻底禁掉鼠标事件。
