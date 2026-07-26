---
name: ezui-list-group
description: 指导如何使用EZUI框架的ListGroup列表组控件，包括htm属性、C++ API、事件冒泡机制、样式配置等。
---

# EZUI ListGroup 列表组控件

## 概述

`ListGroup` 列表组容器，管理多个 `ListGroupItem` 列表项。模仿 Bootstrap 5 的 list-group 组件，支持激活状态、禁用状态、颜色变体、flush 模式、自定义内容、点击事件等功能。

## 架构

```
ListGroup (VLayout)
├── ListGroupItem (VLayout) — 容器，可放置任意子控件
│   ├── [用户内容]（Label、hlayout、svg 等）
│   └── [caption]（可选，通过 text 属性自动创建 Label）
├── ListGroupItem
└── ...
```

## HTM 属性

### ListGroup 标签属性

```html
<list-group flush="true">
    <list-group-item text="项目1"></list-group-item>
</list-group>
```

| 属性 | 说明 | 示例 |
|------|------|------|
| `flush` | 去除边框和圆角 | `flush="true"` |

### ListGroupItem 标签属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `text` | 快捷设置文字，自动创建 Label | `text="项目1"` |
| `active` | 激活状态（蓝色高亮） | `active="true"` |
| `disabled` | 禁用状态（灰色不可点击） | `disabled="true"` |
| `kind` | 颜色变体（primary/success/danger 等） | `kind="danger"` |

### 完整示例

```html
<!-- 基本列表 -->
<list-group>
    <list-group-item text="项目1" active="true"></list-group-item>
    <list-group-item text="项目2"></list-group-item>
    <list-group-item text="项目3（禁用）" disabled="true"></list-group-item>
</list-group>

<!-- 颜色变体 -->
<list-group>
    <list-group-item text="Primary" kind="primary"></list-group-item>
    <list-group-item text="Danger" kind="danger"></list-group-item>
    <list-group-item text="Info" kind="info"></list-group-item>
</list-group>

<!-- Flush 模式 -->
<list-group flush="true">
    <list-group-item text="无边框项1"></list-group-item>
    <list-group-item text="无边框项2" active="true"></list-group-item>
</list-group>

<!-- 自定义内容 -->
<list-group>
    <list-group-item>
        <hlayout>
            <spacer width="12"></spacer>
            <svg width="24" height="24" bsicon="person"></svg>
            <spacer width="10"></spacer>
            <vbox>
                <label text="标题" font-size="14" style="font-weight: bold;"></label>
                <label text="描述" font-size="11" style="color: rgb(120,120,120);"></label>
            </vbox>
        </hlayout>
    </list-group-item>
</list-group>
```

## C++ API

### ListGroup

```cpp
ListGroupItem* AddItem(const UIString& text);  // 快捷创建带文字的项
ListGroupItem* GetItem(int index);
int GetItemCount();
void SetActiveItem(int index);   // 设置激活项（自动反激活之前的）
int GetActiveItem();
void SetFlush(bool flush);
```

### ListGroupItem

```cpp
void SetActive(bool active);
bool IsActive();
void SetDisabled(bool disabled);
bool IsDisabled();
void SetText(const UIString& text);   // 快捷设置文字

// 点击事件
std::function<void(ListGroupItem*)> OnClick;

// kind 颜色变体
void SetItemKind(const UIString& kind);
```

C++ 中使用示例：

```cpp
auto* group = (ListGroup*)FindControl(L"myGroup");
if (group) {
    // 添加新项
    auto* item = group->AddItem(L"新项目");
    item->SetActive(true);
    
    // 监听点击
    item->OnClick = [](ListGroupItem* sender) {
        OutputDebugString(L"点击了项目\n");
    };
}
```

## 实现原理

### 控件结构

`ListGroup` 继承 `VLayout`，`ListGroupItem` 继承 `VLayout`，每个 item 是一个容器，用户可放入任意子控件。

- `ListGroup` 管理 item 列表（`m_items`）和激活状态
- `ListGroupItem` 包含边框（默认 1px solid，bottom=0）、背景色
- `text` 属性自动创建内部 Label，`Margin.Left = 16` 留出内边距

### 激活/禁用状态切换

`SetActive`/`SetDisabled` 调用 `UpdateStyle()`，根据状态切换背景色、前景色、边框色：

- **正常**：白色背景、深色文字、灰色边框
- **激活**：蓝色背景（`#0d6efd`）、白色文字
- **禁用**：灰色背景、灰色文字
- **kind 颜色**：通过 `SetAttribute("kind")` 使用框架的 `ControlKind` 颜色体系

### 事件冒泡机制（核心）

为了解决子控件抢占鼠标事件的问题，框架层新增了 `m_eventBubble` 事件冒泡机制：

**改动点：**

1. `Control.h` — 新增 `m_eventBubble` 成员 + `SetEventBubble`/`GetEventBubble` + 虚方法 `OnChildEvent`
2. `Control.cpp` — 在 `OnMouseEnter`/`OnMouseLeave`/`OnMouseDown`/`OnMouseUp`/`OnMouseMove` 默认实现的末尾调用 `Parent->OnChildEvent(this, args)`
3. `OnChildEvent` 默认实现：检查 `GetEventBubble()` → 触发本控件的 `SendEvent(args)` → 继续向父控件冒泡

**ListGroupItem 中的使用：**

- 构造时调用 `SetEventBubble(true)`，让自己成为冒泡目标
- `Add` 方法调用 `Control::EnableEventBubbleRecursive(childCtl)` 递归设置所有子控件及后代控件的冒泡
- 子控件的鼠标事件自动冒泡到 `ListGroupItem`，触发 `EventHandler` 中的 hover/click 处理

**`EnableEventBubbleRecursive` 静态方法（`Control.h` 中声明，`Control.cpp` 中实现）：**

```cpp
static void Control::EnableEventBubbleRecursive(Control* ctl) {
    if (!ctl) return;
    ctl->SetEventBubble(true);
    for (auto& c : ctl->GetControls()) {
        if (!c->IsSpacer()) EnableEventBubbleRecursive(c);
    }
}
```

后续其他需要事件冒泡的控件（如 Carousel、Accordion）可以直接调用此方法，无需重复实现递归逻辑。

**冒泡流程：**

```
鼠标进入 label（在 hlayout 内）
  → Control::OnMouseEnter（label）
    → Parent->OnChildEvent(this, &args)  // Parent = hlayout
      → hlayout 的 m_eventBubble = true
        → SendEvent(args) → 触发 hlayout 的 EventHandler（如果有）
        → Parent->OnChildEvent(this, &args)  // Parent = ListGroupItem
          → ListGroupItem 的 m_eventBubble = true
            → SendEvent(args) → 触发 ListGroupItem 的 EventHandler
              → hover 变色
            → Parent->OnChildEvent(this, &args)  // Parent = ListGroup
              → ListGroup 的 m_eventBubble = false → 停止
```

## 踩坑记录

### 坑1：子控件收不到鼠标事件，导致 hover/click 无效

**问题**：`ListGroupItem` 内部的子控件（hlayout、label、svg 等）抢占了鼠标事件，`ListGroupItem` 收不到 `OnMouseEnter`/`OnMouseLeave`/`OnMouseDown`，hover 和点击无效。

**尝试过的方案：**

1. ❌ **在 ListGroupItem 上重写 `OnMouseEnter`/`OnMouseLeave` 虚函数** — 鼠标在子控件上时，事件只发给子控件，虚函数不会被调用
2. ❌ **在 Window.cpp 的消息分发中增加冒泡循环** — 需要修改框架核心，改动大且容易引发括号配对问题
3. ❌ **在子控件收到事件时主动调用父控件的冒泡方法** — 需要在每个子控件里加代码，不通用
4. ✅ **在 `Control` 的鼠标事件虚函数默认实现中调用 `Parent->OnChildEvent`** — 只要子控件不重写这些虚函数（或重写时调用 `__super`），事件就会冒泡到父控件

### 坑2：冒泡时需要防止递归

`OnChildEvent` 中调用了 `SendEvent(args)`，而 `SendEvent` 会触发 `EventHandler`，如果 `EventHandler` 中又触发了鼠标事件，可能导致无限递归。

**修复**：在 `Control` 中添加 `m_bubbling` 标志，`OnChildEvent` 开头检查，正在冒泡时直接返回。

### 坑3：冒泡需要递归设置所有后代控件

`ListGroupItem::Add` 中只对直接子控件设 `SetEventBubble(true)` 不够，子控件内部的后代控件（如 hlayout 内部的 label）没有开启冒泡。

**修复**：使用递归 lambda 遍历设置所有后代控件。后来抽取为 `Control::EnableEventBubbleRecursive` 静态方法，方便其他控件复用。

## 文件位置

- 头文件：`src/include/EzUI/ListGroup.h`
- 实现文件：`src/sources/ListGroup.cpp`
- 事件冒泡基础：`src/include/EzUI/Control.h` + `src/sources/Control.cpp`
- 测试 htm：`bin/ui/htm/listGroupBrowser.htm`
- 测试窗口：`src/demo/helloWorld/main.cpp` 中的 `ListGroupBrowser` 类
