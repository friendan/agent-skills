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
void SetOutline(bool outline);      // 轮廓模式（透明背景 + 彩色边框，hover填充）
bool IsOutline();                   // 是否轮廓模式
void SetUrl(const UIString& url);   // 设置关联URL
UIString GetUrl();                  // 获取关联URL
```

### 全局辅助函数

```cpp
void ApplyControlKindColors(Control* ctl, ControlKind kind);
```

### HTM 属性

所有控件（`<button>`、`<label>`、`<tabbutton>` 等）都支持以下属性：

```html
<button kind="primary"></button>
<label kind="danger"></label>
<tabbutton kind="info"></tabbutton>
<!-- 轮廓按钮 -->
<button kind="primary" outline="true"></button>
<!-- 链接按钮 -->
<button kind="link" url="https://www.baidu.com"></button>
```

| 属性 | 说明 |
|------|------|
| `kind` | 风格类型（primary/secondary/success/danger/warning/info/light/dark/link） |
| `outline` | `"true"` 或 `"1"` 启用轮廓模式 |
| `url` | 关联URL，`kind="link"` 时点击用默认浏览器打开 |

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

## 轮廓模式（Outline）

轮廓模式对应 Bootstrap 的 `btn-outline-*` 系列，效果：
- **正常状态**：透明背景 + kind 色边框 + kind 色文字
- **hover 状态**：填充背景色 + 白色文字
- **active 状态**：更深背景色 + 白色文字

```cpp
btn->SetKind(ControlKind::Primary);
btn->SetOutline(true);
```

HTM：

```html
<button kind="primary" outline="true"></button>
```

特殊处理：`Light` 在白色背景上看不清，outline 时自动改用 Dark 颜色（#212529）代替。

## 链接模式（Link）

`kind="link"` 的控件特点：
- 透明背景、蓝色文字、蓝色边框
- 自动设置 `Cursor::HAND`（手型光标）
- 如果设置了 `url` 属性，点击后调用 `ShellExecuteW` 用默认浏览器打开

```html
<button kind="link" url="https://www.baidu.com">百度</button>
<label kind="link" url="https://github.com">GitHub</label>
```

## 禁用状态（Disabled）

控件禁用后：鼠标事件被拦截、显示灰色样式、光标变为默认箭头。

### HTM 属性

```html
<button kind="primary" disabled="true">禁用</button>
<button kind="primary" enable="false">等效写法</button>
```

| 属性 | 说明 |
|------|------|
| `disabled="true"` | 禁用一个控件 |
| `disabled="false"` | 启用一个控件 |
| `enable="true"` | 启用（等效） |
| `enable="false"` | 禁用（等效） |

### C++ 接口

```cpp
btn->SetEnabled(false);  // 禁用
btn->SetEnabled(true);   // 启用
btn->IsEnabled();        // 判断是否启用
```

### 禁用样式

`SetEnabled(false)` 自动设置 `DisabledStyle` 为 Bootstrap 禁用样式：
- 背景色 `#e9ecef`、边框 `#dee2e6`、文字 `#adb5bd`
- 光标设为默认箭头（`Cursor::ARROW`，禁用 link 等的手型光标）
- 启用时恢复 link 的手型光标

## 关键注意事项

### 1. ControlKind 颜色会被手动覆盖

如果手动设置 `Style.BackColor`，会覆盖 `ControlKind` 设置的颜色。需要在手动设色的地方加 `GetKind()` 判断。

### 2. TabButton 的 HoverStyle 不自动生效

TabButton 的鼠标事件由 TabBar 代理，框架的 HoverStyle 自动切换机制不生效。TabBar 必须手动读取 `tab->HoverStyle.BackColor` 来设置 hover 颜色。

### 3. 激活标签的特殊处理

TabButton 激活时统一使用白色背景，kind 颜色只影响非激活状态。

### 4. 修改全局颜色后需要重新应用

修改 `g_controlKindColors` 后，已存在的控件需要调用 `SetKind(GetKind())` 来刷新颜色。

### 5. SetKind 和 SetOutline 的顺序

先调用 `SetKind` 再调用 `SetOutline`，或者先 `SetOutline` 再 `SetKind` 都可以正常工作。`SetKind` 会保留 outline 状态重新应用颜色。

## 踩坑记录

### 坑1：DisabledStyle 不能在 SetKind/SetOutline 中设置

最初在 `SetKind` 和 `SetOutline` 中强制设置了 `DisabledStyle`，导致所有带 `kind` 的控件（包括角标测试行的 label）都受到影响，即使它们并没有被禁用。

**正确做法**：只在 `SetEnabled(false)` 中设置 `DisabledStyle`，且只对 `kind != Default` 的控件设置。

### 坑2：LoadCursor 名称冲突

EzUI.h 中有 `#undef LoadCursor`，不能用 Win32 的 `IDC_ARROW` 宏。必须用 `ezui::Cursor::ARROW`（即 `LoadCursor(Cursor::ARROW)`）。

```cpp
// 错误
DisabledStyle.Cursor = LoadCursor(IDC_ARROW);

// 正确
DisabledStyle.Cursor = LoadCursor(Cursor::ARROW);
```

### 坑3：htm 中无值属性的处理

htm 中的 `disabled`（无值属性）被 TiXml 解析时 `attr->Value()` 返回空字符串，导致 `SetAttribute("disabled", "")` 没有匹配任何条件，行为不可预期。

**解决方法**：htm 属性必须指定值才能生效，如 `disabled="true"` 或 `disabled="false"`。无值写法 `disabled` 无效。
