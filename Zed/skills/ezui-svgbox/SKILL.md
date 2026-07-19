---
name: ezui-svgbox
description: 指导如何使用EZUI框架的SvgBox控件和BootstrapIcons/IconPark/Lucide/Tabler/RemixIcon图标库，包括SVG控件使用、bsicon/iconpark/lucide/tabler-filled/tabler-outline/remixicon属性、图标数据生成工具gen_icons等。
---

# EZUI SvgBox 控件 & 图标库集成

## 概述

`SvgBox` 是一个专门的 SVG 图标显示控件，底层使用 lunasvg 库渲染 SVG 矢量图。配合图标辅助类，16000+ 图标已全部编译到库中，无需外部文件。

## 架构

```
htm: <svg bsicon="search"> 或 <svg iconpark="search"> 或 <svg lucide="search">
         ↓
SvgBox::SetAttribute → 对应辅助类::GetIcon
         ↓
从嵌入数据查表 → lunasvg::Document::loadFromData → 渲染到 D2D 位图
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

**总计：6 个库，约 15880 个图标**

## HTM 属性

```html
<!-- Bootstrap Icons -->
<svg width="32" height="32" bsicon="search"></svg>

<!-- IconPark -->
<svg width="32" height="32" iconpark="user"></svg>

<!-- Lucide Icons -->
<svg width="32" height="32" lucide="house"></svg>

<!-- Tabler Icons Outline -->
<svg width="32" height="32" tabler-outline="home"></svg>

<!-- Tabler Icons Filled -->
<svg width="32" height="32" tabler-filled="home"></svg>

<!-- Remix Icon（文件名自带 -fill/-line 后缀） -->
<svg width="32" height="32" remixicon="arrow-down-fill"></svg>
<svg width="32" height="32" remixicon="arrow-down-line"></svg>

<!-- 从内存 SVG 数据加载 -->
<svg width="32" height="32" data="<svg>...</svg>"></svg>

<!-- 设置图标颜色（通过 kind 色调） -->
<svg kind="danger" width="32" height="32" bsicon="trash-fill"></svg>

<!-- 短名称也支持 -->
<svgbox width="32" height="32" bsicon="gear"></svgbox>
```

| 属性 | 说明 |
|------|------|
| `bsicon` | Bootstrap Icons 图标名 |
| `iconpark` | IconPark 图标英文名 |
| `lucide` | Lucide Icons 图标名 |
| `tabler-outline` | Tabler Icons Outline 风格 |
| `tabler-filled` | Tabler Icons Filled 风格 |
| `remixicon` | Remix Icon 图标名（含 -fill/-line 风格后缀） |
| `src` | SVG 文件路径（不推荐，发布时不带文件） |
| `data` | 内联 SVG XML 数据 |
| `kind` | 图标色调（同 ControlKind） |

## C++ 接口

```cpp
class SvgBox : public Control {
    void LoadFromFile(const UIString& path);
    void LoadFromData(const UIString& svgContent);
    void SetTintColor(const Color& color);
    Color GetTintColor();
};
```

## 图标辅助类（用法一致）

```cpp
UIString svg = BootstrapIcons::GetIcon("search");
UIString svg = IconParkIcons::GetIcon("user");
UIString svg = LucideIcons::GetIcon("house");
UIString svg = TablerOutlineIcons::GetIcon("home");
UIString svg = TablerFilledIcons::GetIcon("home");
UIString svg = RemixIconIcons::GetIcon("arrow-down-fill");
```

`SvgBox` 的对应属性内部自动调用对应方法。

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

第4个参数（htm输出目录）可选，指定后生成图标浏览器 htm 文件。

示例：
```bat
gen_icons_dbg.exe BootstrapIcons bootstrap/bootstrap-icons-1.13.1 src/include/EzUI src/sources
gen_icons_dbg.exe TablerOutline bootstrap/tabler-icons-3.45.0/icons/outline src/include/EzUI src/sources bin/ui/htm
gen_icons_dbg.exe TablerFilled bootstrap/tabler-icons-3.45.0/icons/filled src/include/EzUI src/sources bin/ui/htm
gen_icons_dbg.exe RemixIcon bootstrap/RemixIcon_Svg_v4.9.1 src/include/EzUI src/sources bin/ui/htm
```

### 递归扫描

gen_icons 使用 `fs::recursive_directory_iterator` 扫描 SVG 目录，支持多级子文件夹结构（如 RemixIcon 的分类文件夹），无需手动合并文件。

### 文件名处理

- **Bootstrap Icons**：文件名如 `search.svg`，直接使用
- **IconPark**：文件名 `中文名_英文名.svg`，自动取下划线后面的英文部分
- **Lucide / Tabler**：直接使用文件名作为 name
- **RemixIcon**：文件名如 `arrow-down-fill.svg`，直接使用（包含 -fill/-line 风格后缀）

### 图标浏览器 htm

gen_icons 工具会生成 `XXXIconsBrowser.htm` 文件，按前缀分类展示所有图标，每个可点击复制名称到剪贴板：

```html
<vlayout width="120" action="copy" copy-text="search">
    <svg width="28" height="28" event="none" data="..."></svg>
    <label text="search" class="icon-name" event="none"></label>
</vlayout>
```

### 生成的数据结构

```cpp
struct IconEntry {
    const wchar_t* name;  // 图标名
    const char* data;     // SVG XML 内容
};
extern const IconEntry g_remixIconIcons[3229];
extern const int g_remixIconIconCount;
```

变量名规则：`g_` + 首字母小写的名称 + `Icons` / `IconCount`
- `TablerOutline` → `g_tablerOutlineIcons` / `g_tablerOutlineIconCount`
- `RemixIcon` → `g_remixIconIcons` / `g_remixIconIconCount`

### 添加新的 SVG 图标库

1. 将 SVG 文件夹放到 `bootstrap/` 下
2. 运行 `gen_icons_dbg.exe 库名 <SVG目录> src/include/EzUI src/sources bin/ui/htm`
3. 创建辅助类（仿照现有实现）：
   ```cpp
   class 库名Icons {
   public:
       static UIString GetIcon(const UIString& name);
   };
   ```
4. 在 `SvgBox::SetAttribute` 中加属性解析
5. 如果是分风格的图标库（如 Tabler 的 filled/outline），分别运行两次 gen_icons，创建两个辅助类和两个属性
6. 如果是多级子目录的图标库（如 RemixIcon 的分类文件夹），gen_icons 会自动递归扫描

## 第三方依赖

SvgBox 使用 **lunasvg 3.5.0**（MIT 协议）渲染 SVG：
- 源码位置：`src/3rd/lunasvg-3.5.0/`
- 包含 `plutovg` 子模块
- 静态编译进 EzUI，无 DLL 依赖

## 关键特性

1. **完全内嵌** — 6 个图标库共 15880+ 图标全部编译到库中，用户无需附带任何文件
2. **按需渲染** — 控件每次绘制时实时渲染 SVG 到控件大小，自动缩放
3. **色调支持** — 通过 `kind` 属性或 `SetTintColor` 直接修改 SVG 颜色

## 踩坑记录

### 坑1：lunasvg 静态链接符号找不到
需定义宏 `LUNASVG_BUILD`、`LUNASVG_BUILD_STATIC`、`PLUTOVG_BUILD`、`PLUTOVG_BUILD_STATIC`。

### 坑2：SvgBox 的鼠标事件穿透
给 SVG 和文字标签加 `event="none"`，让鼠标事件穿透到父容器触发 `action="copy"`。

### 坑3：htm 中布尔属性必须带值
不支持无值属性，必须写成 `disabled="true"`。

### 坑4：Window::OnCopyCompleted 必须 public
因为它在 `DefaultNotify` 中被调用，`protected` 不可访问。

### 坑5：中文 SVG 文件名编码问题
IconPark 文件名包含中文，gen_icons 只取英文部分避免编码问题。

### 坑6：图标浏览器窗口使用 htm 布局更简单
用 gen_icons 工具直接生成完整的 htm 文件，C++ 只需 `LoadXml` 加载。

### 坑7：gen_icons 输出目录路径
独立编译时 `CMAKE_SOURCE_DIR` 不对，需用 `CMAKE_CURRENT_SOURCE_DIR/../../../bin`。

### 坑8：分风格图标库的处理
对于同一个图标库有不同风格（如 Tabler 的 filled/outline），分别运行两次 gen_icons 生成独立数据文件，创建独立的辅助类和 htm 属性。

### 坑9：多级子目录的递归扫描
gen_icons 使用 `recursive_directory_iterator` 支持递归子目录。若添加的图标库存在多级文件夹结构（如 RemixIcon 的分类文件夹），无需额外处理。
