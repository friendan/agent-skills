---
name: ezui-progress
description: 指导如何使用EZUI框架的Progress进度条控件，包括htm属性设置、C++ API调用、布局原理、踩坑记录等。
---

# EZUI Progress 进度条控件指南

## 概述

Progress 进度条控件，类似 Bootstrap 的 `.progress` + `.progress-bar`，用于显示任务完成进度。支持百分比显示、颜色变体、条纹样式和动画条纹。

## 架构

```
Progress (HLayout)
├── ProgressBar (Label) — 进度条，颜色由 kind 控制，宽度按百分比
└── Spacer — 占满剩余空间
```

## 实现原理

- `Progress` 继承 `HLayout`，容器背景灰色
- `ProgressBar` 继承 `Label`，显示百分比文字和进度条颜色
- `Spacer` 占满剩余空间，确保进度条不会撑满容器
- 宽度用 `SetRateWidth(pct)` 按百分比分配，由 HLayout 布局
- 值过渡动画用 `Animation` 类驱动，600ms ease

## HTM 中使用

### 标签属性方式

```html
<progress value="50"></progress>
```

### 支持的属性

| 属性 | 说明 | 默认值 | 示例 |
|------|------|--------|------|
| `value` | 进度百分比（0~100） | `0` | `value="75"` |
| `kind` | 进度条颜色风格（复用 ControlKind） | `primary` | `kind="success"` |
| `height` | 进度条高度（像素） | `16` | `height="24"` |
| `show-label` | 是否显示百分比文字 | `false` | `show-label="true"` |
| `striped` | 是否条纹样式 | `false` | `striped="true"` |
| `animated` | 是否动画条纹 | `false` | `animated="true"` |

### 完整示例

```html
<!-- 基础进度 -->
<progress value="25"></progress>
<progress value="50" kind="success"></progress>
<progress value="75" kind="danger"></progress>

<!-- 不同颜色 -->
<progress value="60" kind="warning"></progress>
<progress value="40" kind="info"></progress>
<progress value="80" kind="dark"></progress>

<!-- 不同高度 -->
<progress value="50" kind="primary" height="8"></progress>
<progress value="50" kind="primary" height="20"></progress>
<progress value="50" kind="primary" height="28"></progress>

<!-- 显示百分比 -->
<progress value="33" kind="success" height="24" show-label="true"></progress>

<!-- 条纹和动画条纹 -->
<progress value="45" kind="info" striped="true"></progress>
<progress value="70" kind="warning" striped="true" animated="true"></progress>
```

## C++ API

```cpp
// 创建进度条
auto* pg = new Progress(parent);

// 设置值
pg->SetValue(75);
int val = pg->GetValue();

// 设置颜色
pg->SetKind(ControlKind::Success);

// 显示百分比文字
pg->SetShowLabel(true);
bool show = pg->IsShowLabel();

// 条纹
pg->SetStriped(true);
bool striped = pg->IsStriped();

// 动画条纹
pg->SetAnimated(true);
bool anim = pg->IsAnimated();

// 高度
pg->SetBarHeight(24);
int h = pg->GetBarHeight();
```

## 颜色处理

进度条颜色复用 `ControlKind` 体系（`g_controlKindColors`）：

- `kind` 的 `Normal.BackColor` 作为进度条颜色
- Progress 容器背景保持灰色（`#e9ecef`），不随 kind 改变
- 文字颜色固定为白色

## 关联代码

| 文件 | 内容 |
|------|------|
| `src/include/EzUI/Progress.h` | Progress + ProgressBar 类声明 |
| `src/sources/Progress.cpp` | Progress + ProgressBar 类实现 |
| `src/sources/UIManager.cpp` | `RegisterControl<Progress>("progress")` 注册 |
| `src/include/EzUI/EzUI.h` | `class Progress;` 前向声明 |

## 踩坑记录

### 坑1：Progress 容器背景被 kind 覆盖铺满

`SetAttribute("kind")` 通过基类 `Control::SetKind` → `ApplyControlKindColors` 把容器背景也设成了 kind 颜色，导致进度条看起来铺满。

**解决**：在 `SetAttribute("kind")` 处理后，立即恢复 `Style.BackColor = Color(233, 236, 239)`（灰色），只让 `ProgressBar` 子控件应用 kind 颜色。

### 坑2：进度条宽度撑满容器

最初用 `Width()` 计算百分比宽度，但父 HLayout 布局后 `Width()` 可能不是 `width="200"` 设置的值，导致 bar 宽度为 100% 容器。

**解决**：用 `SetRateWidth(pct)` 替代手动算宽度，让 HLayout 按百分比分配 `ProgressBar` 和 `Spacer` 的宽度。

### 坑3：`SetRateWidth` 后需要重新布局

`SetRateWidth` 设置了比例宽度后，HLayout 需要重新布局才能让 Spacer 正确占满剩余空间。

**解决**：`UpdateBar()` 中调用 `__super::OnLayout()` 触发重新布局。

### 坑4：子控件 `Add` 被外部调用

用户可能在 htm 中往 `<progress>` 标签里加子控件，破坏内部结构。

**解决**：重写 `Add` 方法，直接返回传入的 child 不添加。

### 坑5：`m_bar->SetText` 后文字不显示

`SetText` 后 label 没有重绘，或者 label 宽度为 0 导致文字被截断。

**解决**：`UpdateBar` 中调用 `m_bar->Invalidate()` 强制重绘，且使用 `SetRateWidth` 确保 label 有足够宽度。
