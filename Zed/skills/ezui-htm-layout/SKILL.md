---
name: ezui-htm-layout
description: 指导如何用EZUI框架编写htm布局文件，包括样式定义、CSS选择器规则、iframe样式隔离等核心规则，避免常见的踩坑点。
---

# EZUI HTM 布局文件编写指南

## 样式定义

样式必须写在 htm 文件自带的 `<style>` 标签中，与控件标签**平级**：

```html
<vbox id="mainLayout">
    ...
</vbox>

<style>
    /* ID选择器 */
    #minBtn { background-color: rgb(50,50,60); }

    /* 伪类 :hover / :active / :checked */
    #minBtn:hover { background-color: rgb(80,80,90); }
    #minBtn:active { background-color: rgb(100,100,110); }

    /* 多选择器 */
    #minBtn, #maxBtn { color: rgb(200,200,200); }
</style>
```

## ❌ 常见错误

- 不要用 `LoadStyle()` 加载外部 `.css` 文件（路径解析不可靠）
- 不要用 `hover-style="..."` / `active-style="..."` 属性（EZUI 不支持）
- 不要把 `<style>` 放在容器标签内部（如 `<vbox>` 里面）
- 不要用C++代码设置 HoverStyle/ActiveStyle（会失去htm的灵活性）
- 不支持后代选择器（如 `#parent .child`），每个控件用ID直接定位
- **逗号分隔的ID+伪类选择器可能不生效**，如 `#btn1:hover, #btn2:hover`。改用类选择器：给控件加 `class="myClass"`，然后写 `.myClass:hover`

## 样式隔离

- `<iframe>` 加载的子页面，样式必须写在子页面自己的 `<style>` 中
- 父页面的样式不会穿透到 iframe 中
- 每个 htm 文件独立管理自己的样式

## 支持的CSS选择器

| 选择器 | 示例 | 说明 |
|--------|------|------|
| ID选择器 | `#minBtn` | 匹配 `id="minBtn"` 的控件 |
| 类选择器 | `.btn:hover` | 匹配 `class="btn"` 的控件 |
| 标签选择器 | `button` | 匹配所有 `<button>` 标签 |
| 属性选择器 | `[name=close]` | 匹配 `name="close"` 的控件 |
| 伪类 | `:hover` / `:active` / `:checked` / `:disabled` | 悬停/按下/选中/禁用状态 |
| 多选择器（静态） | `#minBtn, #maxBtn` | 逗号分隔，同时匹配多个。注意：**带伪类时建议用类选择器代替** |
| 滚动条 | `#list::-webkit-scrollbar` | 自定义滚动条样式 |

## C++ 自定义控件注意事项

### 需要绘制文字时继承 `Label` 而非 `Control`

`Control` 的 `OnForePaint` 中 `DrawString` 绘制文字可能不生效。如果需要自定义控件显示文字，直接继承 `Label`，用 `SetText` 设置文字，在 `OnForePaint` 中调用 `__super::OnForePaint(args)` 让基类绘制：

```cpp
class TabButton : public ezui::Label {
public:
    TabButton(Object* parent) : Label(parent) {
        SetText(L"标题");
    }
protected:
    void OnForePaint(PaintEventArgs& args) override {
        // 先设置文字颜色
        Style.ForeColor = Colors::TextColor();
        // 自定义背景绘制
        args.Graphics.SetColor(bgColor);
        args.Graphics.FillRectangle(GetRect());
        // 让Label绘制文字
        __super::OnForePaint(args);
        // 自定义前景绘制
    }
};
```

### 子控件必须 `Add` 到父控件

构造时传入父控件只会添加到 `m_childObjects`（生命周期管理），不会添加到 `m_controls`（布局和绘制）。必须显式调用 `parentControl->Add(childControl)`。

```cpp
TabButton* btn = new TabButton(this);  // 传入this，只管理生命周期
this->Add(btn);                         // 必须Add到m_controls，才能被绘制
```

## 功能归属

| 功能 | 位置 | 说明 |
|------|------|------|
| 布局结构 | htm | 控件树、容器嵌套 |
| 静态样式 | htm `<style>` | 背景色、文字颜色、字体、大小等 |
| 交互样式 | htm `<style>` | `:hover`、`:active`、`:checked` |
| 窗口行为 | htm `action` 属性 | `mini` / `max` / `close` / `title` / `move` |
| 业务逻辑 | C++ 代码 | 标签切换、导航、事件处理 |

## 布局撑满窗口

作为 `IFrame` 子控件的**根布局**（`vbox`/`vlayout`/`hlayout`），必须加 `dock="fill"` 才能撑满整个窗口：

```html
<vbox id="mainLayout" dock="fill">
```

## 模拟内边距（padding）

EZUI 的控件不支持 `padding` 属性。可以用容器 + `spacer` 模拟：

```html
<!-- 给 edit 添加左右内边距 10px -->
<hlayout id="urlBoxContainer" style="background-color: rgb(30,30,38); border-radius: 15px; border: 1px solid...">
    <spacer width="10"></spacer>
    <edit id="urlBox" style="background-color: rgba(0,0,0,0);"></edit>
    <spacer width="10"></spacer>
</hlayout>
```

把边框和圆角放到外层容器上，内部控件透明无边框，`spacer` 撑出间距。

## 内联样式

支持在标签上用 `style` 属性设置内联样式，优先级最高。

```html
<label id="title" style="background-color: rgb(50,50,60); color: rgb(200,200,200);"></label>
```

内联样式只对静态状态（`ControlState::Static`）生效，不适用于 hover/active 等交互状态。

## ⚠️ `action` 属性与 C++ EventHandler 冲突

htm 中设置了 `action="mini"`/`action="max"`/`action="close"` 的控件，**EzUI 框架已内置处理**对应行为（最小化/最大化/关闭），不要再用 C++ 代码绑定 `EventHandler`，否则会覆盖框架行为导致不生效。

```html
<!-- htm 中用 action 属性声明窗口行为 -->
<label id="maxBtn" action="max"></label>
```

```cpp
// ❌ C++ 中不要重复绑定，会覆盖 action 的框架处理
if (m_maxBtn) {
    m_maxBtn->EventHandler = [this](auto*, auto&) { OnMaximizeRestore(); };
}
```

## ⚠️ `hlayout` 宽度陷阱

**`hlayout` 嵌套 `hlayout` 时，内层的 `width="auto"` 会计算出 0 宽度**，导致子控件不可见。

```html
<!-- ❌ 内层 hlayout width="auto" 宽度为 0，主页标签不可见 -->
<hlayout id="tabBar">
    <hlayout id="tab0" class="tab" width="auto">
        <label id="tabTitle0" text="主页"></label>
        <label id="tabClose0" text="✕"></label>
    </hlayout>
</hlayout>

<!-- ✅ 给内层 hlayout 设置固定宽度 -->
<hlayout id="tabBar">
    <hlayout id="tab0" class="tab" width="80">
        <label id="tabTitle0" text="主页"></label>
        <label id="tabClose0" text="✕"></label>
    </hlayout>
</hlayout>

<!-- ✅ 或者直接用 label 代替内层 hlayout（label 的 width="auto" 正常） -->
<hlayout id="tabBar">
    <label id="tab0" class="tab" text="主页" width="auto"></label>
</hlayout>
```

注意：`label` 的 `width="auto"` 能正常包裹文字宽度，不受此限制。

## ⚠️ `hlayout` 背景色修改后需要 `Invalidate` 刷新

C++ 中修改 `hlayout` 的 `Style.BackColor` 后，**不会立即触发重绘**。需要调用父容器的 `Invalidate()` 强制刷新，否则背景色变化要等到鼠标划过触发 `:hover` 伪类时才显现。

```cpp
void SwitchTab(int index) {
    // 修改 hlayout 背景色
    m_tabs[index]->Style.BackColor = activeColor;
    
    // 必须强制刷新父容器，背景色立即生效
    auto* tabBar = FindControl(L"tabBar");
    if (tabBar) tabBar->Invalidate();
}
```

同时注意：**`hlayout` 的子控件如果设置了不透明背景色，会遮挡 `hlayout` 的背景变化**。需要把子控件背景设为透明（不设 `background-color` 或设为 `rgba(0,0,0,0)`），才能透出父容器的背景色。

## ⚠️ 标签拖拽排序的实现要点

EzUI 中实现标签拖拽排序的关键：

### 使用 `WndProc` 而非 `EventHandler` 处理鼠标拖拽

`EventHandler` 中的 `OnMouseMove`/`OnMouseUp` 事件在部分控件（如 `hlayout`）上可能不触发。可靠的方案是**重写 `Window::WndProc`** 虚方法，直接处理 Win32 鼠标消息：

```cpp
// 在头文件中声明
virtual LRESULT WndProc(UINT uMsg, WPARAM wParam, LPARAM lParam) override;

// 实现
LRESULT BrowserWindow::WndProc(UINT uMsg, WPARAM wParam, LPARAM lParam) {
    if (uMsg == WM_LBUTTONDOWN) {
        POINT pt = { GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam) };
        // 检测鼠标是否点击了标签，启动拖拽
    }
    if (uMsg == WM_MOUSEMOVE && m_dragging) {
        // 用 lParam 坐标 + GetRect 检测目标标签，高亮
    }
    else if (uMsg == WM_LBUTTONUP && m_dragging) {
        // 松开鼠标，执行交换
    }
    return BaseClass::WndProc(uMsg, wParam, lParam);
}
```

### 坐标转换

`WM_MOUSEMOVE` 的 `lParam` 是**窗口客户区坐标**。需要减去父容器（如 `tabBar`）的 `GetRect().X/Y` 得到相对坐标，然后用每个标签的 `GetRect()` 来判断鼠标在哪个标签区域内。

### 交互流程
1. `WM_LBUTTONDOWN` — 记录 `m_dragSrcIdx`，设置 `m_dragging = true`
2. `WM_MOUSEMOVE` — 遍历可见标签，用 `GetRect` 判断鼠标在哪个标签上，高亮目标标签
3. `WM_LBUTTONUP` — 执行 `SwapTabs(src, target)`，恢复所有标签颜色

### 颜色恢复
拖拽结束后需要恢复所有标签的颜色（激活的恢复为激活色，其他的恢复为非激活色），否则高亮色会残留。
