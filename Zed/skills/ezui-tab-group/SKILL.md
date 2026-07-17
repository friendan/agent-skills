---
name: ezui-tab-group
description: 指导如何使用EZUI框架实现标签栏分组功能，包括htm空容器布局、C++动态创建标签、分组切换、角标更新、拖拽排序和TabButton自定义控件。
---

# EZUI 标签栏分组功能

## 概述

标签栏分组功能将标签页划分为多个独立分组，每个分组维护自己的标签列表和页面容器，切换分组时显示对应的标签栏和页面内容。角标显示每个分组的标签数量。

适用于需要多组标签页隔离的场景（如浏览器多用户/多身份切换）。

## 总体架构

### 数据流

```
htm 布局 (空容器)
    ↓ LoadXml + SetupUI
C++ 动态创建 TabButton + Control(页面)
    ↓ Add 到对应 tabBar 和 pageContainer
std::vector<TabInfo> 管理每个分组的标签列表
    ↓
切换分组时: tabBar.SetVisible + pageContainer.SetVisible
    ↓
刷新布局: RefreshLayout + Invalidate
```

### 关键类

| 类 | 作用 |
|------|------|
| `BrowserWindow` (继承 `BorderlessWindow`) | 主窗口，管理分组逻辑、标签创建/切换/关闭 |
| `TabButton` (继承 `HLayout`) | 自定义标签按钮控件，封装外观和交互 |
| `TabLayout` | EZUI 提供的页面容器，支持页面切换 |

## HTM 布局设计

### 空容器方案

htm 中只声明空容器，标签和页面全部由 C++ 动态创建，避免大量隐藏控件干扰布局：

```html
<!-- 标题栏 + 标签栏 + 窗口控制 -->
<hlayout id="titleBarRow" height="36">
    <!-- 分组 A（默认可见） -->
    <hlayout id="tabBar0" class="tabBar" action="title"></hlayout>
    <!-- 分组 B~E（默认隐藏） -->
    <hlayout id="tabBar1" class="tabBar" action="title" visible="false"></hlayout>
    <hlayout id="tabBar2" class="tabBar" action="title" visible="false"></hlayout>
    <hlayout id="tabBar3" class="tabBar" action="title" visible="false"></hlayout>
    <hlayout id="tabBar4" class="tabBar" action="title" visible="false"></hlayout>

    <label id="newTabBtn" text="+" width="30" height="36"></label>
    <hlayout id="windowControls" width="135">
        <label id="minBtn" class="winBtn" text="0" action="mini" width="45" height="36"></label>
        <label id="maxBtn" class="winBtn" text="1" action="max" width="45" height="36"></label>
        <label id="closeBtn" class="winBtn" text="r" action="close" width="45" height="36"></label>
    </hlayout>
</hlayout>

<!-- 页面容器 -->
<hbox>
    <tablayout id="pageContainer0"></tablayout>
    <tablayout id="pageContainer1" visible="false"></tablayout>
    <tablayout id="pageContainer2" visible="false"></tablayout>
    <tablayout id="pageContainer3" visible="false"></tablayout>
    <tablayout id="pageContainer4" visible="false"></tablayout>
</hbox>
```

### 分组按钮和角标

分组按钮使用 `label`，用 EZUI 的 `badge-text` 属性实现角标：

```html
<hlayout id="groupSwitcher" margin="0,4,0,4">
    <label id="groupBtnLabel0" class="groupBtnLabel" text="A" badge-text=""
           badge-size="12" badge-offset-x="13" badge-offset-y="5"
           badge-color="rgba(0,0,0,0)" badge-fore-color="rgb(200,60,60)"
           width="48" height="32"></label>
    <label id="groupBtnLabel1" class="groupBtnLabel" text="B" ...></label>
    <label id="groupBtnLabel2" class="groupBtnLabel" text="C" ...></label>
    <label id="groupBtnLabel3" class="groupBtnLabel" text="D" ...></label>
    <label id="groupBtnLabel4" class="groupBtnLabel" text="E" ...></label>
</hlayout>
```

角标属性说明：
- `badge-text` — 角标文字（空字符串隐藏）
- `badge-size` — 角标直径
- `badge-offset-x/y` — 偏移微调
- `badge-color` — 背景色
- `badge-fore-color` — 文字颜色

## C++ 数据结构

```cpp
static constexpr int kGroupCount = 5;  // A~E 共 5 组

struct TabInfo {
    TabButton* tab = nullptr;      // 标签按钮
    ezui::Control* page = nullptr; // 对应的页面
};

// 每个分组独立维护标签列表
std::vector<TabInfo> m_tabs[kGroupCount];

// 每组当前活动的标签索引
int m_activeTabInGroup[kGroupCount] = {};

// 当前显示的分组
int m_currentGroup = 0;
```

## 核心功能实现

### 1. 初始化

```cpp
void InitUI() {
    m_uiManager = new ezui::UIManager();
    m_uiManager->LoadXml(L"resource/ui/browser.htm");
    m_uiManager->SetupUI(this);

    // 获取 pageContainer
    for (int g = 0; g < kGroupCount; ++g) {
        m_pageContainers[g] = (ezui::TabLayout*)FindControl(L"pageContainer" + g);
    }

    // 新建标签按钮
    auto* newTabBtn = FindControl(L"newTabBtn");
    newTabBtn->EventHandler = [this](...) {
        CreateTab(m_currentGroup, L"新标签");
        SwitchTab(m_currentGroup, m_tabs[m_currentGroup].size() - 1);
        UpdateGroupBadge(m_currentGroup);
    };

    // 分组切换按钮
    for (int g = 0; g < kGroupCount; ++g) {
        auto* label = FindControl(L"groupBtnLabel" + g);
        label->EventHandler = [this, group = g](...) {
            if (arg.EventType == OnMouseDown) SwitchGroup(group);
            else if (arg.EventType == OnMouseEnter) { /* hover 高亮 */ }
            else if (arg.EventType == OnMouseLeave) { /* 恢复颜色 */ }
        };
    }

    // 启动时创建第一个标签
    CreateTab(0, L"主页");
    SwitchTab(0, 0);
    SwitchGroup(0);
}
```

### 2. 创建标签

```cpp
TabInfo CreateTab(int group, const std::wstring& title) {
    auto* tabBar = FindControl(L"tabBar" + group);
    auto* container = m_pageContainers[group];

    TabInfo info;
    info.page = new ezui::Control();
    container->Add(info.page);

    info.tab = new TabButton(tabBar, title);
    tabBar->Add(info.tab);

    int idx = m_tabs[group].size();
    info.tab->OnClick = [this, group, idx]() { SwitchTab(group, idx); };
    info.tab->OnCloseClick = [this, group, idx]() { CloseTab(group, idx); };

    m_tabs[group].push_back(info);
    return info;
}
```

### 3. 切换分组

```cpp
void SwitchGroup(int group) {
    if (group == m_currentGroup) return;

    // 隐藏旧分组
    auto* oldBar = FindControl(L"tabBar" + m_currentGroup);
    auto* oldContainer = m_pageContainers[m_currentGroup];
    oldBar->SetVisible(false);
    oldContainer->SetVisible(false);

    // 显示新分组
    auto* newBar = FindControl(L"tabBar" + group);
    auto* newContainer = m_pageContainers[group];
    newBar->SetVisible(true);
    newBar->RefreshLayout();
    newContainer->SetVisible(true);

    // 如果新分组没有标签则自动创建一个
    if (m_tabs[group].empty()) {
        CreateTab(group, L"新标签");
    }
    SwitchTab(group, m_activeTabInGroup[group]);

    m_currentGroup = group;
    UpdateGroupBadge(group);

    // 强制刷新整个布局
    auto* rootLayout = GetLayout();
    rootLayout->Parent->RefreshLayout();
    rootLayout->RefreshLayout();

    RefreshGroupHighlight();
}
```

### 4. 切换标签

```cpp
void SwitchTab(int group, int index) {
    auto& tabs = m_tabs[group];

    for (auto& t : tabs) {
        t.tab->SetActive(false);
        t.page->SetVisible(false);
    }

    m_activeTabInGroup[group] = index;
    tabs[index].tab->SetActive(true);
    tabs[index].page->SetVisible(true);

    if (group == m_currentGroup) {
        m_pageContainers[group]->SetPage(tabs[index].page);
    }

    FindControl(L"tabBar" + group)->Invalidate();
}
```

### 5. 关闭标签

```cpp
void CloseTab(int group, int index) {
    auto& tabs = m_tabs[group];
    auto* tabBar = FindControl(L"tabBar" + group);

    tabBar->Remove(tabs[index].tab, true);
    m_pageContainers[group]->Remove(tabs[index].page, true);
    tabs.erase(tabs.begin() + index);

    // 更新剩余标签的回调索引（关键！）
    for (int i = 0; i < tabs.size(); ++i) {
        tabs[i].tab->OnClick = [this, group, i]() { SwitchTab(group, i); };
        tabs[i].tab->OnCloseClick = [this, group, i]() { CloseTab(group, i); };
    }

    UpdateGroupBadge(group);

    // 切换到前一个标签
    int prev = index - 1;
    if (prev < 0 && !tabs.empty()) prev = 0;
    if (prev >= 0 && prev < tabs.size()) {
        SwitchTab(group, prev);
    }

    tabBar->RefreshLayout();
    tabBar->Invalidate();
}
```

### 6. 更新角标

```cpp
void UpdateGroupBadge(int group) {
    int count = m_tabs[group].size();
    auto* label = FindControl(L"groupBtnLabel" + group);
    if (label) {
        if (count > 0) {
            label->SetBadge(std::to_wstring(count));
        } else {
            label->SetBadge(L"");
        }
        label->Invalidate();
    }
}
```

### 7. 拖拽排序

使用 `WndProc` 处理 Win32 鼠标消息（`EventHandler` 在 `hlayout` 上不触发鼠标事件）：

```cpp
// 拖拽状态
bool m_dragging = false;
int m_dragSrcIdx = -1;
int m_dragTargetIdx = -1;

LRESULT WndProc(UINT uMsg, WPARAM wParam, LPARAM lParam) {
    int group = m_currentGroup;
    auto& tabs = m_tabs[group];

    if (uMsg == WM_LBUTTONDOWN) {
        POINT pt = { GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam) };
        auto* tabBar = FindControl(L"tabBar" + group);
        auto barRect = tabBar->GetRect();
        int localX = pt.x - barRect.X;
        int localY = pt.y - barRect.Y;
        if (localY >= 0 && localY <= barRect.Height) {
            for (int i = 0; i < tabs.size(); ++i) {
                auto r = tabs[i].tab->GetRect();
                if (localX >= r.X && localX < r.X + r.Width) {
                    m_dragging = true;
                    m_dragSrcIdx = i;
                    break;
                }
            }
        }
    }
    else if (uMsg == WM_MOUSEMOVE && m_dragging) {
        POINT pt = { GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam) };
        auto* tabBar = FindControl(L"tabBar" + group);
        auto barRect = tabBar->GetRect();
        int localX = pt.x - barRect.X;
        int localY = pt.y - barRect.Y;
        if (localY >= 0 && localY <= barRect.Height) {
            int hoverIdx = -1;
            for (int i = 0; i < tabs.size(); ++i) {
                auto r = tabs[i].tab->GetRect();
                if (localX >= r.X && localX < r.X + r.Width) {
                    hoverIdx = i;
                    break;
                }
            }
            // 只高亮目标标签，不交换
            if (hoverIdx != m_dragTargetIdx) {
                // 恢复旧目标标签的颜色
                if (m_dragTargetIdx >= 0) {
                    tabs[m_dragTargetIdx].tab->SetActive(
                        m_dragTargetIdx == m_activeTabInGroup[group]);
                }
                m_dragTargetIdx = hoverIdx;
                // 高亮新目标标签
                if (hoverIdx >= 0 && hoverIdx != m_dragSrcIdx) {
                    tabs[hoverIdx].tab->Style.BackColor = Colors::TabHoverBg();
                    tabs[hoverIdx].tab->Invalidate();
                }
            }
        }
    }
    else if (uMsg == WM_LBUTTONUP) {
        if (m_dragging && m_dragTargetIdx >= 0 && m_dragTargetIdx != m_dragSrcIdx) {
            SwapTabs(group, m_dragSrcIdx, m_dragTargetIdx);
        }
        m_dragging = false;
        m_dragSrcIdx = -1;
        m_dragTargetIdx = -1;
        for (auto& t : tabs) {
            t.tab->SetActive(i == m_activeTabInGroup[group]);
        }
    }

    return BorderlessWindow::WndProc(uMsg, wParam, lParam);
}
```

交换实现：

```cpp
void SwapTabs(int group, int from, int to) {
    auto& tabs = m_tabs[group];
    std::swap(tabs[from], tabs[to]);

    // 重新排列 tabBar 中子控件的顺序
    auto* tabBar = FindControl(L"tabBar" + group);
    std::vector<TabButton*> allTabs;
    for (auto& t : tabs) {
        allTabs.push_back(t.tab);
        tabBar->Remove(t.tab, false);  // false 表示不释放内存
    }
    for (auto* t : allTabs) {
        tabBar->Add(t);
    }
    tabBar->RefreshLayout();
    tabBar->Invalidate();

    // 更新活动标签索引
    if (m_activeTabInGroup[group] == from)
        m_activeTabInGroup[group] = to;
    else if (m_activeTabInGroup[group] == to)
        m_activeTabInGroup[group] = from;

    // 更新回调索引
    for (int i = 0; i < tabs.size(); ++i) {
        tabs[i].tab->OnClick = [this, group, i]() { SwitchTab(group, i); };
        tabs[i].tab->OnCloseClick = [this, group, i]() { CloseTab(group, i); };
    }
}
```

## 关键注意事项

### 1. `FindControl` 无法搜索不可见父容器中的子控件

如果容器设置了 `visible="false"`，它内部的子控件无法通过 `FindControl` 找到。但分组功能不依赖 `FindControl` 查找动态创建的标签，因为标签通过 `m_tabs[g]` 向量管理。

### 2. 切换分组后必须刷新布局

调用 `rootLayout->Parent->RefreshLayout()` + `rootLayout->RefreshLayout()` 确保布局重新计算，否则 tabBar 宽度可能不正确。

### 3. 删除标签后必须更新回调索引

删除中间标签后，后面所有标签在 `vector` 中的索引都变了，必须逐个重建 `OnClick` 和 `OnCloseClick` 回调，否则点击会操作错误的标签。

### 4. `TabButton` 的销毁方式

`tabBar->Remove(tab, true)` — 第二个参数传 `true` 会释放控件内存，传 `false` 只从父控件中移除但不释放。

### 5. 角标的显示/隐藏

`SetBadge("")` 时角标自动隐藏，`SetBadge("数字")` 时自动显示。不需要手动控制 `SetVisible`。

### 6. 拖拽坐标转换

`WM_MOUSEMOVE` 的 `lParam` 是**窗口客户区坐标**，需要减去 `tabBar->GetRect().X/Y` 得到相对坐标，再用每个标签的 `GetRect()` 判断鼠标在哪个标签上。

### 7. 拖拽流程：高亮 ≠ 交换

- `WM_MOUSEMOVE` — 只高亮目标标签（设背景色），**不执行交换**
- `WM_LBUTTONUP` — 才执行 `SwapTabs`
