# EZUI Toast 通知实现

## 概述

Toast 通知使用 `LayeredWindow`（分层窗口）实现，每个 Toast 是一个独立的无焦点小窗口。内部通过 htm 布局字符串加载控件，利用框架现有的 `kind` 色调体系和 `action` 行为机制。

## 完整 API

```cpp
// ===== 各风格快捷方法 =====
Toast::ShowSuccess(L"操作成功");          // 默认 3s
Toast::ShowDanger(L"删除失败");           // 默认 5s
Toast::ShowWarning(L"网络不稳定");        // 默认 4s
Toast::ShowInfo(L"你有新消息");           // 默认 3s

// ===== 自定义自动关闭时间（传 0 禁止自动关闭） =====
Toast::ShowSuccess(L"操作成功", 2000);
Toast::ShowDanger(L"删除失败", 0);        // 不自动关闭

// ===== 通用 Toast（通过 ToastOptions） =====
Toast::Show(L"保存成功", ToastOptions().Kind(ControlKind::Primary).Duration(3000));

// ===== 关闭所有 =====
Toast::DismissAll();
```

## 显示位置

```cpp
// 屏幕位置
ToastAlign::ScreenTopLeft
ToastAlign::ScreenTopCenter
ToastAlign::ScreenTopRight
ToastAlign::ScreenBottomLeft
ToastAlign::ScreenBottomCenter
ToastAlign::ScreenBottomRight
ToastAlign::ScreenCenter          // 屏幕居中（不堆叠）

// 窗口位置
ToastAlign::WindowTopLeft
ToastAlign::WindowTopCenter
ToastAlign::WindowTopRight
ToastAlign::WindowBottomLeft
ToastAlign::WindowBottomCenter
ToastAlign::WindowBottomRight
ToastAlign::WindowCenter          // 窗口居中（不堆叠）
```

示例：
```cpp
Toast::Show(L"左上角警告", ToastOptions()
    .Kind(ControlKind::Warning)
    .Duration(4000)
    .Align(ToastAlign::ScreenTopLeft));

Toast::Show(L"屏幕居中", ToastOptions()
    .Kind(ControlKind::Primary)
    .Duration(0)                  // 不自动关闭
    .Align(ToastAlign::ScreenCenter));
```

## 架构

```
Toast::ShowSuccess("操作成功")
  → CreateToast(ControlKind::Success, text, 3000, align)
    → BuildHtm("success", escapedText)      // snprintf 拼接 htm
    → new ToastPopup (LayeredWindow)
      → UIManager::LoadXml + SetupUI 加载布局
      → SetupCloseBtn(popup)                // 关闭按钮 hover
    → ShowToast(popup, owner, align)        // 定位 + 堆叠 + Show
    → popup->StartTimer(3000)               // 启动倒计时
```

每个 Toast 是独立的分层窗口，不依赖主窗口控件树。

## HTM 布局

```cpp
static std::string BuildHtm(const char* kn, const std::string& tu) {
    char b[2048];
    snprintf(b, sizeof(b),
        R"(<vbox id="root" dock="fill" style="background-color: rgb(255,255,255); border: 1px solid rgba(0,0,0,0.12); border-radius: 8px;">
  <hlayout id="toast-header" kind="%s" dock="fill" action="title" style="padding-left: 14px; padding-right: 6px; border-radius: 8px;">
    <spacer width="6"></spacer>
    <label text="%s" font-size="13" style="font-weight: bold;" action="title" valign="middle"></label>
    <spacer width="star"></spacer>
    <label id="toastTimer" text="" width="30" font-size="11" style="color: rgba(255,255,255,0.7);" valign="middle" halign="center"></label>
    <label id="toastCloseBtn" text="" width="22" height="22" bsicon="x-lg" icon-size="12" action="close" style="cursor: hand; border-radius: 4px;" valign="middle" halign="center"></label>
  </hlayout>
</vbox>)", kn, tu.c_str());
    return b;
}
```

关键点：
- **外部 vbox**：白色背景 + 1px 半透明边框 + 8px 圆角
- **header（hlayout）**：`kind="xxx"` 提供色调，`action="title"` 支持拖动
- **文字标签**：`action="title"` 使文字区域也可拖动
- **倒计时标签**（`toastTimer`）：显示 `3s` `2s` `1s`，每秒更新
- **关闭按钮**：`bsicon="x-lg"` SVG 关闭图标，`action="close"`

## 窗口样式

```cpp
LayeredWindow(w, h, owner, WS_POPUP,
    WS_EX_NOACTIVATE | WS_EX_LAYERED | WS_EX_TOOLWINDOW)
```

- `WS_EX_NOACTIVATE` — 点击时不获取焦点
- `WS_EX_LAYERED` — 分层窗口，支持透明
- `WS_EX_TOOLWINDOW` — 不在任务栏显示
- **不要加** `WS_EX_TRANSPARENT` — 否则鼠标事件穿透

## 关闭按钮 Hover

```cpp
static void SetupCloseBtn(ToastPopup* popup) {
    auto* btn = popup->FindControl(L"toastCloseBtn");
    if (btn) {
        btn->HoverStyle.BackColor = Color(0, 0, 0, 50);    // 半透明黑
        btn->HoverStyle.ForeColor = Color(255, 255, 255);  // 白色图标
    }
}
```

统一使用半透明黑 + 白色图标，在任何颜色背景上都能看到变化。

## 堆叠定位

```cpp
static void ShowToast(ToastPopup* popup, HWND owner, ToastAlign align) {
    CleanupClosedToasts();
    int index = (int)g_activeToasts.size();
    g_activeToasts.push_back(popup->Hwnd());
    // 根据 align 计算 x, y，堆叠间距 8px
    ...
    popup->Show();
}
```

- 每次新 Toast 按 `index` 堆叠，间距 8px
- 超出屏幕时限制 `maxIndex`
- `CleanupClosedToasts` 清理已销毁窗口的 HWND

## 倒计时实现

```cpp
void StartTimer(int ms) {
    m_duration = ms;
    m_remaining = ms;
    if (m_duration <= 0) { /* 隐藏倒计时标签 */ return; }
    UpdateTimerLabel();
    SetTimer(Hwnd(), (UINT_PTR)this, 1000, nullptr);
}

void OnTimer() {
    m_remaining -= 1000;
    if (m_remaining <= 0) {
        m_dismissed = true;
        StopTimer();
        CloseToastAsync();  // 隐藏 + PostMessage 异步销毁
    } else {
        UpdateTimerLabel();
        Invalidate();
    }
}
```

- 使用 Win32 `SetTimer`（主线程，不需要 `BeginInvoke`）
- `Invalidate()` 确保 LayeredWindow 重绘更新文字

## 异步销毁机制（核心）

Toast 的 `DestroyWindow` 和 `delete` **不能在 `WndProc` 中执行**，必须异步到 `__EzUI_MessageWnd` 的消息处理中执行：

```cpp
// ToastPopup::WndProc(WM_CLOSE) 或 OnTimer 中：
::ShowWindow(Hwnd(), SW_HIDE);
::PostMessage(__EzUI_MessageWnd, WM_GUI_SYSTEM,
    WM_GUI_TOAST_CLEANUP, (LPARAM)this);
return 0;  // 立即返回，不让基类继续处理
```

```cpp
// Application.cpp 消息窗口的处理：
else if (wParam == WM_GUI_TOAST_CLEANUP) {
    HWND h = ((Window*)lParam)->Hwnd();
    if (h && ::IsWindow(h)) {
        ::DestroyWindow(h);     // 走完整 DefWindowProc
    }
    delete (Window*)lParam;     // 释放 C++ 对象
}
```

关闭方式（全部走异步）：

| 触发方式 | 操作 |
|---------|------|
| 倒计时到 0 | `OnTimer` → `ShowWindow(SW_HIDE)` + `PostMessage` |
| 点关闭按钮 | `action="close"` → `WM_CLOSE` → 同上 |
| `DismissAll` | `PostMessage(WM_CLOSE)` 异步发给所有 Toast |

`DismissAll` 也用 `PostMessage` 而非 `SendMessage`，避免同步递归：

```cpp
void Toast::DismissAll() {
    auto copy = g_activeToasts;
    g_activeToasts.clear();
    for (auto h : copy) {
        if (::IsWindow(h)) {
            ::PostMessage(h, WM_CLOSE, 0, 0);
        }
    }
}
```

## 关联代码

| 文件 | 内容 |
|------|------|
| `src/sources/Toast.cpp` | `ToastPopup` 类、`CreateToast`、各 `Show*` 方法 |
| `src/sources/ToastManager.cpp` | `ToastManager` 单例，窗口注册管理 |
| `src/include/EzUI/Toast.h` | `Toast`、`ToastOptions`、`ToastAlign`、`ToastAnimation` |
| `src/include/EzUI/ToastManager.h` | `ToastManager` 类声明 |
| `src/include/EzUI/UIDef.h` | `WM_GUI_TOAST_CLEANUP` 宏定义 |
| `src/sources/Application.cpp` | `__EzUI_MessageWnd` 窗口过程处理 `WM_GUI_TOAST_CLEANUP` |
| `src/include/EzUI/Window.h` | `m_toastLayer` 浮层容器 |
| `src/sources/Window.cpp` | `m_toastLayer` 初始化和布局更新 |

## 踩坑记录

### 坑1：子控件在自定义容器中不被绘制

最早尝试把 Toast 作为 Control 添加到 `m_toastLayer`，子控件不显示。

**原因**：`m_toastLayer` 没加入控件树，绘制链断裂。

**解决**：改用独立 LayeredWindow。

### 坑2：`WS_EX_TRANSPARENT` 导致鼠标穿透

关闭按钮点不到。

**解决**：去掉 `WS_EX_TRANSPARENT`。

### 坑3：LayeredWindow 黑边

拖动窗口时边缘出现黑线。

**解决**：不在 `OnPaint` 画任何边框，用 htm 的 `border` 样式。

### 坑4：Timer 子线程操作 UI 崩溃

`Timer::Tick` 在子线程，直接操作 UI 崩溃。

**解决**：用 Win32 `SetTimer`（主线程消息），或用 `BeginInvoke` 投递到主线程。

### 坑5：htm 文本未转义

文本中的 `& < > "` 导致 htm 解析错误。

**解决**：`EscapeHtml` 手动转义。

### 坑6：LayeredWindow 加载 htm

`LayeredWindow` 没有 `LoadXmlFromString`。

**解决**：
```cpp
auto* umg = GetFrame()->GetUIManager();
umg->LoadXml(htmUtf8.c_str(), htmUtf8.size());
umg->SetupUI(GetFrame());
```

### 坑7：倒计时不实时更新

倒计时标签只在鼠标移入时才更新。

**解决**：更新标签后调用 `Invalidate()`。

### ❗坑8：WM_NCDESTROY 中 delete this 崩溃（最坑）

**现象**：每次关闭 Toast 时程序闪退。

**原因**：`WM_NCDESTROY` 中 `delete this` 后，`DestroyWindow` 返回，但调用栈中 `Window::WndProc(WM_CLOSE)` 还有后续代码（`DefWindowProc(Hwnd(), ...)` 等）访问已释放的 `this`。即使 `WM_CLOSE` 中 `return 0` 拦截了基类处理，`DestroyWindow` 内部走完 `WM_NCDESTROY` 返回后，Windows 内部仍可能访问已释放的 C++ 对象。

**尝试过的失败方案**：

| 方案 | 结果 |
|------|------|
| `WM_NCDESTROY` 中 `delete this` + `return 0` | 闪退 |
| `WM_NCDESTROY` 中 `PostMessage` 异步 delete | 闪退 |
| `WM_CLOSE` 中 `DestroyWindow` + `return 0` | 闪退 |

**最终解决方案**：**不在 `WndProc` 中做任何 `DestroyWindow` 或 `delete` 操作。** 改为：
1. `WM_CLOSE` / `OnTimer` 中只做清理（停定时器、隐藏窗口）
2. `PostMessage` 到 `__EzUI_MessageWnd`（框架的隐藏消息窗口）
3. 在消息窗口的消息处理中统一执行 `DestroyWindow` + `delete`

关键点：
- `DestroyWindow` 和 `delete` 在**消息循环中执行**，不在 WndProc 中
- `WndProc` 返回后不会再有代码访问 `this`
- `DismissAll` 用 `PostMessage(WM_CLOSE)` 而非 `SendMessage`，避免同步递归
- 需要在 `UIDef.h` 定义 `WM_GUI_TOAST_CLEANUP` 宏

### 坑9：按钮没有 Hover 效果

`kind` 样式的按钮默认没有 hover 效果（不像 Button 自带）。

**解决**：手动设置 `HoverStyle.BackColor` 和 `HoverStyle.ForeColor`。

### 坑10：`SendMessage` 同步导致 DismissAll 崩溃

`DismissAll` 中用 `SendMessage(WM_CLOSE)` 会同步进入每个 Toast 的 `WndProc`，在 `WndProc` 中又 `PostMessage` 异步销毁。如果同时有多个 Toast，`SendMessage` 嵌套可能导致时序问题。

**解决**：`DismissAll` 用 `PostMessage(WM_CLOSE)` 异步发送，先 `clear()` 再逐个 `PostMessage`。
