---
name: ezui-toast
description: 指导如何在EZUI框架中实现Toast通知功能，包括LayeredWindow方案、htm布局、kind色调、关闭按钮、拖动、堆叠定位、倒计时等。
---

# EZUI Toast 通知实现

## 概述

Toast 通知使用 `LayeredWindow`（分层窗口）实现，每个 Toast 是一个独立的无焦点小窗口。内部通过 htm 布局字符串加载控件，利用框架现有的 `kind` 色调体系和 `action` 行为机制。

## 架构

```
Toast::ShowSuccess("操作成功")
  → CreateKindToast(ControlKind::Success, text, 3000)
    → BuildToastHtm("success", escapedText)   // 用 snprintf 拼接 htm
    → new ToastPopup (LayeredWindow)
      → UIManager::LoadXml + SetupUI 加载布局
      → SetupCloseBtn(popup)                   // 设置关闭按钮 hover 样式
    → ShowAtBottomRight(popup, owner)          // 定位 + Show
    → popup->StartTimer(3000)                  // 启动倒计时
```

每个 Toast 是一个独立的分层窗口，不依赖于主窗口的控件树。

## 完整 API

```cpp
// ===== 最简用法（带默认自动关闭时间） =====
Toast::ShowSuccess("操作成功");       // 3s
Toast::ShowDanger("删除失败");        // 5s
Toast::ShowWarning("网络不稳定");     // 4s
Toast::ShowInfo("你有新消息");        // 3s

// ===== 自定义时长（毫秒），传 0 禁止自动关闭 =====
Toast::ShowSuccess("操作成功", 2000);
Toast::ShowDanger("删除失败", 0);     // 不自动关闭
Toast::ShowWarning("网络延迟较高", 6000);

// ===== 通用 Toast（通过 ToastOptions） =====
Toast::Show("保存成功", ToastOptions().Kind(ControlKind::Primary).Duration(3000));

// ===== 关闭所有 =====
Toast::DismissAll();
```

## 各风格默认自动关闭时间

| 方法 | 默认时间 |
|------|---------|
| `ShowSuccess` | 3000ms |
| `ShowDanger` | 5000ms |
| `ShowWarning` | 4000ms |
| `ShowInfo` | 3000ms |

传 `0` 禁止自动关闭：`Toast::ShowInfo("手动关闭", 0)`

## HTM 布局（BuildToastHtm）

使用 `snprintf` + 一个 `R"(...)"` 整体拼接，避免多个 R 片段：

```cpp
static std::string BuildToastHtm(const char* kindName, const std::string& textUtf8) {
    char buf[2048];
    snprintf(buf, sizeof(buf),
        R"(<vbox id="root" dock="fill" style="background-color: rgb(255,255,255); border: 1px solid rgba(0,0,0,0.12); border-radius: 8px;">
  <hlayout id="toast-header" kind="%s" dock="fill" action="title" style="padding-left: 14px; padding-right: 6px; border-radius: 8px;">
    <spacer width="6"></spacer>
    <label text="%s" font-size="13" style="font-weight: bold;" action="title" valign="middle"></label>
    <spacer width="star"></spacer>
    <label id="toastTimer" text="" width="30" font-size="11" style="color: rgba(255,255,255,0.7);" valign="middle" halign="center"></label>
    <label id="toastCloseBtn" text="" width="22" height="22" bsicon="x-lg" icon-size="12" action="close" style="cursor: hand; border-radius: 4px;" valign="middle" halign="center"></label>
  </hlayout>
</vbox>)",
        kindName, textUtf8.c_str());
    return buf;
}
```

关键点：
- **外部 vbox**：白色背景 (`rgb(255,255,255)`) + 细边框 + 圆角
- **header（hlayout）**：`kind="xxx"` 提供全局色调（按钮同款颜色），`action="title"` 支持拖动
- **文字标签**：`action="title"` 使文字区域也可拖动
- **倒计时标签**（`toastTimer`）：显示 `3s` `2s` `1s`，每秒更新
- **关闭按钮**：`bsicon="x-lg"` 显示 SVG 关闭图标，`action="close"` 关闭窗口

## 关闭按钮 Hover 效果

所有 kind 统一使用半透明黑色背景，在任何色调上都能看到变化：

```cpp
static void SetupCloseBtn(ToastPopup* popup) {
    auto* btn = popup->FindControl(L"toastCloseBtn");
    if (btn) {
        btn->HoverStyle.BackColor = Color(0, 0, 0, 50);   // 半透明黑
        btn->HoverStyle.ForeColor = Color(255, 255, 255); // 白色图标
    }
}
```

## 倒计时实现

```cpp
void StartTimer(int ms) {
    m_duration = ms;
    m_remaining = ms;
    if (m_duration <= 0) { /* 隐藏倒计时标签，不启动定时器 */ return; }
    UpdateTimerLabel();  // 显示 "3s"
    StartTick();
}

void StartTick() {
    // 每秒触发一次
    m_closeTimer->Tick = [this](Timer* t) {
        t->Stop();
        Sleep(1000);
        m_remaining -= 1000;
        if (m_remaining <= 0) {
            BeginInvoke([this]() { Close(0); });  // 自动关闭
        } else {
            BeginInvoke([this]() {
                UpdateTimerLabel();   // 更新 "2s" → "1s"
                Invalidate();         // 强制刷新
                StartTick();          // 启动下一秒
            });
        }
    };
    m_closeTimer->Start();
}
```

- 使用链式 `StartTick` 每秒触发，不依赖 Timer 的 Interval 循环
- UI 操作通过 `BeginInvoke` 投递到主线程
- 调用 `Invalidate()` 确保 LayeredWindow 更新倒计时文本

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
static void ShowAtBottomRight(ToastPopup* popup, HWND owner) {
    int w = popup->Width();
    int h = popup->Height();
    int index = (int)g_activeToasts.size();
    g_activeToasts.push_back(popup);
    if (owner) {
        RECT r;
        GetWindowRect(owner, &r);
        int x = r.right - w - 16;
        int y = r.bottom - h - 16 - index * (h + 8);
        popup->SetRect(Rect(x, y, w, h));
    }
    popup->Show();
}
```

每次新 Toast 往上堆叠，间距 8px。

## 文本转义

动态拼接 htm 时，文本中的 `& < > "` 需要转义：

```cpp
static std::string EscapeHtml(const std::wstring& text) {
    std::wstring s = text;
    size_t pos = 0;
    while ((pos = s.find(L'&', pos)) != std::wstring::npos)  { s.replace(pos, 1, L"&amp;");  pos += 5; }
    while ((pos = s.find(L'<', pos)) != std::wstring::npos)  { s.replace(pos, 1, L"&lt;");   pos += 4; }
    while ((pos = s.find(L'>', pos)) != std::wstring::npos)  { s.replace(pos, 1, L"&gt;");   pos += 4; }
    while ((pos = s.find(L'"', pos)) != std::wstring::npos)  { s.replace(pos, 1, L"&quot;"); pos += 6; }
    std::string utf8;
    ui_text::UnicodeToUTF8(s, &utf8);
    return utf8;
}
```

## 实现步骤

1. 创建 `ToastPopup` 继承 `LayeredWindow`
2. 用 `BuildToastHtm` 拼接 htm 布局（`snprintf` + 一个 `R"(...)"`）
3. 通过 `GetFrame()->GetUIManager()` 加载 htm
4. 设置关闭按钮 HoverStyle
5. 定位到右下角并 Show
6. 启动倒计时（链式 StartTick）

## 踩坑记录

### 坑1：子控件在自定义容器中不被绘制

最早尝试把 Toast 作为 Control 添加到 Window 的 `m_toastLayer`（一个普通 Control），但子控件（Label）完全不显示。

**原因**：`m_toastLayer` 没有加到控件树中，子控件的递归绘制链因缺少 Parent 或 Frame 引用而断裂。

**解决方案**：改用独立窗口（LayeredWindow），每个 Toast 是独立窗口，控件树完整。

### 坑2：`WS_EX_TRANSPARENT` 导致鼠标事件穿透

关闭按钮点击无反应。

**原因**：`WS_EX_TRANSPARENT` 让鼠标事件穿透。

**解决方案**：去掉 `WS_EX_TRANSPARENT`。

### 坑3：LayeredWindow 边缘出现黑边

`OnPaint` 中绘制阴影边框后，拖动窗口边缘出现黑线。

**原因**：LayeredWindow 只绘制客户区内内容，边框抗锯齿像素在边缘外显示为黑边。

**解决方案**：不在 `OnPaint` 中画任何边框，由 htm 的 `border` 样式提供。

### 坑4：Timer 线程安全问题

`Timer::Tick` 在子线程中执行，直接操作 UI 导致崩溃。

**解决方案**：使用 `BeginInvoke()` 将 UI 操作投递到主线程。

### 坑5：htm 文本未转义导致布局错乱

文本中的 `& < > "` 导致 htm 解析错误。

**解决方案**：用 `EscapeHtml` 手动转义。

### 坑6：LayeredWindow 加载 htm 需用 UIManager

`LayeredWindow` 没有 `LoadXmlFromString` 方法。

**解决方案**：
```cpp
auto* umg = GetFrame()->GetUIManager();
umg->LoadXml(htmUtf8.c_str(), htmUtf8.size());
umg->SetupUI(GetFrame());
```

### 坑7：倒计时不实时更新

倒计时标签只在鼠标移入窗口时才更新。

**原因**：`BeginInvoke` 投递的 UI 更新需要 `Invalidate()` 触发 LayeredWindow 重绘。

**解决方案**：在更新标签后调用 `Invalidate()`。
