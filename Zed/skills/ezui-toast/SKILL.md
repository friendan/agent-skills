---
name: ezui-toast
description: 指导如何在EZUI框架中实现Toast通知功能，包括LayeredWindow方案、htm布局、kind色调、关闭按钮、拖动、堆叠定位等。
---

# EZUI Toast 通知实现

## 概述

Toast 通知使用 `LayeredWindow`（分层窗口）实现，每个 Toast 是一个独立的无焦点小窗口。内部通过 htm 布局字符串加载控件，利用框架现有的 `kind` 色调体系和 `action` 行为机制。

## 架构

```
Toast::Show("保存成功")
  → new ToastPopup (LayeredWindow)
    → 拼接 htm 布局字符串
    → UIManager::LoadXml + SetupUI 加载布局
    → 设置窗口位置（右下角堆叠）
    → Show()
```

每个 Toast 是一个独立的分层窗口，不依赖于主窗口的控件树，避免了子控件在自定义容器中不被绘制的问题。

## 完整 API

```cpp
// 最简用法
Toast::Show("保存成功");              // 默认 Info 风格

// 便捷方法（自动带对应 kind）
Toast::ShowSuccess("操作已完成");
Toast::ShowDanger("删除失败");
Toast::ShowWarning("网络不稳定");
Toast::ShowInfo("你有新消息");

// 带标题和正文
Toast::Show("文件保存成功", "已保存到 D:\\report.pdf");

// 完整配置（通过 ToastOptions）
Toast::Show("操作失败", ToastOptions()
    .Kind(ControlKind::Danger)
    .Duration(3000)
    .ShowClose(true)
    .MinWidth(400)
    .Align(ToastAlign::TopRight)
);
```

## ToastOptions

| 方法 | 说明 | 默认值 |
|------|------|--------|
| `Title(text)` | 标题文本 | 同 `text` 参数 |
| `Text(text)` | 正文文本 | 空 |
| `Kind(kind)` | 色调（Success/Danger/Warning/Info 等） | `Info` |
| `Icon(lib, name)` | SVG 图标 | 无 |
| `Duration(ms)` | 自动关闭毫秒数，0=不自动关闭 | 4000 |
| `ShowClose(bool)` | 是否显示关闭按钮 | true |
| `Animation(anim)` | 动画类型 | None |
| `Align(align)` | 对齐位置 | BottomRight |
| `Target(control)` | 相对于某个控件定位 | 无（相对主窗口） |
| `MinWidth(w)` | 最小宽度 | 350 |
| `MaxWidth(w)` | 最大宽度 | 600 |

## HTM 布局

Toast 内部使用 htm 字符串布局，关键结构：

```html
<vbox id="root" dock="fill" style="background-color: rgb(255,255,255); border: 1px solid rgba(0,0,0,0.12); border-radius: 8px;">
    <hlayout id="toast-header" kind="success" dock="fill" action="title" style="padding-left: 14px; padding-right: 6px; border-radius: 8px;">
        <spacer width="6"></spacer>
        <label id="toastText" text="..." font-size="13" style="font-weight: bold;" action="title" valign="middle"></label>
        <spacer width="star"></spacer>
        <label id="toastCloseBtn" text="" width="22" height="22" bsicon="x-lg" icon-size="12" action="close" style="cursor: hand; border-radius: 4px;" valign="middle" halign="center"></label>
    </hlayout>
</vbox>
```

关键点：
- **外部 vbox**：白色背景 + 边框 + 圆角
- **header（hlayout）**：`kind="xxx"` 提供全局色调，`action="title"` 支持拖动
- **文字标签**：`action="title"` 使文字区域也可拖动
- **关闭按钮**：`bsicon="x-lg"` 显示 SVG 关闭图标，`action="close"` 关闭窗口

## 关闭按钮 Hover 效果

加载 htm 后通过 C++ 设置 HoverStyle：

```cpp
auto* closeBtn = FindControl(L"toastCloseBtn");
if (closeBtn) {
    closeBtn->HoverStyle.BackColor = Color(220, 50, 50, 200); // 红色背景
    closeBtn->HoverStyle.ForeColor = Color(255, 255, 255);   // 白色图标
}
```

## 窗口样式

```cpp
LayeredWindow(w, h, owner, WS_POPUP,
    WS_EX_NOACTIVATE | WS_EX_LAYERED | WS_EX_TOOLWINDOW)
```

- `WS_EX_NOACTIVATE` — 点击时不获取焦点，不抢主窗口焦点
- `WS_EX_LAYERED` — 分层窗口，支持透明
- `WS_EX_TOOLWINDOW` — 不在任务栏显示
- **不带** `WS_EX_TRANSPARENT` — 否则鼠标事件穿透，关闭按钮无法点击

## 堆叠定位

```cpp
if (owner) {
    RECT r;
    GetWindowRect(owner, &r);
    int x = r.right - w - 16;
    int y = r.bottom - h - 16 - index * (h + 8);
    popup->SetRect(Rect(x, y, w, h));
}
```

每次新 Toast 往上堆叠，间距 8px。

## 实现步骤（新增 SVG 图标库流程对比）

1. 创建 `ToastPopup` 继承 `LayeredWindow`
2. 构造函数中拼接 htm 布局字符串
3. 通过 `GetFrame()->GetUIManager()` 加载 htm
4. 在 `OnPaint` 中**不要画** LayeredWindow 边缘的内容（避免黑边）
5. `Toast::Show` 静态方法创建 `ToastPopup` 并显示

## 踩坑记录

### 坑1：子控件在自定义容器中不被绘制

最早尝试把 Toast 作为 Control 添加到 Window 的 `m_toastLayer`（一个普通 Control），但子控件（Label）完全不显示。

**原因**：`m_toastLayer` 是一个独立的 Control，没有加到控件树中，虽然 `Window::OnPaint` 中调用了 `SendEvent(m_toastLayer, arg)`，但子控件的递归绘制链可能因为缺少 Parent 或 Frame 引用而断裂。即使手动调用了 `OnChildPaint`，子控件的 `OnPaintBefore` 中的裁剪计算也可能出错。

**解决方案**：改用独立窗口（LayeredWindow），每个 Toast 是一个独立的窗口，自己的控件树完整。

### 坑2：Label 子控件不显示文字

即使把 Toast 加到 IFrame 中，Label 子控件也不显示文字。

**原因**：Label 依赖框架的布局和绘制链。在 `Toast::OnPaint` 中如果重写了 `OnPaint` 但没有调用 `Control::OnPaint(args)`，子控件的 `OnChildPaint` 虽然会被 `OnPaintBefore` 调用，但 `OnBackgroundPaint` 和 `OnForePaint` 不会被调用，导致 Label 的文字绘制缺失。

**解决方案**：用 htm 布局加载控件，让框架完整处理子控件的创建和绘制链。

### 坑3：`WS_EX_TRANSPARENT` 导致鼠标事件穿透

最初的 LayeredWindow 加了 `WS_EX_TRANSPARENT` 样式，导致关闭按钮点击无反应。

**原因**：`WS_EX_TRANSPARENT` 让窗口透明，鼠标事件穿透到下层窗口。

**解决方案**：去掉 `WS_EX_TRANSPARENT`，改为只加 `WS_EX_NOACTIVATE`（不抢焦点）。

### 坑4：LayeredWindow 边缘出现黑边

在 `OnPaint` 中绘制多层阴影边框（`DrawRectangle` 在窗口边缘外），拖动窗口后边缘出现黑线。

**原因**：LayeredWindow 只绘制客户区内的内容，阴影边框的抗锯齿像素在窗口边缘外显示为黑边。

**解决方案**：不在 `OnPaint` 中画任何内容，完全由 htm 中的 vbox 样式的 `border` 属性提供边框。

### 坑5：action="close" 在 LayeredWindow 中无效

直接用 `action="close"` 的 Label 无法关闭窗口。

**原因**：LayeredWindow 的事件处理链可能没有正确传递 `Close` action。实际上经过测试，去掉 `WS_EX_TRANSPARENT` 后 `action="close"` 可以正常工作。

**解决方案**：确保窗口不带 `WS_EX_TRANSPARENT`，Label 使用 `action="close"` 即可。

### 坑6：C++ lambda 回调中 Timer 线程安全问题

如果使用 `Timer` 做自动关闭，`Timer::Tick` 在子线程中执行，直接调用 UI 操作（如 `SetOpacity`、`Dismiss`）会导致崩溃。

**解决方案**：使用 `BeginInvoke()` 或 `PostMessage` 将 UI 操作投递到主线程执行。或者直接不用 Timer，让用户手动关闭。

### 坑7：UIText 拼接 operator+ 问题

`UIString`（即 `ui_text::String`）没有重载 `operator+`，不能直接用 `str1 + str2`。

**解决方案**：用 `.unicode()` 转 `std::wstring` 拼接，再用 `UIString(wstr)` 构造。

### 坑8：htm 布局文本需要转义

动态拼接 htm 时，文本内容中的 `& < > "` 需要转义为实体，否则 htm 解析会出错或被截断。

**解决方案**：手动 replace 转义：
```cpp
text.replace(L"&", L"&amp;");
text.replace(L"<", L"&lt;");
text.replace(L">", L"&gt;");
text.replace(L"\"", L"&quot;");
```

### 坑9：LayeredWindow 的 htm 布局要用 UIManager 加载

`LayeredWindow` 没有 `LoadXmlFromString` 方法，需要手动通过 `GetFrame()->GetUIManager()` 加载：

```cpp
auto* umg = GetFrame()->GetUIManager();
std::string htmUtf8;
UnicodeToUTF8(htmWString, &htmUtf8);
umg->LoadXml(htmUtf8.c_str(), htmUtf8.size());
umg->SetupUI(GetFrame());
```
