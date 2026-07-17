---
name: ezui-tab-button
description: 指导如何创建和使用EZUI框架的TabButton自定义标签控件，包括继承HLayout、鼠标事件处理、hover样式、关闭按钮交互等。
---

# EZUI TabButton 自定义标签控件

## 概述

`TabButton` 是一个封装标签页按钮的自定义控件，继承 `ezui::HLayout`，包含标题文字和关闭按钮，支持 hover 背景色变化、关闭按钮悬停高亮、激活/非激活状态切换。

适用于浏览器或多标签页应用中的单个标签按钮。

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

    // 所有子控件 SetHitTestVisible(false) 让事件穿透到 HLayout
    auto* leftSpacer = new ezui::Spacer();
    leftSpacer->SetFixedWidth(8);
    leftSpacer->SetHitTestVisible(false);
    Add(leftSpacer);

    m_titleLabel = new ezui::Label(this);
    m_titleLabel->SetText(title);
    m_titleLabel->Style.ForeColor = inactiveColor;
    m_titleLabel->Style.FontSize = 12;
    m_titleLabel->SetHitTestVisible(false);
    Add(m_titleLabel);

    auto* midSpacer = new ezui::Spacer();
    midSpacer->SetHitTestVisible(false);
    Add(midSpacer);

    m_closeBtn = new ezui::Label(this);
    m_closeBtn->SetText(L"\u2715");
    m_closeBtn->Style.ForeColor = ezui::Color(140, 140, 140);
    m_closeBtn->Style.FontSize = 10;
    m_closeBtn->Style.BackColor = ezui::Color::Transparent;
    m_closeBtn->SetFixedWidth(20);
    m_closeBtn->SetHitTestVisible(false);
    Add(m_closeBtn);

    // 鼠标事件绑在 HLayout 自身
    BindEvents();
    UpdateStyle();
}
```

### 事件处理

所有鼠标事件通过 `EventHandler` 绑定在 `HLayout` 自身：

```cpp
void BindEvents() {
    EventHandler = [this](ezui::Control*, ezui::EventArgs& arg) {
        if (arg.EventType == ezui::Event::OnMouseDown) {
            auto closeRect = GetCloseRect();
            auto& mouseArg = (ezui::MouseEventArgs&)arg;
            if (closeRect.Contains(mouseArg.Location)) {
                if (OnCloseClick) OnCloseClick();
            } else {
                if (OnClick) OnClick();
            }
        }
        else if (arg.EventType == ezui::Event::OnMouseEnter) {
            if (!m_active) {
                Style.BackColor = hoverColor;
                Invalidate();
            }
        }
        else if (arg.EventType == ezui::Event::OnMouseLeave) {
            m_closeHovered = false;
            m_closeBtn->Style.BackColor = ezui::Color::Transparent;
            UpdateStyle();
            Invalidate();
        }
        else if (arg.EventType == ezui::Event::OnMouseMove) {
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
}
```

### 状态切换

```cpp
void TabButton::SetActive(bool active) {
    m_active = active;
    UpdateStyle();
    Invalidate();
}

void TabButton::UpdateStyle() {
    if (m_active) {
        Style.BackColor = Colors::TabActiveBg();
        m_titleLabel->Style.ForeColor = Colors::TabTextActive();
    } else {
        Style.BackColor = Colors::TabInactiveBg();
        m_titleLabel->Style.ForeColor = Colors::TabTextInactive();
    }
}
```

## 颜色配置参考

```cpp
// 标签背景色
inline ezui::Color TabActiveBg() { return ezui::Color(30, 30, 38); }
inline ezui::Color TabInactiveBg() { return ezui::Color(60, 60, 70); }
inline ezui::Color TabHoverBg() { return ezui::Color(100, 120, 140); }

// 标签文字颜色
inline ezui::Color TabTextActive() { return ezui::Color(240, 240, 245); }
inline ezui::Color TabTextInactive() { return ezui::Color(200, 200, 205); }
```

## 在项目中使用

### 创建标签

```cpp
auto* tabBar = FindControl(L"tabBar0");
auto* tab = new TabButton(tabBar, L"主页");
tabBar->Add(tab);

int idx = tabs.size();
tab->OnClick = [this, idx]() { SwitchTab(idx); };
tab->OnCloseClick = [this, idx]() { CloseTab(idx); };
```

### 切换激活状态

```cpp
tab->SetActive(true);   // 激活
tab->SetActive(false);  // 非激活
```

### 关闭标签

```cpp
tabBar->Remove(tab, true);  // true 表示释放内存
```

## 关键注意事项

1. **`HLayout` 不响应鼠标事件** — 必须让所有子控件 `SetHitTestVisible(false)`，鼠标事件才能穿透到 `HLayout` 的 `EventHandler`
2. **不要用 `HLayout` 的虚函数处理鼠标**（如 `OnMouseEnter`） — 这些虚函数在 `HLayout` 上不会触发，必须用 `EventHandler`
3. **关闭按钮坐标计算** — `GetCloseRect` 用 `m_closeBtn->GetRect()` 获取相对于父控件（TabButton 自身）的坐标
4. **回调中的索引问题** — 删除标签后 vector 中的索引变化，需要重建回调
