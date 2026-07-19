---
name: ezui-svgbox
description: 指导如何使用EZUI框架的SvgBox控件和BootstrapIcons/IconPark图标库，包括SVG控件使用、bsicon/iconpark属性、图标数据生成工具gen_icons等。
---

# EZUI SvgBox 控件 & 图标库集成

## 概述

`SvgBox` 是一个专门的 SVG 图标显示控件，底层使用 lunasvg 库渲染 SVG 矢量图。配合图标辅助类（`BootstrapIcons`、`IconParkIcons`），数千个图标已全部编译到库中，无需外部文件。

## 架构

```
htm: <svg bsicon="search"> 或 <svg iconpark="search">
         ↓
SvgBox::SetAttribute → BootstrapIcons::GetIcon / IconParkIcons::GetIcon
         ↓
从嵌入数据查表 → lunasvg::Document::loadFromData → 渲染到 D2D 位图
```

## 支持的图标库

| 图标库 | 数量 | htm 属性 | 数据文件 | 辅助类 |
|--------|------|----------|----------|--------|
| Bootstrap Icons | 2079 | `bsicon` | `BootstrapIconsData.h/cpp` | `BootstrapIcons` |
| IconPark | 2658 | `iconpark` | `IconParkIconsData.h/cpp` | `IconParkIcons` |

## HTM 属性

```html
<!-- Bootstrap Icons（推荐，完全内嵌） -->
<svg id="icon1" width="32" height="32" bsicon="search"></svg>

<!-- IconPark 图标 -->
<svg id="icon2" width="32" height="32" iconpark="user"></svg>

<!-- 从内存 SVG 数据加载 -->
<svg id="icon3" width="32" height="32" data="<svg>...</svg>"></svg>

<!-- 设置图标颜色（通过 kind 色调） -->
<svg id="icon4" kind="danger" width="32" height="32" bsicon="trash-fill"></svg>

<!-- 短名称也支持 -->
<svgbox id="icon5" width="32" height="32" bsicon="gear"></svgbox>
```

| 属性 | 说明 |
|------|------|
| `bsicon` | Bootstrap Icons 图标名（不带后缀和路径），如 `search`、`alarm-fill`、`9-circle` |
| `iconpark` | IconPark 图标英文名（不带后缀和路径），如 `search`、`user`、`home` |
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

## 图标辅助类

```cpp
// Bootstrap Icons（BootstrapIconsData.h/cpp 中 2079 个图标）
UIString svg = BootstrapIcons::GetIcon("search");

// IconPark（IconParkIconsData.h/cpp 中 2658 个图标）
UIString svg = IconParkIcons::GetIcon("search");
```

`SvgBox` 的 `bsicon` / `iconpark` 属性内部自动调用对应方法。

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

也可单独编译：
```bat
cmake -B build_clang_ninja_debug\tools\gen_icons -S src\tools\gen_icons -G Ninja ^
    -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl -DCMAKE_BUILD_TYPE=Debug
cmake --build build_clang_ninja_debug\tools\gen_icons --config Debug
```

### 用途

将 SVG 图标目录中的所有 `.svg` 文件嵌入到 C++ 代码中，生成 `XXXIconsData.h`、`XXXIconsData.cpp` 以及图标浏览器 htm 文件。

### 用法（新版，支持 --name 参数）

```bat
gen_icons_dbg.exe [--name 名称] <svg目录> <头文件输出目录> <cpp文件输出目录> [htm输出目录]
```

- `--name 名称` — 可选，指定数据文件名前缀和变量名前缀，默认 `BootstrapIcons`
- 第4个参数（htm输出目录）可选，指定后生成图标浏览器 htm 文件

示例（生成 Bootstrap Icons）：
```bat
gen_icons_dbg.exe bootstrap/bootstrap-icons-1.13.1 src/include/EzUI src/sources bin/ui/htm
```

示例（生成 IconPark）：
```bat
gen_icons_dbg.exe --name IconPark bootstrap/iconpark src/include/EzUI src/sources bin/ui/htm
```

### IconPark 文件名处理

IconPark 的 SVG 文件名格式为 `中文名_英文名.svg`（如 `搜索_search.svg`），gen_icons 工具会自动：
- 只取下划线后面的英文部分作为存储的图标 name（如 `search`）
- 不包含中文字符，避免源码编码问题

### 图标浏览器 htm

gen_icons 工具会按前缀分类生成图标浏览器 htm 文件：
- Bootstrap Icons → `iconBrowser.htm`
- IconPark → `IconParkIconsBrowser.htm`

浏览器按前缀分类展示所有图标（带分类标题行、每行8个图标），每个图标可点击复制名称到剪贴板：

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

### 生成的数据结构

所有图标库使用相同的数据结构，只是变量名不同：

```cpp
// IconParkIconsData.h（示例，名称由 --name 参数决定）
struct IconEntry {
    const wchar_t* name;  // 图标名
    const char* data;     // SVG XML 内容
};
extern const IconEntry g_iconParkIcons[2658];   // 变量名规则：g_ + 首字母小写的名称 + Icons
extern const int g_iconParkIconCount;            // 变量名规则：g_ + 首字母小写的名称 + IconCount
```

变量名生成规则：
- `BootstrapIcons` → `g_bootstrapIcons` / `g_bootstrapIconCount`
- `IconPark` → `g_iconParkIcons` / `g_iconParkIconCount`

### 添加新的 SVG 图标库

1. 将新的 SVG 图标文件夹放到项目中
2. 运行 gen_icons：
   ```bat
   gen_icons_dbg.exe --name 库名 <SVG目录> src/include/EzUI src/sources bin/ui/htm
   ```
3. 创建新的辅助类（仿照 `BootstrapIcons.h/cpp` 或 `IconParkIcons.h/cpp`）：
   ```cpp
   class 库名Icons {
   public:
       static UIString GetIcon(const UIString& name);
   };
   ```
4. 在 `SvgBox::SetAttribute` 中加对应的属性解析
5. 在 helloWorld 中加测试

## 第三方依赖

SvgBox 使用 **lunasvg 3.5.0**（MIT 协议）渲染 SVG：
- 源码位置：`src/3rd/lunasvg-3.5.0/`
- 包含 `plutovg` 子模块
- 静态编译进 EzUI，无 DLL 依赖
- 支持 SVG 1.1 大部分特性（路径、渐变、蒙版、文本等），不支持动画和滤镜

## 关键特性

1. **完全内嵌** — 两个图标库共 4700+ 图标全部编译到库中，无需外部文件
2. **按需渲染** — 控件每次绘制时实时渲染 SVG 到控件大小，自动缩放
3. **色调支持** — 通过 `kind` 属性或 `SetTintColor` 直接修改 SVG 颜色
4. **高性能** — lunasvg 每次 `OnPaint` 都重新渲染，适合图标这种小尺寸场景

## 踩坑记录

### 坑1：lunasvg 静态链接符号找不到

把 lunasvg 直接作为源码编译进 EzUI 时，需要在 CMakeLists.txt 中设置 `CXX_STANDARD 17`，并定义宏 `LUNASVG_BUILD`、`LUNASVG_BUILD_STATIC`、`PLUTOVG_BUILD`、`PLUTOVG_BUILD_STATIC`，否则头文件中的 `dllimport` 会导致链接错误。

### 坑2：SvgBox 的鼠标事件穿透

SVG 图标本身不需要响应鼠标事件。在 htm 中给 SVG 和文字标签加 `event="none"`，让鼠标事件穿透到父容器（`vlayout`）来触发 `action="copy"`。

### 坑3：htm 中布尔属性必须带值

EZUI 的 htm 解析器不支持无值属性（如 `<button disabled>`），必须写成 `disabled="true"` 才生效。

### 坑4：Window::OnCopyCompleted 必须 public

`OnCopyCompleted` 虚方法声明在 `Window` 中必须是 `public` 的，因为它在 `DefaultNotify` 全局函数中被调用，无法访问 `protected` 成员。

### 坑5：中文 SVG 文件名编码问题

IconPark 的文件名包含中文（如 `搜索_search.svg`），使用 `std::filesystem::path::stem().string()` 在 Windows 下返回的是 GBK 编码。解决方案：gen_icons 只取英文部分（下划线后面的内容）作为存储的 name 字段，避免在源码中出现中文宽字符字面量。

### 坑6：图标浏览器窗口使用 htm 布局更简单

不要尝试在 C++ 中动态创建大量图标控件，慢且代码复杂。改为用 `gen_icons` 工具直接生成完整的 `XXXIconsBrowser.htm` 文件，C++ 只需 `LoadXml` 加载即可。

### 坑7：gen_icons 输出目录路径

`gen_icons` 作为独立项目编译时，`CMAKE_SOURCE_DIR` 指向的是 `src/tools/gen_icons/` 而不是项目根目录。需要使用 `CMAKE_CURRENT_SOURCE_DIR` 相对路径 `../../../bin` 才能正确输出到 `trunk/bin`。
