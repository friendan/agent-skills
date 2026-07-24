---
name: ezui-tooltip
description: 指导如何使用EZUI框架的Tooltip提示功能，包括htm内存布局、自适应宽度、圆角窗口、显示位置、定时隐藏等。
---

# EZUI Tooltip 提示实现

## 概述

Tooltip 使用 `LayeredWindow`（分层窗口）实现，是一个独立无焦点小窗口。内部通过内存 htm 布局加载控件，利用框架的 `kind` 色调体系、`width="auto"` 自适应宽度，以及 `SetAutoWidth` + `RefreshLayout` 实现窗口大小自适应。

## 完整 API

### C++ 用法

```cpp
// ===== 基础设置（在 Control 基类上） =====
btn->SetToolTip(L"提示文本");                              // 设文字
btn->GetToolTip().Kind = L"success";                       // 设颜色
btn->GetToolTip().AutoHideDelayMs = 3000;                  // 3秒后自动隐藏
btn->GetToolTip().MaxWidth = 500;                          // 最大宽度
btn->GetToolTip().Offset = { 20, 20 };                     // 鼠标偏移
btn->GetToolTip().ExtraStyle = L"border-radius: 10px;";    // 额外样式

// 一次性设置全部
TooltipData& td = btn->GetToolTip();
td.Text = L"保存成功";
td.Kind = L"success";
td.AutoHideDelayMs = 3000;
```

### htm 属性

```html
<!-- 基础用法 -->
<button tooltip="保存成功"></button>
<button tooltip="危险" tooltip-kind="danger"></button>

<!-- 显示位置 -->
<button tooltip="右侧显示" tooltip-placement="right"></button>
<button tooltip="上方显示" tooltip-placement="top"></button>
<button tooltip="下方显示" tooltip-placement="bottom"></button>
<button tooltip="左侧显示" tooltip-placement="left"></button>

<!-- 自动隐藏 -->
<button tooltip="3秒后消失" tooltip-hide="3000"></button>

<!-- 最大宽度 -->
<button tooltip="长文本" tooltip-maxwidth="200"></button>

<!-- 显示延迟 -->
<button tooltip="1秒后才显示" tooltip-delay="1000"></button>

<!-- 自定义圆角 -->
<button tooltip="大圆角" tooltip-style="border-radius: 12px;"></button>

<!-- 组合使用 -->
<button tooltip="保存成功" tooltip-kind="success" tooltip-placement="bottom" tooltip-hide="3000"></button>
```

| htm 属性 | 说明 | 可选值 |
|----------|------|-------|
| `tooltip` | 提示文字 | 任意文本 |
| `tooltip-kind` | 颜色主题 | success/danger/warning/info/primary/secondary/dark/light |
| `tooltip-placement` | 显示位置 | mouse（默认）/top/bottom/left/right |
| `tooltip-hide` | 自动隐藏时间（毫秒） | 数字，0=不自动隐藏 |
| `tooltip-maxwidth` | 最大宽度（像素） | 数字 |
| `tooltip-delay` | 显示延迟（毫秒） | 数字，默认500 |
| `tooltip-style` | 额外内联样式 | CSS 样式字符串 |

### 全局 API

```cpp
// Set tooltip for a control
Tooltip::Set(ctl, L"文本", L"success");
Tooltip::Set(ctl, opts);                            // 用 TooltipOptions

// Remove
Tooltip::Remove(ctl);

// Show/Hide
Tooltip::Show(opts, text, screenX, screenY, owner);
Tooltip::Hide();

// Clear all
Tooltip::ClearAll();
```

### 各风格默认自动隐藏时间

| 方法 | 默认 auto-hide |
|------|---------------|
| 默认 | 0（不自动隐藏，鼠标离开时隐藏） |

传 `AutoHideDelayMs > 0` 启用自动隐藏：`td.AutoHideDelayMs = 3000;`

## 架构

```
鼠标进入控件
  → Control::OnMouseEnter
    → 启动 Timer（延迟 500ms）
    → Timer 触发 → Tooltip::Show(opts, text, screenX, screenY, owner)
    
鼠标离开控件
  → Control::OnMouseLeave
    → 停止 Timer
    → Tooltip::Hide()

Tooltip::Show 内部：
  → 如果已显示同个 tooltip，只 reposition 不重建
  → DestroyCurrentTooltip()           // 隐藏并异步销毁旧的
  → BuildTooltipHtm(kind, text, maxWidth, extraStyle)
  → new TooltipPopup(600, 30, ...)    // 大窗口让布局不受限
    → LoadXml + SetupUI
  → root->Style.Border.Radius = 6     // 设置圆角（和普通控件一样）
  → root->Style.Border.Color = bgColor
  → root->SetAutoWidth(true)
  → root->RefreshLayout()             // 强制计算
  → root->Width() 获取实际宽度
  → SetRect 调整窗口到实际大小
  → SetWindowRgn 设置圆角窗口区域
  → Show()
  → StartAutoHideTimer (如果配置了)
```

和 Toast 的最大区别：**由 Control 基类的 `OnMouseEnter`/`OnMouseLeave` 驱动**，不依赖 Window 的 `WM_MOUSEHOVER`。

## HTM 布局

```cpp
static std::string BuildTooltipHtm(const std::string& kindName, const std::string& escapedText,
    int maxWidth, const std::string& extraStyle) {
    char b[2048];
    std::string htmStyle = extraStyle.empty() ? "border-radius: 6px;" : extraStyle;
    snprintf(b, sizeof(b),
        R"(<hlayout dock="fill" kind="%s" style="%s">
  <spacer width="8"></spacer>
  <label text="%s" font-size="12" halign="left" width="auto" max-width="%d"></label>
  <spacer width="8"></spacer>
</hlayout>)", kindName.c_str(), htmStyle.c_str(), escapedText.c_str(), maxWidth);
    return b;
}
```

关键点：
- **hlayout**：`kind="%s"` 提供全局色调（与按钮同款颜色），`dock="fill"` 撑满 IFrame
- **spacer**：左右各 8px 边距，避免文字贴边
- **label**：`width="auto"` 自适应文字宽度，`halign="left"` 左对齐，`max-width` 限制最大宽度
- **style**：默认 `border-radius: 6px;`，可通过 `extraStyle` 覆盖

## 窗口样式

```cpp
LayeredWindow(w, h, owner, WS_POPUP,
    WS_EX_NOACTIVATE | WS_EX_LAYERED | WS_EX_TOOLWINDOW | WS_EX_TRANSPARENT)
```

- `WS_EX_NOACTIVATE` — 点击时不获取焦点
- `WS_EX_LAYERED` — 分层窗口，支持透明
- `WS_EX_TOOLWINDOW` — 不在任务栏显示
- `WS_EX_TRANSPARENT` — 鼠标穿透，避免 tooltip 盖在控件上导致闪烁
- `SetShadow(0)` — 关闭阴影边框，消除白边

## Control 基类集成

Tooltip 逻辑内置在 `Control` 基类中，所有控件自动获得 tooltip 功能：

```cpp
// Control.h — 成员变量
TooltipData m_tooltip;        // Tooltip 配置
Timer* m_tooltipTimer;        // 延迟显示定时器

// Control.cpp — OnMouseEnter
void Control::OnMouseEnter(const MouseEventArgs& args) {
    this->Invalidate();
    if (!m_tooltip.Text.empty() && m_hWnd) {
        if (!m_tooltipTimer) {
            m_tooltipTimer = new Timer;
            m_tooltipTimer->Interval = 0;
            m_tooltipTimer->Tick = [this](Timer* t) {
                t->Stop();
                Sleep(m_tooltip.ShowDelayMs > 0 ? m_tooltip.ShowDelayMs : 500);
                BeginInvoke([this]() {
                    if (!m_tooltip.Text.empty()) {
                        POINT pt; ::GetCursorPos(&pt);
                        IFrame* f = GetFrame();
                        HWND owner = f ? f->Hwnd() : m_hWnd;
                        TooltipOptions opts;
                        opts.Kind = m_tooltip.Kind;
                        opts.AutoHideDelayMs = m_tooltip.AutoHideDelayMs;
                        opts.MaxWidth = m_tooltip.MaxWidth;
                        opts.Offset = m_tooltip.Offset;
                        opts.ExtraStyle = m_tooltip.ExtraStyle;
                        Tooltip::Show(opts, m_tooltip.Text, pt.x, pt.y, owner);
                    }
                });
            };
        }
        m_tooltipTimer->Start();
    }
}

// Control.cpp — OnMouseLeave
void Control::OnMouseLeave(const MouseEventArgs& args) {
    this->Invalidate();
    if (m_tooltipTimer) m_tooltipTimer->Stop();
    if (!m_tooltip.Text.empty()) Tooltip::Hide();
}
```

## 宽度自适应核心代码

```cpp
g_tooltipPopup = new TooltipPopup(600, 30, owner, htm);

int popupW = 280, popupH = 30;
auto& children = g_tooltipPopup->GetFrame()->GetControls();
if (!children.empty()) {
    auto* root = *children.begin();
    if (root) {
        // 设置圆角边框（和普通控件一样）
        root->Style.Border.Radius = 6;
        root->Style.Border.Color = root->GetBackColor();
        root->Style.Border.Left = root->Style.Border.Top =
        root->Style.Border.Right = root->Style.Border.Bottom = 1;

        root->SetAutoWidth(true);
        root->SetAutoHeight(true);
        root->RefreshLayout();
        int rw = root->Width();
        int rh = root->Height();
        if (rw > 20 && rh > 10) {
            popupW = min(rw, opts.MaxWidth + 16);
            popupH = rh;
        }
    }
}

g_tooltipPopup->SetRect(Rect(x, y, popupW, popupH));
g_tooltipPopup->Show();
```

**重要**：初始窗口要够大（如 600x30），让布局计算不受容器宽度限制。之后 resize 到实际内容大小。

## 圆角实现

EZUI 中 `border-radius` 只影响边框线的绘制（`OnBorderPaint`），背景始终是 `FillRectangle` 填满整个矩形。要同时让背景有圆角，需要：

1. 设置 `root->Style.Border.Radius = 6` — 让边框线绘制时带圆角
2. 设置 `root->Style.Border.Color = root->GetBackColor()` — 让边框线和背景色一致（视觉上就像背景也有圆角）
3. 设置 `border-width = 1` — 边框线宽度 1px

这样圆角效果和框架中按钮等控件的圆角效果完全一致，没有 `SetWindowRgn` 的锯齿问题。

## 控件数据结构

```cpp
// Control.h — TooltipData 结构
struct UI_EXPORT TooltipData {
    UIString Text;
    UIString Kind = L"secondary";
    UIString ExtraStyle;          // 额外的 htm style
    int ShowDelayMs = 500;
    int AutoHideDelayMs = 0;      // 0 = 不自动隐藏
    int MaxWidth = 400;
    Point Offset = { 12, 20 };
};

// Tooltip.h — TooltipOptions（同字段）
struct UI_EXPORT TooltipOptions {
    UIString Kind = L"secondary";
    UIString ExtraStyle;
    int ShowDelayMs = 500;
    int AutoHideDelayMs = 0;
    int MaxWidth = 400;
    Point Offset = { 12, 20 };
};
```

## 异步销毁

和 Toast 完全相同的模式——不在 WndProc 中销毁窗口：

```cpp
// TooltipPopup::WndProc(WM_CLOSE) 中：
::ShowWindow(Hwnd(), SW_HIDE);
::PostMessage(__EzUI_MessageWnd, WM_GUI_SYSTEM, WM_GUI_TOOLTIP_CLEANUP, (LPARAM)this);
return 0;

// Application.cpp 消息窗口处理：
else if (wParam == WM_GUI_TOOLTIP_CLEANUP) {
    HWND h = ((Window*)lParam)->Hwnd();
    if (h && ::IsWindow(h)) ::DestroyWindow(h);
    delete (Window*)lParam;
}
```

## 关联代码

| 文件 | 内容 |
|------|------|
| `src/sources/Tooltip.cpp` | `TooltipPopup` 类、`Tooltip` 静态 API、`BuildTooltipHtm` |
| `src/include/EzUI/Tooltip.h` | `TooltipOptions`、`Tooltip` 类声明 |
| `src/sources/Control.cpp` | `OnMouseEnter`/`OnMouseLeave` 集成 tooltip |
| `src/include/EzUI/Control.h` | `TooltipData`、`m_tooltip`、`m_tooltipTimer` |
| `src/include/EzUI/UIDef.h` | `WM_GUI_TOOLTIP_CLEANUP` 宏定义 |
| `src/sources/Application.cpp` | 消息窗口处理 `WM_GUI_TOOLTIP_CLEANUP` |
| `src/sources/Window.cpp` | `Tooltip::Hide()` 调用（控件切换时） |

## 踩坑记录

### ❗坑1：WM_NCDESTROY 中 delete this 崩溃（同 Toast）

**现象**：关闭 Tooltip 时闪退。

**原因和解决方案**：和 Toast 完全相同——不能在 WndProc 中 `DestroyWindow` + `delete this`，必须 `PostMessage` 到 `__EzUI_MessageWnd` 异步执行。详情见 Toast 坑8。

### ❗坑2：Tooltip 闪烁

**现象**：鼠标移到按钮上，Tooltip 出现后闪烁不停。

**原因**：Tooltip 窗口盖在按钮上，鼠标移到 Tooltip 上触发按钮的 `OnMouseLeave` → `Tooltip::Hide()`，Tooltip 消失后鼠标回到按钮 → 再次触发进入 → 再次显示。

**解决方案**：
1. **`WS_EX_TRANSPARENT`**（主要）：让鼠标穿透 Tooltip，鼠标事件直接到主窗口
2. **Tooltip::Show 中的保护**：如果已显示未销毁，只 reposition 不重建

### ❗坑3：宽度不自适应

**现象**：Tooltip 宽度固定，文字多时显示不全，文字少时太宽。

**尝试过的错误方案**：

| 方案 | 结果 |
|------|------|
| label 设 `width="auto"`，窗口固定 280px | 长内容显示不全 |
| GDI 计算文本像素宽度 | 和 EZUI 布局的文本宽度不一致 |
| `GetLayout()->Width()` | 返回的是容器宽度，不是内容宽度 |
| `FindControl` 找 label 拿 Width | 容器宽度限制导致 label 宽度计算错误 |

**最终解决方案**：
1. 创建足够大的初始窗口（600x30），让布局不受容器宽度限制
2. 对 root 设 `SetAutoWidth(true)` + `RefreshLayout()`
3. `root->Width()` 获取自适应后的实际宽度，resize 窗口

### ❗坑4：圆角不生效

**现象**：border-radius 设了但看不到圆角。

**原因**：EZUI 中 `border-radius` 只影响边框线的绘制（`OnBorderPaint`），而边框线的绘制需要 `border.Color != 0` 且 `border.Left/Top/Right/Bottom > 0`。只设 `style="border-radius: 6px"` 时颜色为透明、宽度为 0，边框线不会绘制，圆角没有视觉效果。

**解决**：在 C++ 中设：
```cpp
root->Style.Border.Radius = 6;
root->Style.Border.Color = root->GetBackColor();  // 和背景色一致
root->Style.Border.Left = root->Style.Border.Top =
root->Style.Border.Right = root->Style.Border.Bottom = 1;
```

**错误方案**：用 `SetWindowRgn` 做窗口级别圆角——锯齿严重。

### 坑5：右侧白色方块

**现象**：Tooltip 窗口右侧或底部有白色区域。

**解决**：`SetRect` 触发 `LayeredWindow::OnSize` 重新创建位图，同时 `SetShadow(0)` 关闭阴影边框。

### 坑6：多种触发方案的选择

最初在 `Window::OnMouseHover` 中触发，但逻辑复杂（需要处理同一个控件 auto-hide 后不重复触发的问题）。最终改为 **`Control::OnMouseEnter`/`OnMouseLeave` 驱动**，每个控件天然自带 tooltip，鼠标离开控件自动取消延迟和隐藏，无需额外状态管理。

### 坑7：Timer 子线程访问 UI

`Timer` 在子线程运行，需要在 Tick 中用 `BeginInvoke` 投递到主线程执行 UI 操作（`Tooltip::Show`）。

### 坑8：`TooltipOptions` 和 `TooltipData` 分离

两个结构体字段相同，用途不同：`TooltipData` 是 Control 成员的配置，`TooltipOptions` 是 `Tooltip::Show` 的参数。`Control::OnMouseEnter` 中需要把 `TooltipData` 转换成 `TooltipOptions` 再传给 `Tooltip::Show`。
