---
name: ezui-control-kind
description: 指导如何使用EZUI框架通用风格体系（ControlKind），包括全局颜色配置、htm kind属性设置、Bootstrap风格按钮，以及TabButton集成ControlKind的注意事项。
---

# EZUI ControlKind 通用控件风格体系

## 概述

`ControlKind` 是 EZUI 框架通用的控件风格枚举，参考 Bootstrap 5.3.8 的按钮变体设计，所有控件（Label、Button、TabButton 等）都能使用。

风格颜色存储在全局数组 `g_controlKindColors[10]` 中，修改一处即可全局换肤。

## 核心概念

### ControlKind 枚举

定义在 `EzUI.h` 中：

```cpp
enum class ControlKind {
    Default,    // 默认无样式
    Primary,    // 主要（蓝）
    Secondary,  // 次要（灰）
    Success,    // 成功（绿）
    Danger,     // 危险（红）
    Warning,    // 警告（黄）
    Info,       // 信息（浅蓝）
    Light,      // 亮色
    Dark,       // 暗色
    Link        // 链接样式（透明背景，蓝色文字）
};
```

### 颜色数据结构

```cpp
struct ControlKindColors {
    struct StateColors {
        Color BackColor;
        Color BorderColor;
        Color ForeColor;
    };
    StateColors Normal;   // 正常状态
    StateColors Hover;    // 鼠标悬停
    StateColors Active;   // 鼠标按下
};

// 全局颜色配置表
extern ControlKindColors g_controlKindColors[10];
```

### 初始化

在 `Application` 构造函数中自动调用 `InitDefaultControlKindColors()`，初始化 Bootstrap 5.3.8 标准颜色值。程序启动后可修改 `g_controlKindColors` 全局换肤。

## 接口

### Control 基类（所有控件可用）

```cpp
void SetKind(ControlKind kind);     // 设置风格
ControlKind GetKind();              // 获取当前风格
```

### 全局辅助函数

```cpp
void ApplyControlKindColors(Control* ctl, ControlKind kind);
```

### HTM 属性

所有控件（`<button>`、`<label>`、`<tabbutton>` 等）都支持 `kind` 属性：

```html
<button kind="primary"></button>
<label kind="danger"></label>
<tabbutton kind="info"></tabbutton>
```

`kind` 属性在 `Control::SetAttribute` 中统一解析，不需要子类重复实现。

## 实现原理

`ApplyControlKindColors` 将 `g_controlKindColors[kind]` 中的 Normal/Hover/Active 三状态颜色分别应用到控件的 `Style`、`HoverStyle`、`ActiveStyle`：

```cpp
applyToStyle(ctl->Style,      colors.Normal);
applyToStyle(ctl->HoverStyle, colors.Hover);
applyToStyle(ctl->ActiveStyle, colors.Active);
```

EZUI 框架在鼠标事件中自动切换 `Control::State`（Static → Hover → Active），并使用 `GetStyle(State)` 获取对应状态的样式来绘制。因此给控件设置 `HoverStyle` 和 `ActiveStyle` 后框架自动处理状态切换。

## 全局换肤

```cpp
// 将所有 Primary 按钮改为红色主题
g_controlKindColors[(int)ControlKind::Primary].Normal.BackColor  = Color(220, 40, 40);
g_controlKindColors[(int)ControlKind::Primary].Normal.BorderColor = Color(220, 40, 40);
g_controlKindColors[(int)ControlKind::Primary].Hover.BackColor   = Color(190, 30, 30);
g_controlKindColors[(int)ControlKind::Primary].Hover.BorderColor = Color(170, 25, 25);
g_controlKindColors[(int)ControlKind::Primary].Active.BackColor  = Color(170, 25, 25);

// 所有使用 ControlKind::Primary 的控件立即生效（需调用 SetKind 重新应用）
```

## Button 控件

`Button` 继承 `Label`，唯一的特殊之处是构造时设定了手型光标：

```cpp
void Button::Init() {
    Style.Cursor = LoadCursor(Cursor::HAND);
}
```

Button 没有重写 `SetKind`，也没有重写 `SetAttribute` 的 `kind` 解析，全部重用 Control 基类的实现。

## TabButton + ControlKind 的特殊处理

### 问题背景

TabButton 继承 `HLayout`，但自身 `SetHitTestVisible(false)`，鼠标事件由 TabBar 代理。因此 TabButton 的 hover 效果**不能依赖框架的 HoverStyle 自动切换**，而由 TabBar 手动设置。

### TabBar 中的 hover 处理

`TabBar::OnMouseMove` 中手动管理 hover 状态：

```cpp
if (hover) {
    if (tab->GetKind() == ControlKind::Default) {
        tab->Style.BackColor = Color(225, 230, 240);
    } else {
        tab->Style.BackColor = tab->HoverStyle.BackColor;
    }
} else {
    tab->SetActive(tab->IsActive());
}
```

### TabButton::UpdateStyle 中的处理

`UpdateStyle` 由 `SetActive` 调用，用于恢复标签到正确颜色：

```cpp
void TabButton::UpdateStyle() {
    if (m_active) {
        // 激活标签：白色背景 + 深色文字，无边框
        Style.BackColor = Color(255, 255, 255);
        m_titleLabel->Style.ForeColor = Color(50, 50, 50);
        Style.Border.Style = StrokeStyle::None;
    }
    else if (GetKind() != ControlKind::Default) {
        ApplyControlKindColors(this, GetKind());
    }
    else {
        Style.BackColor = Color(210, 215, 225);
        m_titleLabel->Style.ForeColor = Color(100, 100, 110);
        Style.Border.Bottom = 1;
        Style.Border.Color = Color(180, 180, 190);
        Style.Border.Style = StrokeStyle::Solid;
    }
}
```

## Bootstrap 5.3.8 默认颜色值

| 变体 | Normal 背景 | Normal 文字 | Hover 背景 | Active 背景 |
|------|------------|------------|------------|-------------|
| Primary | #0d6efd | #fff | #0b5ed7 | #0a58ca |
| Secondary | #6c757d | #fff | #5c636a | #565e64 |
| Success | #198754 | #fff | #157347 | #146c43 |
| Danger | #dc3545 | #fff | #bb2d3b | #b02a37 |
| Warning | #ffc107 | #000 | #ffca2c | #ffcd39 |
| Info | #0dcaf0 | #000 | #31d2f2 | #3dd5f3 |
| Light | #f8f9fa | #000 | #d3d4d5 | #c6c7c8 |
| Dark | #212529 | #fff | #424649 | #4d5154 |
| Link | 透明 | #0d6efd | 透明 | 透明 |

## 关键注意事项

### 1. ControlKind 颜色会被手动覆盖

如果手动设置 `Style.BackColor`，会覆盖 `ControlKind` 设置的颜色。需要在手动设色的地方加 `GetKind()` 判断。

### 2. TabButton 的 HoverStyle 不自动生效

TabButton 的鼠标事件由 TabBar 代理，框架的 HoverStyle 自动切换机制不生效。TabBar 必须手动读取 `tab->HoverStyle.BackColor` 来设置 hover 颜色。

### 3. 激活标签的特殊处理

TabButton 激活时统一使用白色背景，kind 颜色只影响非激活状态。

### 4. 修改全局颜色后需要重新应用

修改 `g_controlKindColors` 后，已存在的控件需要调用 `SetKind(GetKind())` 来刷新颜色。
