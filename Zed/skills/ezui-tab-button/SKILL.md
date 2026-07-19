---
name: ezui-tab-button
description: 指导如何使用EZUI框架的TabBar和TabButton控件，包括浏览器风格的标签栏、拖拽滚动、右键交换、增删标签、事件监听、htm属性设置等。
---

# EZUI TabBar + TabButton 标签栏控件

## 概述

`TabBar` 和 `TabButton` 是一对配合使用的浏览器风格标签栏控件：

- **TabBar** — 继承 `HLayout`，标签容器，管理标签增删、切换、拖拽滚动、右键交换排序
- **TabButton** — 继承 `HLayout`，单个标签按钮，包含标题文字和关闭按钮，支持 hover/active 状态

标签的所有鼠标事件（点击切换、拖拽滚动、右键交换）由 **TabBar** 统一管理，TabButton 只负责绘制和关闭按钮的 hover 交互。

## 核心架构

### 数据流

```
htm 声明 <tabbar> 容器
    ↓ LoadXml
TabBar::Add() 接管 TabButton 子控件管理
    ↓
用户交互触发 TabBar::OnMouseDown/Up/Move
    ↓
TabBar 操作 m_tabs vector（增删/交换/滚动）
    ↓
RefreshLayout + Invalidate 刷新显示
```

### 关键设计点

| 设计 | 说明 |
|------|------|
| TabButton `SetHitTestVisible(false)` | 让鼠标事件穿透到 TabBar 统一处理 |
| 关闭按钮 `SetHitTestVisible(true)` | 唯一可以命中鼠标的子控件 |
| `m_tabs` vector | TabBar 独立管理标签列表，与 HLayout 子控件列表一一对应 |
| 滚动方式 | `SetScrollOffset` 直接修改所有标签的 `r.X` 坐标 |
| 标签交换 | `std::swap(m_tabs[from], m_tabs[to])` + 同步交换 HLayout 子控件位置 |

### TabBar 成员变量

```cpp
std::vector<TabButton*> m_tabs;     // 标签列表
int m_activeTab = -1;               // 当前激活标签索引
int m_scrollOffset = 0;             // 当前滚动偏移（负值，0 表示无滚动）
int m_maxScrollOffset = 0;          // 最大可滚动偏移（负值）
int m_rightDragSrcIdx = -1;         // 右键拖拽起始索引
int m_dragSrcIdx = -1;              // 左键拖拽滚动起始索引
int m_tabWidth = 150;               // 默认标签宽度
std::vector<bool> m_tabHover;       // hover 状态记录
```

## HTM 属性

### TabBar 属性

```html
<tabbar id="tabBar" height="36" tab-width="150">
    <tabbutton title="主页"></tabbutton>
</tabbar>
```

| 属性 | 说明 |
|------|------|
| `height` | 标签栏高度 |
| `tab-width` | 默认标签宽度（最小40），不设置则为150 |
| `id` | 控件标识，用于 `FindControl` |

### TabButton 属性

```html
<tabbutton title="百度" url="https://www.baidu.com" dir="C:\docs" active="true"></tabbutton>
```

| 属性 | 说明 |
|------|------|
| `title` / `text` | 标签标题文字 |
| `active` | `"true"` 或 `"1"` 设置激活状态 |
| `url` | 关联的网页地址 |
| `dir` | 关联的目录路径 |
| `locked` | `"true"` 或 `"1"` 锁定标签（不可关闭，关闭按钮隐藏） |

## C++ 接口

### TabBar 接口

```cpp
// 事件
std::function<bool(int oldIndex, int newIndex)> OnTabChanging;  // 标签切换前，返回false阻止切换
std::function<void(int oldIndex, int newIndex)> OnTabChanged;   // 标签切换，参数为旧索引和新索引
std::function<void(int index)> OnTabClosed;                     // 标签关闭

// 增删标签
TabButton* AddTab(const UIString& title);                              // 追加标签
TabButton* InsertTab(int index, const UIString& title);                // 插入标签到指定位置
void RemoveTab(int index);                                             // 按索引删除
void RemoveTab(TabButton* tab);                                        // 按指针删除
void RemoveAllTabs();                                                  // 清空所有标签
int GetTabCount();                                                     // 标签数量
TabButton* GetTab(int index);                                          // 按索引获取
int GetTabIndex(TabButton* tab);                                       // 获取索引
int FindTabByTitle(const UIString& title);                             // 按标题查找
const std::vector<TabButton*>& GetAllTabs();                           // 获取全部标签

// 切换标签
void SetActiveTab(int index);                                          // 激活指定标签
int GetActiveTab();                                                    // 获取激活索引
TabButton* GetActiveTabButton();                                       // 获取激活的标签指针

// 标签宽度
void SetTabWidth(int width);                                           // 默认宽度（最小40）
int GetTabWidth();

// 标签移动
void MoveTab(int from, int to);                                        // 交换两个标签
```

### TabButton 接口

```cpp
void SetTabTitle(const UIString& title);
UIString GetTabTitle();
void SetActive(bool active);
bool IsActive();
void SetButtonWidth(int width);   // 单个标签宽度（最小40）
int GetButtonWidth();
void SetUrl(const UIString& url);
UIString GetUrl();
void SetDir(const UIString& dir);
UIString GetDir();
void SetLocked(bool locked);
bool IsLocked();
Label* GetTitleLabel();           // 获取标题 Label（可定制样式）
Label* GetCloseBtn();             // 获取关闭按钮 Label
Rect GetCloseRect();              // 获取关闭按钮相对坐标矩形

// 回调
std::function<void()> OnClick;
std::function<void()> OnCloseClick;
```

## 使用示例

```cpp
// 获取 TabBar
TabBar* tabBar = (TabBar*)FindControl(L"tabBar");

// 添加标签
TabButton* tab = tabBar->AddTab(L"新标签");

// 设置其他属性
tab->SetUrl(L"https://example.com");
tab->SetDir(L"C:\\MyFolder");

// 监听事件
tabBar->OnTabChanging = [](int oldIdx, int newIdx) -> bool {
    if (newIdx == 2) return false; // 阻止切换到索引2
    return true;
};
tabBar->OnTabChanged = [this](int oldIdx, int newIdx) {
    // 从 oldIdx 切换到 newIdx 的处理
};
tabBar->OnTabClosed = [this](int index) {
    // 关闭 index 标签后的处理
};

// 插入标签到指定位置
tabBar->InsertTab(0, L"置顶标签");

// 获取当前激活标签
TabButton* activeTab = tabBar->GetActiveTabButton();
if (activeTab) {
    UIString title = activeTab->GetTabTitle();
}

// 查找标签
int idx = tabBar->FindTabByTitle(L"主页");

// 锁定/解锁标签
m_tabBar->GetTab(0)->SetLocked(true);  // 主页锁定，不可关闭

// 清空
tabBar->RemoveAllTabs();
```

## 交互逻辑

### 鼠标事件

所有鼠标事件在 **TabBar** 上处理，TabButton 自身 `SetHitTestVisible(false)`：

| 操作 | 行为 |
|------|------|
| **左键按下 + 同位置弹起** | 激活该标签；若点击在关闭按钮上则关闭（锁定标签忽略关闭操作） |
| **左键按下 + 不同位置弹起** | 计算拖拽距离，滚动标签 |
| **右键按下 + 弹起** | 交换两个标签位置 |
| **鼠标移动** | 更新 hover 效果和关闭按钮高亮 |
| **鼠标离开** | 清除所有 hover 效果 |

### 滚动机制

```cpp
// 所有标签的 r.X 同时偏移，实现滚动效果
void TabBar::SetScrollOffset(int offset) {
    if (offset < m_maxScrollOffset) offset = m_maxScrollOffset;  // 限制最大滚动
    if (offset > 0) offset = 0;                                  // 限制不超出左侧
    int delta = offset - m_scrollOffset;
    m_scrollOffset = offset;
    for (auto* tab : m_tabs) {
        Rect r = tab->GetRect();
        r.X += delta;
        tab->SetRect(r);
    }
    Invalidate();
}
```

### 标签交换（右键）

```cpp
void TabBar::SwapTabs(int from, int to) {
    if (from == to) return;

    // 1. 交换 HLayout 子控件列表中的位置
    Remove(m_tabs[from], false);
    Remove(m_tabs[to], false);
    Insert(to, m_tabs[from]);
    Insert(from, m_tabs[to]);

    // 2. 交换 m_tabs 指针
    std::swap(m_tabs[from], m_tabs[to]);

    // 3. 更新激活索引
    if (m_activeTab == from) m_activeTab = to;
    else if (m_activeTab == to) m_activeTab = from;

    UpdateTabCallbacks();
    RefreshLayout();
    Invalidate();
}
```

## 关键注意事项

### 1. 标签交换必须同时交换子控件位置和 m_tabs

只交换 `m_tabs` 指针但不交换 HLayout 子控件位置，布局不会更新。正确做法：先 `Remove` 两个再 `Insert` 回去互换位置，然后同步 `m_tabs`。

### 2. 不要用 `Remove` + `Insert` 交替交换

如果用 `Remove(b)→Insert(posA, b)→Remove(a)→Insert(posB, a)` 这种交替操作，中间状态索引可能错乱导致 `Control::Add` 中的 `ASSERT` 崩溃。正确做法：**先把两个都 Remove，再分别 Insert 到目标位置**。

### 3. 关闭按钮坐标计算

`GetCloseRect()` 返回的是相对于 TabButton 自身的坐标。在 TabBar 中判断是否点击关闭按钮时需要加上标签的 X 偏移：
```cpp
Rect closeRect = tab->GetCloseRect();
closeRect.X += tab->GetRect().X;
if (closeRect.Contains(arg.Location)) { /* 关闭 */ }
```

### 4. 标签宽度默认 150

- `TabBar::m_tabWidth` 控制新添加标签的默认宽度
- `TabButton` 构造函数中 `SetFixedWidth(150)` 是初始宽度
- htm `<tabbar tab-width="xxx">` 会覆盖默认值
- 已有标签的宽度不受 `tab-width` 变化影响

### 5. 事件触发时机

- `OnTabChanging` — 在 `SetActiveTab` 中最先触发，返回 `false` 可阻止切换
- `OnTabChanged` — 在 `SetActiveTab` 中切换成功后触发，参数为 `(oldIndex, newIndex)`
  - 如果 `index == m_activeTab` 则跳过，不触发任何事件
- `OnTabClosed` — 在 `RemoveTab` 中触发（在控件已删除、布局已刷新之后）
- `SwapTabs` 不触发事件

### 6. 最小宽度限制

所有宽度设置（TabBar::SetTabWidth、TabButton::SetButtonWidth、htm tab-width）都有最小 40 的限制。

### 7. 标签锁定

- `SetLocked(true)` 后关闭按钮自动隐藏，`RemoveTab` 跳过锁定标签
- 锁定标签仍然可以右键交换位置
- htm 中设置 `<tabbutton title="主页" locked="true">`

## TabButton 鼠标事件由 TabBar 代理

TabButton 自身 `SetHitTestVisible(false)`，所有鼠标事件（hover、点击、拖拽）都在 **TabBar** 的 `OnMouseMove/OnMouseDown/OnMouseUp/OnMouseLeave` 中处理。

因此 TabButton 的 `OnMouseEnter/OnMouseLeave` 虚函数不会被触发。hover 效果直接由 TabBar 设置 `Style.BackColor` 实现。
