# EZUI Tooltip 提示实现

## 概述

Tooltip 使用 `LayeredWindow`（分层窗口）实现，是一个独立无焦点小窗口。内部通过内存 htm 布局加载控件，利用框架的 `kind` 色调体系和 `width="auto"` 自适应宽度。

## 完整 API

```cpp
// ===== 设置 Tooltip =====
Tooltip::Set(ctl, L"提示文本");                             // 默认 secondary 颜色
Tooltip::Set(ctl, L"保存成功", L"success");                 // 指定 kind 颜色
Tooltip::Set(ctl, L"删除失败", L"danger");

// ===== 移除 Tooltip =====
Tooltip::Remove(ctl);

// ===== 清空所有 =====
Tooltip::ClearAll();
```

Tooltip 通过 `Control::SetToolTip()` 设置文字，`Window::OnMouseHover` 自动触发显示：

```cpp
btn->SetToolTip(L"点我保存");              // 设文字
// 或配合 Tooltip::Set 指定颜色
Tooltip::Set(btn, L"点我保存", L"success");
```

## 架构

```
鼠标悬停 500ms
  → Window::WndProc(WM_MOUSEHOVER)
    → Window::OnMouseHover(point)
      → HitTestControl 找到控件
      → SendEvent(ctl, OnMouseHover)
      → ctl->GetToolTip() 非空 → Tooltip::Show(ctl, screenX, screenY, ownerHwnd)

Tooltip::Show 内部：
  → DestroyCurrentTooltip()          // 隐藏并异步销毁旧的
  → BuildTooltipHtm(kind, text)      // 拼接 htm
  → new TooltipPopup(600, 30, ...)   // 创建大窗口让布局不受限
    → LoadXml + SetupUI
  → root->SetAutoWidth(true)         // 让 hlayout 自适应内容宽度
  → root->RefreshLayout()            // 强制计算
  → root->Width() 获取实际宽度
  → SetRect 调整窗口到实际大小
  → Show()
```

## HTM 布局

```cpp
static std::string BuildTooltipHtm(const std::string& kindName, const std::string& escapedText) {
    char b[2048];
    snprintf(b, sizeof(b),
        R"(<hlayout dock="fill" kind="%s" style="border-radius: 6px;">
  <spacer width="8"></spacer>
  <label text="%s" font-size="12" halign="left" width="auto"></label>
  <spacer width="8"></spacer>
</hlayout>)", kindName.c_str(), escapedText.c_str());
    return b;
}
```

关键点：
- **hlayout**：`kind="%s"` 提供全局色调（与按钮同款颜色），`dock="fill"` 撑满 IFrame
- **spacer**：左右各 8px 边距
- **label**：`width="auto"` 自适应文字宽度，`halign="left"` 左对齐

## 窗口样式

```cpp
LayeredWindow(w, h, owner, WS_POPUP,
    WS_EX_NOACTIVATE | WS_EX_LAYERED | WS_EX_TOOLWINDOW | WS_EX_TRANSPARENT)
```

- `WS_EX_NOACTIVATE` — 点击时不获取焦点
- `WS_EX_LAYERED` — 分层窗口，支持透明
- `WS_EX_TOOLWINDOW` — 不在任务栏显示
- `WS_EX_TRANSPARENT` — 鼠标穿透，避免 tooltip 盖在控件上导致闪烁
- `SetShadow(0)` — 关闭阴影，消除白边

## 宽度自适应核心代码

创建 TooltipPopup 后，拿到根控件（hlayout）主动设置 auto 并强制布局：

```cpp
g_tooltipPopup = new TooltipPopup(600, 30, owner, htm);

int popupW = 280, popupH = 30;
auto& children = g_tooltipPopup->GetFrame()->GetControls();
if (!children.empty()) {
    auto* root = *children.begin();
    if (root) {
        root->SetAutoWidth(true);
        root->SetAutoHeight(true);
        root->RefreshLayout();
        int rw = root->Width();
        int rh = root->Height();
        if (rw > 20 && rh > 10) {
            popupW = rw;
            popupH = rh;
        }
    }
}
// 用计算出的实际大小设置窗口
g_tooltipPopup->SetRect(Rect(x, y, popupW, popupH));
g_tooltipPopup->Show();
```

**重要**：初始窗口要够大（如 600x30），让布局计算不受容器宽度限制。之后 resize 到实际内容大小。

## 异步销毁

和 Toast 完全相同的模式——不在 WndProc 中销毁窗口：

```cpp
// WndProc(WM_CLOSE) 中：
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
| `src/sources/Tooltip.cpp` | `TooltipPopup` 类、`Tooltip` 静态 API |
| `src/include/EzUI/Tooltip.h` | `Tooltip` 类声明 |
| `src/include/EzUI/UIDef.h` | `WM_GUI_TOOLTIP_CLEANUP` 宏定义 |
| `src/sources/Application.cpp` | 消息窗口处理 `WM_GUI_TOOLTIP_CLEANUP` |
| `src/sources/Window.cpp` | `Window::OnMouseHover` 触发 `Tooltip::Show` |
| `src/include/EzUI/Control.h` | `m_tooltip` / `SetToolTip()` / `GetToolTip()` |

## 踩坑记录

### ❗坑1：WM_NCDESTROY 中 delete this 崩溃（同 Toast）

**现象**：关闭 Tooltip 时闪退。

**原因和解决方案**：和 Toast 完全相同——不能在 WndProc 中 `DestroyWindow` + `delete this`，必须 `PostMessage` 到 `__EzUI_MessageWnd` 异步执行。详情见 Toast 坑8。

### ❗坑2：Tooltip 闪烁

**现象**：鼠标移到按钮上，Tooltip 出现后闪烁不停。

**原因**：Tooltip 窗口盖在按钮上，鼠标移到 Tooltip 上触发按钮的 `OnMouseLeave` → `Tooltip::Hide()`，Tooltip 消失后鼠标回到按钮 → `OnMouseEnter` → `OnMouseHover` → 再次 `Tooltip::Show`，循环往复。

**解决方案一**（推荐）：加 `WS_EX_TRANSPARENT` 让鼠标穿透 Tooltip，鼠标事件直接透到主窗口的按钮上。

**解决方案二**（代码防护）：`Tooltip::Show` 开头判断如果已经显示且未销毁，只 reposition 不重建：
```cpp
if (g_tooltipPopup && !g_tooltipPopup->m_dismissed) {
    // 只更新位置，不销毁重建
    g_tooltipPopup->SetRect(Rect(x, y, w, h));
    return;
}
```

### ❗坑3：宽度不自适应

**现象**：Tooltip 宽度固定，文字多时显示不全，文字少时太宽。

**原因**：创建 TooltipPopup 时初始窗口大小固定（如 280px），hlayout 的 `dock="fill"` 撑满父容器（280px），label 的 `width="auto"` 虽然计算了文字宽度，但父容器 hlayout 宽度不会自动收缩。

**尝试过的错误方案**：

| 方案 | 结果 |
|------|------|
| label 设 `width="auto"`，窗口固定 280px | 宽内容显示不全 |
| label 设 `width="auto"` + GDI 计算文本像素宽度 | 和 EZUI 布局的文本宽度不一致，总是差一些 |
| `GetLayout()->Width()` 获取宽度 | 返回的是 hlayout 宽度（即窗口宽度），不是内容宽度 |
| `FindControl` 找 label 拿 Width | label 宽度虽是 auto 但容器宽度限制导致计算结果不对 |

**最终解决方案**：
1. 创建足够大的初始窗口（600x30），让布局不受容器宽度限制
2. 获取根控件（hlayout），主动调用 `SetAutoWidth(true)` + `RefreshLayout()`
3. hlayout 的 `RefreshLayout` 递归计算时，label 的 auto 宽度不受限于父容器（`IsAutoWidth()` 时 `maxWidth = EZUI_FLOAT_MAX`），hlayout 自己也会根据子控件总宽度 auto 计算出正确大小
4. 用 `root->Width()` 拿到 hlayout 的实际内容宽度，resize 窗口

关键代码：
```cpp
root->SetAutoWidth(true);
root->SetAutoHeight(true);
root->RefreshLayout();
int rw = root->Width();
int rh = root->Height();
```

### 坑4：右侧白色方块

**现象**：Tooltip 窗口右侧或底部有白色区域。

**原因**：初始窗口（600x30）的 `m_winBitmap` 位图大小是 600x30，resize 后旧的位图残留白色。

**解决**：`SetRect` 调用 `::MoveWindow` 触发 `WM_SIZE` → `LayeredWindow::OnSize` → 删除旧位图创建新位图。所以不是真问题。如果有白边，检查 `BorderlessWindow` 的 `ShadowBox` 阴影窗口，用 `SetShadow(0)` 关闭。
