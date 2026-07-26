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
- 用 `Graphics::DrawPie` 画一个 270° 的弧线模拟"缺口圆环旋转"
- 用框架的 `Animation` 类驱动角度从 0→360 无限循环
- 颜色复用全局 `g_controlKindColors` 体系

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

在 `OnForePaint` 中用 `DrawPie` 画弧线：

1. 获取 kind 颜色作为 spinner 颜色（优先 `GetForeColor()`，后备 `GetBackColor()`，最后回退 Bootstrap Primary 蓝）
2. 计算居中绘制区域
3. 从当前角度 `m_angle` 到 `m_angle + 270°` 画弧
4. 弧线宽度 = 尺寸的 1/8（最小 1px）

```cpp
g.SetColor(spinnerColor);
g.DrawPie(ellipseRect, startAngle, endAngle, m_borderWidth);
```

## 动画驱动

```
Animation 周期 750ms，从 0 → 360 循环
ValueChanged → m_angle = value % 360 → Invalidate()
```

- 动画默认自动启动（`m_spinning = true`）
- 控件隐藏时自动停止动画（重写 `SetVisible`）
- 控件显示且 `m_spinning = true` 时自动恢复动画

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

**解决**：改用 `DrawPie` 画 270° 扇形弧线替代。

### 坑3：颜色获取

`Graphics` 没有 `GetColor()` 方法，只有 `SetColor()`。

**解决**：直接设置颜色而不保存恢复（每次绘制前设一次即可）。

### 坑4：`UIString` 没有 `utf8()` 方法

`ui_text::String` 继承 `std::string`，本身就是 UTF-8 编码。

**解决**：直接用 `value.c_str()` 获取 C 字符串。

### 坑5：新文件未被 CMake 编译

`CMakeLists.txt` 使用 `file(GLOB)`，但 GLOB 只在 cmake 配置时执行，添加新文件后需要重新运行 cmake 才能被识别。

**解决**：在 `build_clang_ninja_debug` 目录重新执行 cmake 配置，或删除已存在的 build 目录重新执行 `build_clang_ninja_debug.bat`。
