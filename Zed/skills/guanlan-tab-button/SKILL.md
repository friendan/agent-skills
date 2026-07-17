---
name: guanlan-tab-button
description: 指导如何创建和使用观澜浏览器的TabButton自定义标签控件，包括继承HLayout、鼠标事件处理、hover样式、关闭按钮交互等。
---

# 观澜浏览器 TabButton 标签控件

## 概述

`TabButton` 是观澜浏览器中封装标签页按钮的自定义控件，继承 `ezui::HLayout`，包含标题文字和关闭按钮，支持 hover 背景色变化、关闭按钮悬停高亮、激活/非激活状态切换。

## 设计思路

- 继承 `HLayout`，内部用子控件（`Label` + `Spacer` + `Label`）实现布局
- 所有子控件 `SetHitTestVisible(false)`，让鼠标事件穿透到 `HLayout` 自身
- 通过 `EventHandler` 统一处理鼠标事件（enter/leave/move/down）
- 通过回调函数 `OnClick` / `OnCloseClick` 通知外部

## 头文件 (TabButton.h)

```cpp
#pragma once

#include <EzUI/Label.h>
#include <EzUI/HLayout.h>
#include <EzUI/Spacer.h>
#include <functional>

namespace guanlan {

class TabButton : public ezui::HLayout {
public:
    std::function<void()> OnClick;
    std::function<void()> OnCloseClick;

    TabButton(ezui::Control* parent, const std::wstring& title);
    virtual ~TabButton();

    void SetTabTitle(const std::wstring& title);
    void SetActive(bool active);

private:
    ezui::Label* m_titleLabel = nullptr;
    ezui::Label* m_closeBtn = nullptr;
    bool m_active = false;
    bool m_closeHovered = false;

    void UpdateStyle();
    ezui::Rect GetCloseRect() const;
};

} // namespace guanlan
```

## 实现要点 (TabButton.cpp)

### 构造函数

```cpp
TabButton::TabButton(ezui::Control* parent, const std::wstring& title)
    : HLayout(parent) {
    SetFixedWidth(120);
    SetFixedHeight(kTabBarHeight);

    // 子控件：左边距 + 标题 + 弹性spacer + 关闭按钮
    // 所有子控件 SetHitTestVisible(false) 让事件穿透到 HLayout
}
```

### 事件处理

所有鼠标事件通过 `EventHandler` 绑定在 `HLayout` 自身：

```cpp
EventHandler = [this](ezui::Control*, ezui::EventArgs& arg) {
    if (arg.EventType == ezui::Event::OnMouseDown) {
        // 判断是否点击了关闭按钮区域
        auto closeRect = GetCloseRect();
        auto& mouseArg = (ezui::MouseEventArgs&)arg;
        if (closeRect.Contains(mouseArg.Location)) {
            if (OnCloseClick) OnCloseClick();
        } else {
            if (OnClick) OnClick();
        }
    }
    else if (arg.EventType == ezui::Event::OnMouseEnter) {
        // 非激活状态显示 hover 背景色
        if (!m_active) {
            Style.BackColor = Colors::TabHoverBg();
            Invalidate();
        }
    }
    else if (arg.EventType == ezui::Event::OnMouseLeave) {
        // 恢复背景色
        m_closeHovered = false;
        if (m_closeBtn) m_closeBtn->Style.BackColor = ezui::Color::Transparent;
        UpdateStyle();
        Invalidate();
    }
    else if (arg.EventType == ezui::Event::OnMouseMove) {
        // 检测关闭按钮悬停
        auto closeRect = GetCloseRect();
        auto& mouseArg = (ezui::MouseEventArgs&)arg;
        bool overClose = closeRect.Contains(mouseArg.Location);
        if (overClose != m_closeHovered) {
            m_closeHovered = overClose;
            if (overClose) {
                m_closeBtn->Style.BackColor = ezui::Color(180, 40, 40);
                m_closeBtn->Style.ForeColor = ezui::Color(220, 220, 220);
            } else {
                m_closeBtn->Style.BackColor = ezui::Color::Transparent;
                m_closeBtn->Style.ForeColor = ezui::Color(140, 140, 140);
            }
            m_closeBtn->Invalidate();
        }
    }
};
```

## 在 BrowserWindow 中使用

### 创建标签

```cpp
auto* tabBar = FindControl(L"tabBar" + std::to_wstring(group));
auto info = new TabButton(tabBar, title);
tabBar->Add(info);

info->OnClick = [this, group, idx]() { SwitchTab(group, idx); };
info->OnCloseClick = [this, group, idx]() { CloseTab(group, idx); };
```

### 切换激活状态

```cpp
tab->SetActive(true);   // 激活
tab->SetActive(false);  // 非激活
```

### 关闭标签

```cpp
tabBar->Remove(tab, true);
```

## 颜色配置 (UIDef.h)

```cpp
inline ezui::Color TabActiveBg() { return ezui::Color(30, 30, 38); }
inline ezui::Color TabInactiveBg() { return ezui::Color(60, 60, 70); }
inline ezui::Color TabHoverBg() { return ezui::Color(100, 120, 140); }
inline ezui::Color TabTextActive() { return ezui::Color(240, 240, 245); }
inline ezui::Color TabTextInactive() { return ezui::Color(200, 200, 205); }
```

## 关键注意事项

1. **`HLayout` 不响应鼠标事件** — 需要让所有子控件 `SetHitTestVisible(false)`，鼠标事件才能穿透到 `HLayout` 的 `EventHandler`
2. **不要用 `HLayout` 的虚函数（`OnMouseEnter` 等）** — 经测试这些虚函数在 `HLayout` 上不会触发，必须用 `EventHandler`
3. **关闭按钮坐标计算** — `GetCloseRect` 使用 `m_closeBtn->GetRect()` 获取相对于父控件（TabButton）的坐标
4. **回调索引问题** — 删除标签后后续标签索引会变化，需要重建所有回调
