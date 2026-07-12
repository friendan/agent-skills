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

## 功能归属

| 功能 | 位置 | 说明 |
|------|------|------|
| 布局结构 | htm | 控件树、容器嵌套 |
| 静态样式 | htm `<style>` | 背景色、文字颜色、字体、大小等 |
| 交互样式 | htm `<style>` | `:hover`、`:active`、`:checked` |
| 窗口行为 | htm `action` 属性 | `mini` / `max` / `close` / `title` / `move` |
| 业务逻辑 | C++ 代码 | 标签切换、导航、事件处理 |

## 内联样式

支持在标签上用 `style` 属性设置内联样式，优先级最高。

```html
<label id="title" style="background-color: rgb(50,50,60); color: rgb(200,200,200);"></label>
```

内联样式只对静态状态（`ControlState::Static`）生效，不适用于 hover/active 等交互状态。
