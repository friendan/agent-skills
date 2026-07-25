---
name: ezui-accordion
description: 指导如何使用EZUI框架的Accordion手风琴控件，包括htm属性、C++ API、样式配置、展开/折叠事件等。
---

# EZUI Accordion 手风琴控件

## 概述

`Accordion` 手风琴容器，管理多个 `AccordionItem` 可折叠面板。模仿 Bootstrap 5 的 Accordion 组件，支持 single/multiple 模式、展开高亮、hover 变色、事件回调等功能。

## 架构

```
Accordion (VLayout)
├── AccordionItem (VLayout)
│   ├── header (HLayout) — 可点击切换展开/折叠
│   │   ├── m_headerLabel (Label) — 标题文字
│   │   ├── HSpacer(8) — 标题和图标间隔
│   │   └── m_chevronLabel (SvgBox) — 折叠图标
│   └── 内容子控件 — 用户通过 htm 添加的内容
├── AccordionItem
│   └── ...
└── ...
```

## HTM 属性

### Accordion 标签属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `mode` | 模式：`single`（默认，同一时间只展开一个）/ `multiple`（可同时展开多个） | `mode="multiple"` |
| `header-height` | 所有子面板的默认标题高度（可被 item 的 `header-height` 覆盖） | `header-height="30"` |

### AccordionItem 标签属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `title` | 面板标题文字 | `title="面板1"` |
| `active` | 是否展开：`true` / `false` | `active="true"` |
| `disabled` | 是否禁用：`true` / `false`（禁用时强制展开、隐藏图标、不可点击） | `disabled="true"` |
| `content-padding` | 内容区域内边距，格式同 CSS padding：`all` / `tb lr` / `t r b l` | `content-padding="16 20"` |
| `header-align` | 标题对齐：`left`（默认）/ `center` / `right` | `header-align="center"` |
| `header-height` | 当前面板标题高度（覆盖 accordion 默认值） | `header-height="60"` |
| `header-color` | 标题文字颜色 | `header-color="red"` |
| `header-style` | 应用到 header 的 CSS 样式（背景色、边框等） | `header-style="background-color: #eee;"` |

### 完整示例

```html
<accordion mode="single" header-height="30">
    <accordion-item title="面板1" active="true">
        <vbox>
            <label text="内容1"></label>
        </vbox>
    </accordion-item>
    <accordion-item title="面板2" disabled="true">
        <vbox>
            <label text="禁用面板，强制展开且不可点击"></label>
        </vbox>
    </accordion-item>
    <accordion-item title="面板3" header-align="center" header-color="#e67e22">
        <vbox>
            <label text="居中标题 + 橙色文字"></label>
        </vbox>
    </accordion-item>
</accordion>
```

## C++ API

### Accordion

| 方法 | 说明 |
|------|------|
| `SetMode(const UIString& mode)` | 设置模式 |
| `IsMultipleMode()` | 是否 multiple 模式 |
| `ToggleItem(AccordionItem* item)` | 切换指定面板 |
| `SetDefaultHeaderHeight(int height)` | 设置默认标题高度 |
| `GetActiveItem()` | 获取当前展开的面板（single 模式下返回唯一展开项） |

### AccordionItem

| 方法 | 说明 |
|------|------|
| `SetTitle(const UIString& text)` | 设置标题 |
| `GetTitle()` | 获取标题 |
| `IsActive()` | 是否展开 |
| `IsDisabled()` | 是否禁用 |
| `ApplyDefaultHeaderHeight(int height)` | 继承父 Accordion 的默认标题高度 |

#### 事件回调

```cpp
item->OnExpandChanged = [](AccordionItem* item, bool expanded) {
    if (expanded)
        OutputDebugString(L"面板展开\n");
    else
        OutputDebugString(L"面板折叠\n");
};
```

#### 颜色配置（公开成员变量）

```cpp
item->HeaderBackColor = Color(248, 249, 250);           // 折叠时背景色
item->HeaderActiveBackColor = Color(207, 226, 255);     // 展开时背景色
item->HeaderHoverBackColor = Color(233, 236, 239);      // 折叠时 hover 背景色
item->HeaderActiveHoverBackColor = Color(180, 210, 240); // 展开时 hover 背景色
```

## 实现原理

### 创建面板

`AccordionItem` 构造时：
1. 创建 `HLayout` 作为 header，设置固定高度
2. 添加 `Label` 作为标题文字
3. 添加 `HSpacer(8)` 保证标题和图标之间的间距
4. 添加 `SvgBox` 作为折叠图标（使用 `SetTintColor` 设色，而非 Label 的前景色继承）
5. header 的 `EventHandler` 处理点击事件，通知 `Accordion::ToggleItem`

### 展开/折叠逻辑

`Accordion::ToggleItem` 执行：
1. **single 模式**下，先折叠其他面板
2. 切换目标面板的 `m_active` 状态
3. 同步内容子控件的 `SetVisible`
4. 切换折叠图标（`arrow-down-circle` ↔ `arrow-up-circle`）
5. 切换 header 背景色 + hover 背景色
6. 展开时重新计算高度（先撑大到 2000，布局后按内容计算实际高度）

### 与 htm 解析的交互

htm 解析顺序：先 `BuildControl`（调用 `SetAttribute`）→ 再 `LoadControl`（添加子控件）。

`AccordionItem::Add` 被重写，添加内容子控件时会根据 `m_active` 同步 `SetVisible` 并重新计算展开高度。这解决了 `active="true" disabled="true"` 等属性在子控件添加前就设置好的场景。

### header 背景色管理

展开/折叠时同时设置 `Style.BackColor` 和 `HoverStyle.BackColor`，确保：
- 鼠标移出时显示正确的状态色
- 鼠标悬停时显示对应状态的 hover 色（展开时浅蓝变深，折叠时浅灰变深）

## 踩坑记录

### 坑1：Label 图标颜色为黑色

**问题**：`m_chevronLabel` 用 `Label` + `bsicon` 属性显示折叠图标，图标显示为黑色。

**原因**：`OnIconPaint` 中图标色调取自 `GetForeColor()`，而 `GetForeColor()` 有继承性，`m_chevronLabel` 未设前景色时返回 `Color(0)`，条件 `foreColor.GetValue() != 0` 不成立，不上色。

**修复**：改用 `SvgBox` 控件，通过 `SetTintColor(Color(33, 37, 41))` 直接设色，色调逻辑与 `iconBrowser.htm` 一致。

### 坑2：active 属性的面板内容不可见

**问题**：`active="true" disabled="true"` 的面板打开时看不到内容。

**原因**：htm 解析时 `SetAttribute("active")` 和 `SetAttribute("disabled")` 先执行，但此时子控件（`<vbox>` 内的 `<label>`）还未添加。子控件通过 `LoadControl` 后续添加时，`Add` 中没有同步 `SetVisible`。

**修复**：重写 `AccordionItem::Add`，添加内容子控件时根据 `m_active` 同步 `SetVisible(m_active)`，并重新计算展开高度。

### 坑3：点击时 header 颜色还是 hover 色

**问题**：点击切换展开/折叠后，如果鼠标还在 header 上，显示的是 hover 色而非激活色。

**原因**：框架的 HoverStyle 在鼠标悬停时会覆盖 `Style`。

**修复**：切换状态时同步设置 `HoverStyle.BackColor`，展开时设为 `HeaderActiveBackColor`，折叠时设为 `HeaderHoverBackColor`。

### 坑4：m_defaultHeaderHeight 默认值残留

`Accordion.h` 中 `m_defaultHeaderHeight = 44` 和 `m_headerHeight = 25` 可能不一致。建议两者保持相同默认值，避免父 Accordion 未设置 `header-height` 时子面板继承值与自身默认值不同。

### 坑5：GetActiveItem 不能是 const 方法

`Accordion::GetActiveItem()` 中调用了 `GetControls()`，而 `GetControls()` 不是 const 方法，所以 `GetActiveItem` 不能声明为 const。

## 文件位置

- 头文件：`src/include/EzUI/Accordion.h`
- 实现文件：`src/sources/Accordion.cpp`
- 测试 htm：`bin/ui/htm/accordionBrowser.htm`
- 测试窗口：`src/demo/helloWorld/main.cpp` 中的 `AccordionBrowser` 类
