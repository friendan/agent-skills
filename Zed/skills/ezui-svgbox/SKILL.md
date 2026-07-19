---
name: ezui-svgbox
description: 指导如何使用EZUI框架的SvgBox控件和BootstrapIcons/IconPark/Lucide/Tabler/RemixIcon/Twemoji图标库，包括SVG控件使用、bsicon/iconpark/lucide/tabler-filled/tabler-outline/remixicon/twicon属性、图标数据生成工具gen_icons、控件图标集成等。
---

# EZUI SvgBox 控件 & 图标库集成

## 概述

`SvgBox` 是一个专门的 SVG 图标显示控件，底层使用 lunasvg 库渲染 SVG 矢量图。配合图标辅助类，16000+ 图标已全部编译到库中，无需外部文件。同时，Label/Button 等控件也支持直接设置图标属性，在文字旁显示 SVG 图标。

## 架构

```
SvgBox: <svg bsicon="search"> → SvgBox::SetAttribute → 辅助类::GetIcon → lunasvg → 渲染

控件图标: <button bsicon="save"> → Control::SetAttribute → 存储IconData → Label::OnForePaint → OnIconPaint → lunasvg → 绘制
```

## 支持的图标库

| 图标库 | 数量 | htm 属性 | 辅助类 |
|--------|------|----------|--------|
| Bootstrap Icons | 2079 | `bsicon` | `BootstrapIcons` |
| IconPark | 2658 | `iconpark` | `IconParkIcons` |
| Lucide Icons | 1748 | `lucide` | `LucideIcons` |
| Tabler Icons (Outline) | 5112 | `tabler-outline` | `TablerOutlineIcons` |
| Tabler Icons (Filled) | 1054 | `tabler-filled` | `TablerFilledIcons` |
| Remix Icon | 3229 | `remixicon` | `RemixIconIcons` |
| Twemoji | 3689 | `twicon` | `TwemojiIcons` |

**总计：7 个库，约 19569 个图标**

## HTM 属性

### SvgBox 专用属性

```html
<!-- 各种图标库的 SVG 独立显示 -->
<svg width="32" height="32" bsicon="search"></svg>
<svg width="32" height="32" iconpark="user"></svg>
<svg width="32" height="32" lucide="house"></svg>
<svg width="32" height="32" tabler-outline="home"></svg>
<svg width="32" height="32" tabler-filled="home"></svg>
<svg width="32" height="32" remixicon="arrow-down-fill"></svg>
<svg width="32" height="32" twicon="1f600"></svg>

<!-- 从内存 SVG 数据加载 -->
<svg width="32" height="32" data="<svg>...</svg>"></svg>

<!-- 设置图标颜色（通过 kind 色调） -->
<svg kind="danger" width="32" height="32" bsicon="trash-fill"></svg>
```

### 控件图标属性（Label/Button 等文字控件直接使用）

```html
<!-- 图标默认在文字左边 -->
<button text="保存" kind="primary" bsicon="save"></button>
<button text="编辑" kind="success" lucide="pencil"></button>
<label text="搜索" kind="info" bsicon="search"></label>

<!-- 图标在文字右边 -->
<button text="下一步" kind="primary" bsicon="arrow-right" icon-side="right"></button>

<!-- 调整图标大小和间距 -->
<button text="保存" bsicon="save" icon-size="20" icon-gap="6"></button>
```

| 属性 | 说明 |
|------|------|
| `bsicon` | Bootstrap Icons 图标名 |
| `iconpark` | IconPark 图标英文名 |
| `lucide` | Lucide Icons 图标名 |
| `tabler-outline` | Tabler Icons Outline 风格 |
| `tabler-filled` | Tabler Icons Filled 风格 |
| `remixicon` | Remix Icon 图标名（含 -fill/-line 风格后缀） |
| `twicon` | Twemoji 图标名（Unicode 码点，如 1f600） |
| `icon-size` | 图标大小，默认 16 |
| `icon-gap` | 图标与文字间距，默认 4 |
| `icon-side` | 图标位置，`left`（默认）或 `right` |

### SvgBox 额外属性

| 属性 | 说明 |
|------|------|
| `src` | SVG 文件路径（不推荐，发布时不带文件） |
| `data` | 内联 SVG XML 数据 |
| `kind` | 图标色调（同 ControlKind） |

## 实现原理

### IconData 数据结构（Control.h）

```cpp
struct IconData {
    UIString Lib;       // 图标库，如 "bsicon" / "lucide"
    UIString Name;      // 图标名，如 "search" / "home"
    int Size = 16;      // 图标大小
    int Gap = 4;        // 图标与文字的间距
    bool RightSide = false; // false=左边，true=右边
    bool Visible = false;
};
```

### 图标数据存储

在 `Control` 基类中增加 `m_icon`（protected），所有子控件自动继承。

### SetAttribute 解析

`Control::SetAttribute` 中统一解析图标属性（`bsicon`/`iconpark`/`lucide`/`tabler-outline`/`tabler-filled`/`remixicon`/`twicon`）以及控制属性（`icon-size`/`icon-gap`/`icon-side`），存储到 `m_icon`。

### 绘制流程

1. `Label::OnForePaint` 中判断 `m_icon.Visible`
2. 调用 `OnIconPaint` 绘制 SVG 图标，返回图标占用的宽度
3. 文字绘制时根据 `iconWidth` 和 `icon-side` 偏移文字起始位置
4. 图标库匹配在 `OnIconPaint` 中根据 `m_icon.Lib` 字符串分发到对应的 `GetIcon` 方法

### 可见的视觉效果

- 图标自动使用控件的 `GetForeColor()` 作为色调
- 图标垂直居中
- 图标在左时文字右移避让，在右时文字不动
- 支持所有 6 个图标库，零额外代码

### 影响范围

- 不改动旧控件的绘制逻辑：`m_icon.Visible` 默认 false，不设图标属性的控件不受影响
- Label 子类（Button 等）自动获得图标支持
- 其他文字控件（TabButton 等）如需图标，可在自己的 `OnForePaint` 中调用 `OnIconPaint`

## C++ 接口

```cpp
class Control {
    // 图标绘制（供子控件在绘制文字时调用，返回图标占宽）
    int OnIconPaint(PaintEventArgs& e);

    // 图标数据访问
    IconData& GetIcon();
    void SetIcon(const UIString& lib, const UIString& name);
};
```

## 图标辅助类

```cpp
UIString svg = BootstrapIcons::GetIcon("search");
UIString svg = IconParkIcons::GetIcon("user");
UIString svg = LucideIcons::GetIcon("house");
UIString svg = TablerOutlineIcons::GetIcon("home");
UIString svg = TablerFilledIcons::GetIcon("home");
UIString svg = RemixIconIcons::GetIcon("arrow-down-fill");
UIString svg = TwemojiIcons::GetIcon("1f600");
```

SvgBox 和 Control::OnIconPaint 内部自动调用对应方法。

## gen_icons 图标数据生成工具

### 位置

- 源码：`src/tools/gen_icons/gen_icons.cpp`
- CMakeLists：`src/tools/gen_icons/CMakeLists.txt`
- 编译输出：`bin/gen_icons_dbg.exe`（debug）

### 编译

```bat
cmake -B build_clang_ninja_debug -S src -G Ninja ^
    -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl ^
    -DCMAKE_BUILD_TYPE=Debug -DBUILD_GEN_ICONS=ON
ninja -C build_clang_ninja_debug gen_icons
```

也可单独编译：
```bat
cmake -B build_clang_ninja_debug\tools\gen_icons -S src\tools\gen_icons -G Ninja ^
    -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl -DCMAKE_BUILD_TYPE=Debug
cmake --build build_clang_ninja_debug\tools\gen_icons --config Debug
```

### 用法

```bat
gen_icons_dbg.exe <名称> <svg目录> <头文件输出目录> <cpp文件输出目录> [htm输出目录]
```

示例：
```bat
gen_icons_dbg.exe BootstrapIcons bootstrap/bootstrap-icons-1.13.1 src/include/EzUI src/sources
gen_icons_dbg.exe TablerOutline bootstrap/tabler-icons-3.45.0/icons/outline src/include/EzUI src/sources bin/ui/htm
gen_icons_dbg.exe TablerFilled bootstrap/tabler-icons-3.45.0/icons/filled src/include/EzUI src/sources bin/ui/htm
gen_icons_dbg.exe RemixIcon bootstrap/RemixIcon_Svg_v4.9.1 src/include/EzUI src/sources bin/ui/htm
gen_icons_dbg.exe Twemoji bootstrap/twemoji-14.0.2/svg src/include/EzUI src/sources bin/ui/htm
```

### 递归扫描

gen_icons 使用 `fs::recursive_directory_iterator` 扫描 SVG 目录，支持多级子文件夹结构。

### 文件名处理

- **Bootstrap Icons**：`search.svg` → `search`
- **IconPark**：`中文名_英文名.svg` → 取英文部分
- **Lucide / Tabler**：`home.svg` → `home`
- **RemixIcon**：`arrow-down-fill.svg` → `arrow-down-fill`
- **Twemoji**：`1f600.svg` → `1f600`（Unicode 码点，直接取文件名）

### 图标浏览器 htm

gen_icons 工具会生成 `XXXIconsBrowser.htm` 文件，按前缀分类展示所有图标，可点击复制名称到剪贴板：

```html
<vlayout width="120" action="copy" copy-text="search">
    <svg width="28" height="28" event="none" data="..."></svg>
    <label text="search" class="icon-name" event="none"></label>
</vlayout>
```

### 生成的数据结构

```cpp
struct IconEntry {
    const wchar_t* name;
    const char* data;
};
extern const IconEntry g_remixIconIcons[3229];
extern const int g_remixIconIconCount;
```

变量名规则：`g_` + 首字母小写的名称 + `Icons` / `IconCount`

### 添加新的 SVG 图标库

1. 将 SVG 文件夹放到 `bootstrap/` 下
2. 运行 `gen_icons_dbg.exe 库名 <SVG目录> src/include/EzUI src/sources bin/ui/htm`
3. 创建辅助类（仿照现有实现）
4. 在 `SvgBox::SetAttribute` 中加属性解析
5. 在 `Control::OnIconPaint` 中加对应的图标库分支
6. 如果是分风格（如 filled/outline），分别运行两次 gen_icons

## 第三方依赖

SvgBox 使用 **lunasvg 3.5.0**（MIT 协议）渲染 SVG，静态编译进 EzUI，无 DLL 依赖。

## 关键特性

1. **完全内嵌** — 7 个图标库共 19569 个图标，用户无需附带任何文件
2. **按需渲染** — 每次绘制时实时渲染 SVG，自动缩放
3. **色调支持** — 通过 `kind` 属性或 `SetTintColor` 直接修改 SVG 颜色
4. **控件图标** — Label/Button 等直接设图标属性，自动在文字旁显示，影响文字布局

## 踩坑记录

### 坑1：lunasvg 静态链接符号找不到
需定义 `LUNASVG_BUILD`、`LUNASVG_BUILD_STATIC`、`PLUTOVG_BUILD`、`PLUTOVG_BUILD_STATIC`。

### 坑2：SvgBox 的鼠标事件穿透
给 SVG 和文字标签加 `event="none"`。

### 坑3：htm 中布尔属性必须带值
不支持无值属性，如 `disabled="true"`。

### 坑4：Window::OnCopyCompleted 必须 public

### 坑5：中文 SVG 文件名编码问题
IconPark 文件名包含中文，gen_icons 只取英文部分。

### 坑6：图标浏览器使用 htm 布局更简单

### 坑7：gen_icons 输出目录路径
独立编译时 `CMAKE_SOURCE_DIR` 不对，需用 `CMAKE_CURRENT_SOURCE_DIR/../../../bin`。

### 坑8：分风格图标库的处理
分别运行两次 gen_icons，创建独立的辅助类和属性。

### 坑9：多级子目录的递归扫描
gen_icons 使用 `recursive_directory_iterator`。

### 坑10：m_icon 必须是 protected
子类（Label 等）需要直接访问 `m_icon` 的 Visible/RightSide 等字段来判断文字偏移，因此 `m_icon` 不能是 private。

### 坑11：OnIconPaint 中图标色调
图标使用控件的 `GetForeColor()` 作为渲染色调，确保与当前控件状态（normal/hover/active）的文字颜色一致。
