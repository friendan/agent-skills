---
name: ezui-svgbox
description: 指导如何使用EZUI框架的SvgBox控件和BootstrapIcons图标库，包括SVG控件使用、bsicon属性、图标数据生成工具gen_icons等。
---

# EZUI SvgBox 控件 & Bootstrap Icons 图标库

## 概述

`SvgBox` 是一个专门的 SVG 图标显示控件，底层使用 lunasvg 库渲染 SVG 矢量图。配合 `BootstrapIcons` 类，2000+ Bootstrap 图标已全部编译到库中，无需外部文件。

## 架构

```
htm: <svg bsicon="search">
         ↓
SvgBox::SetAttribute("bsicon", "search")
         ↓
BootstrapIcons::GetIcon("search")  →  从嵌入数据查表
         ↓
SvgBox::OnPaint → lunasvg::Document::loadFromData → 渲染到 D2D 位图
```

## HTM 属性

```html
<!-- 从 Bootstrap Icons 加载（推荐，无需外部文件） -->
<svg id="icon1" width="32" height="32" bsicon="search"></svg>

<!-- 从内存 SVG 数据加载 -->
<svg id="icon2" width="32" height="32" data="<svg>...</svg>"></svg>

<!-- 设置图标颜色（色调） -->
<svg id="icon3" kind="danger" width="32" height="32" bsicon="trash-fill"></svg>

<!-- 短名称也支持 -->
<svgbox id="icon4" width="32" height="32" bsicon="gear"></svgbox>
```

| 属性 | 说明 |
|------|------|
| `bsicon` | Bootstrap Icons 图标名（不带后缀和路径），如 `search`、`alarm-fill`、`9-circle` |
| `src` | SVG 文件路径（不推荐，发布时不带文件） |
| `data` | 内联 SVG XML 数据 |
| `kind` | 图标色调（同 ControlKind，如 `danger`/`success`/`primary` 等） |

## C++ 接口

```cpp
class SvgBox : public Control {
    void LoadFromFile(const UIString& path);       // 从SVG文件加载
    void LoadFromData(const UIString& svgContent); // 从内存SVG数据加载
    void SetTintColor(const Color& color);          // 设置色调
    Color GetTintColor();
};
```

## Bootstrap Icons 类

所有 Bootstrap Icons SVG 数据编译在 `BootstrapIconsData.h/cpp` 中，通过 `BootstrapIcons::GetIcon` 按名查找。

```cpp
// 获取图标 SVG 内容（嵌入数据中查找）
UIString svg = BootstrapIcons::GetIcon("search");
// 返回 SVG XML 字符串，空表示未找到
```

`SvgBox` 的 `bsicon` 属性内部自动调用此方法。

## gen_icons 图标数据生成工具

### 位置

- 源码：`src/tools/gen_icons/gen_icons.cpp`
- CMakeLists：`src/tools/gen_icons/CMakeLists.txt`
- 编译输出：`bin/gen_icons_dbg.exe`（debug）

### 编译

默认不编译此工具。需要时在 CMake 配置中开启：

```bat
cmake -B build_clang_ninja_debug -S src -G Ninja ^
    -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl ^
    -DCMAKE_BUILD_TYPE=Debug -DBUILD_GEN_ICONS=ON
ninja -C build_clang_ninja_debug gen_icons
```

### 用途

将 SVG 图标目录中的所有 `.svg` 文件嵌入到 C++ 代码中，生成 `BootstrapIconsData.h`、`BootstrapIconsData.cpp` 以及图标浏览器 htm 文件。

### 用法

```bat
gen_icons_dbg.exe <svg目录> <头文件输出目录> <cpp文件输出目录> <htm输出目录>
```

第4个参数可选，指定后生成图标浏览器 htm 文件。

示例（重新生成 Bootstrap Icons 数据 + htm 浏览器）：
```bat
gen_icons_dbg.exe ../bootstrap/bootstrap-icons-1.13.1 ../src/include/EzUI ../src/sources ../bin/ui/htm
```

### 图标浏览器 htm

gen_icons 工具会生成 `iconBrowser.htm`，按前缀分类展示所有图标（带分类标题行、每行8个图标），每个图标可点击复制名称到剪贴板：

```html
<vlayout width="120" action="copy" copy-text="search">
    <svg width="28" height="28" event="none" data="..."></svg>
    <label text="search" class="icon-name" event="none"></label>
</vlayout>
```

- `action="copy"` — 点击后复制 `copy-text` 属性中的文本到剪贴板
- `event="none"` — 让鼠标事件穿透到父容器
- `cursor: pointer` — 鼠标移入变手型
- `vlayout:hover` — 悬停高亮背景

### 添加新的 SVG 图标库

1. 将新的 SVG 图标文件夹放到项目中
2. 运行 `gen_icons_dbg.exe <新SVG目录> <输出头文件.h目录> <输出cpp文件.cpp目录> <htm输出目录>`
3. 新建一个类（如 `MaterialIcons`）封装 `GetIcon` 接口，读取生成的 `g_materialIcons` 数据
4. 在目标控件的 `SetAttribute` 中加对应的属性解析（如 `mdicon="home"`）
5. **注意**：每次运行都会覆盖 `BootstrapIconsData.h/cpp`，不同图标库请使用不同的输出文件名

### 生成的数据结构

```cpp
// BootstrapIconsData.h
struct IconEntry {
    const wchar_t* name;  // 图标名（不带后缀）
    const char* data;     // SVG XML 内容
};
extern const IconEntry g_bootstrapIcons[2079];
extern const int g_bootstrapIconCount;
```

## 第三方依赖

SvgBox 使用 **lunasvg 3.5.0**（MIT 协议）渲染 SVG：
- 源码位置：`src/3rd/lunasvg-3.5.0/`
- 包含 `plutovg` 子模块
- 静态编译进 EzUI，无 DLL 依赖
- 支持 SVG 1.1 大部分特性（路径、渐变、蒙版、文本等），不支持动画和滤镜

## 关键特性

1. **完全内嵌** — 2079 个 Bootstrap Icons 全部编译到库中（约 2.3MB），用户无需附带任何 SVG 文件
2. **按需渲染** — 控件每次绘制时实时渲染 SVG 到控件大小，自动缩放
3. **色调支持** — 通过 `kind` 属性或 `SetTintColor` 直接修改 SVG 颜色
4. **高性能** — lunasvg 每次 `OnPaint` 都重新渲染，适合图标这种小尺寸场景；如果 SVG 特别复杂可考虑缓存 D2D 位图

## 踩坑记录

### 坑1：lunasvg 静态链接符号找不到

把 lunasvg 直接作为源码编译进 EzUI 时，需要在 CMakeLists.txt 中设置 `CXX_STANDARD 17`，并定义宏 `LUNASVG_BUILD`、`LUNASVG_BUILD_STATIC`、`PLUTOVG_BUILD`、`PLUTOVG_BUILD_STATIC`，否则头文件中的 `dllimport` 会导致链接错误。

### 坑2：SvgBox 的鼠标事件穿透

SVG 图标本身不需要响应鼠标事件。在 htm 中给 SVG 和文字标签加 `event="none"`，让鼠标事件穿透到父容器（`vlayout`）来触发 `action="copy"`。

### 坑3：htm 中布尔属性必须带值

EZUI 的 htm 解析器不支持无值属性（如 `<button disabled>`），必须写成 `disabled="true"` 才生效。

### 坑4：Window::OnCopyCompleted 必须 public

`OnCopyCompleted` 虚方法声明在 `Window` 中必须是 `public` 的，因为它在 `DefaultNotify` 全局函数中被调用，无法访问 `protected` 成员。

### 坑5：图标浏览器窗口使用 htm 布局更简单

不要尝试在 C++ 中动态创建 2000+ 个图标控件，慢且代码复杂。改为用 `gen_icons` 工具直接生成一个完整的 `iconBrowser.htm` 文件，C++ 只需 `LoadXml` 加载即可。
