---
name: ezui-carousel
description: 指导如何使用EZUI框架的Carousel轮播控件，包括htm属性、C++ API、自动播放、控制按钮、caption配置、踩坑记录等。
---

# EZUI Carousel 轮播控件

## 概述

`Carousel` 轮播容器，管理多个 `CarouselItem` 轮播项。模仿 Bootstrap 5 的 Carousel 组件，支持自动轮播、控制按钮、caption、事件回调等功能。

## 架构

```
Carousel (VLayout)
├── CarouselItem (VLayout) — 轮播项（只显示当前项）
│   ├── 用户内容（任意子控件）
│   └── captionBar (HLayout) — 底部标题栏（可选）
│       ├── captionTitle (Label)
│       └── captionText (Label)
├── CarouselItem
│   └── ...
└── controlBar (HLayout) — 底部控制栏（可选，固定高度30px）
    ├── Spacer — 左边弹性撑开
    ├── firstBtn (Label) — |<
    ├── HSpacer(10) — 间距
    ├── prevBtn (Label) — <
    ├── HSpacer(10)
    ├── nextBtn (Label) — >
    ├── HSpacer(10)
    ├── lastBtn (Label) — >|
    └── Spacer — 右边弹性撑开
```

## HTM 属性

### Carousel 标签属性

```html
<carousel interval="3000" ride="true" wrap="true" controls="true" controls-kind="info" controls-gap="10" dock="fill" height="300">
    <carousel-item>...</carousel-item>
</carousel>
```

| 属性 | 说明 | 示例 |
|------|------|------|
| `interval` | 自动轮播间隔(ms)，0 或负数禁用 | `interval="3000"` |
| `ride` | 是否自动开始播放 | `ride="true"` |
| `wrap` | 是否循环播放 | `wrap="true"` |
| `pause` | 悬停时暂停 | `pause="true"`（默认） |
| `controls` | 是否显示底部控制按钮 | `controls="true"` |
| `controls-kind` | 控制按钮的 kind 风格 | `controls-kind="info"` |
| `controls-gap` | 控制按钮间距(px) | `controls-gap="10"` |

### CarouselItem 标签属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `caption-title` | 底部标题文字 | `caption-title="标题"` |
| `caption-text` | 底部描述文字 | `caption-text="描述"` |
| `caption-align` | 标题对齐：`left` / `center` / `right` | `caption-align="center"` |
| `caption-kind` | 标题栏 kind 风格 | `caption-kind="dark"` |
| `caption-background` | 标题栏自定义背景色 | `caption-background="rgb(50,50,60)"` |

### 图片使用

```html
<carousel-item caption-title="标题" caption-text="描述">
    <!-- 使用 img 标签显示图片 -->
    <img src="ui/images/carousel1.jpg" size-mode="stretch"></img>
</carousel-item>
```

`<img>` 的 `size-mode` 属性：

| 值 | 说明 |
|------|------|
| `stretch` | 拉伸填满整个控件区域（图片可能变形） |
| `fit` | 保持宽高比完整显示（可能留白，默认） |
| `cover` | 保持宽高比裁剪填满 |
| `original` | 显示原始尺寸 |

### 完整示例

```html
<carousel interval="3000" ride="true" wrap="true" controls="true" controls-kind="primary" dock="fill" height="300">
    <carousel-item caption-title="第一张" caption-text="描述文字" caption-align="center">
        <img src="ui/images/photo1.jpg" size-mode="stretch"></img>
    </carousel-item>
    <carousel-item caption-title="第二张" caption-text="右对齐" caption-align="right" caption-kind="info">
        <img src="ui/images/photo2.jpg" size-mode="stretch"></img>
    </carousel-item>
</carousel>

<!-- 无控制按钮 -->
<carousel controls="false" dock="fill" height="200">
    <carousel-item>
        <vbox dock="fill" style="background-color: rgb(13,110,253);">
            <label text="自定义内容" font-size="24" style="color: white;"></label>
        </vbox>
    </carousel-item>
</carousel>
```

## C++ API

### Carousel

```cpp
// 控制
void Next();              // 下一项
void Prev();              // 上一项
void GoTo(int index);     // 跳转到指定项
void Play();              // 开始自动轮播
void Pause();             // 暂停自动轮播
int GetCurrentIndex();    // 当前索引
int GetItemCount();       // 总项数

// 配置（公开成员变量）
int Interval = 5000;      // 自动轮播间隔
bool Wrap = true;         // 是否循环
bool PauseOnHover = true; // 悬停暂停

// 事件
std::function<void(int from, int to)> OnSlideChanged;
```

### CarouselItem

```cpp
// 通过 htm 属性设置 caption，C++ 中可通过 FindControl 获取子控件后自定义
```

## 实现原理

### 轮播机制

1. 所有 `CarouselItem` 构造时默认 `SetVisible(false)`
2. `Carousel::Add` 中将 `CarouselItem` 插入到控制栏之前，首次添加的 item 设为可见
3. `GoTo(index)` 隐藏当前 item，显示目标 item，触发事件和定时器重启
4. `CarouselItem` 设 `SetDockStyle(DockStyle::Fill)` 撑满父容器（但 VLayout 不使用 DockStyle，由布局自动撑满）

### 自动轮播

- `Timer` 在独立线程中运行，`Interval` 毫秒后通过 `BeginInvoke` 切到主线程调用 `Next()`
- 每次切换完成后自动重启 Timer，实现循环
- `m_hovering` 标记控制悬停暂停

### 控制按钮

- 构造时创建一个 `HLayout`（固定高度30px）放在最底部
- 内部用 lambda 创建 4 个 `Label` 按钮，设 `kind="primary"`，圆角5px
- 首尾 `Spacer` 弹性撑开实现居中，按钮间 `HSpacer(10)` 保持间距
- 通过 `controls` 属性控制显示/隐藏

### Caption 标题栏

- 首次设置 `caption-title` 或 `caption-text` 时创建半透明黑色背景的 `HLayout`（固定高度60px）
- 内部 VLayout 包含标题和描述两个 Label
- 支持 `caption-align` 对齐、`caption-kind` 风格、`caption-background` 自定义背景

## 踩坑记录

### 坑1：img(PictureBox) 图片不铺满容器

**问题**：`<img>` 在 CarouselItem 内部没有铺满宽度，左右有空白。

**原因**：图片控件的 `Image->SizeMode` 默认是 `ImageSizeMode::Fit`（保持宽高比完整显示），导致图片在控件内按比例缩放，四周留白。同时 `PictureBox` 的 `GetFixedWidth()` 在布局时返回0，VLayout 分配的宽度看似正确但实际上绘制时用了 Fit 模式。

**修复**：在 `<img>` 上加 `size-mode="stretch"` 属性，`PictureBox::SetAttribute` 中解析 `size-mode` 属性设置 `Image->SizeMode = SizeMode::Stretch`，让图片拉伸填满整个控件区域。

### 坑2：width 属性对 PictureBox 无效

**问题**：`<img width="600">` 或 `<img width="100%">` 没有效果，图片宽度不变。

**原因**：`width="100%"` 调用 `SetRateWidth(1.0)` 设置 `m_rateSize.Width = 1.0`，但 `GetFixedWidth()` 在布局时调用，此时 Parent 宽度可能还是0，导致 `m_fixedSize.Width = 0 * 1.0 = 0`。布局完成后 Parent 宽度正确了，但 `m_fixedSize.Width` 没有重新计算。

**修复**：使用 `size-mode="stretch"` + 不设 width 属性，让 VLayout 分配宽度 + 拉伸绘制即可铺满。

### 坑3：CarouselItem 内容不显示

**问题**：第一次打开测试窗口时 `<carousel-item>` 里的内容看不到。

**原因**：`CarouselItem` 构造时 `SetVisible(false)`，但 `Carousel::Add` 中首次添加时设 `ShowItem(0)` 显示第一项。如果没有触发 `Add`（或者 `Add` 时 CarouselItem 还没添加到 Parent），item 保持隐藏。

**修复**：`Carousel::Add` 中检测 `dynamic_cast<CarouselItem*>`，如果是第一次添加（`m_currentIndex < 0`）则显示该项。

### 坑4：控制按钮用 SetRect 手动定位导致布局混乱

**问题**：最初用 `SetRect` 手动定位 prev/next 按钮到 Carousel 左右两侧，导致与其他子控件布局冲突，原有的轮播功能异常。

**原因**：`VLayout::OnLayout` 会重新布局所有子控件，手动 `SetRect` 的位置会被覆盖。而且 `SetRect` 不是虚函数，`override` 编译报错。

**修复**：改为在 Carousel 最底部加一个固定高度的 `HLayout` 作为控制栏，通过正常的 HLayout 布局来居中按钮，VLayout 自动排列在内容下方，不干扰轮播项布局。

### 坑5：Caption Bar 的创建时机

**问题**：`caption-title` 和 `caption-text` 属性在 htm 中设置时，CarouselItem 还没有任何子控件。如果先设 `caption-title` 再设 `caption-text`，会创建两次 caption bar。

**原因**：两个属性各自触发创建逻辑，没有防重复。

**修复**：在 `SetAttribute("caption-title")` 和 `SetAttribute("caption-text")` 中判断 `m_captionBar` 是否已创建，只有首次设置时才创建。

## 文件位置

- 头文件：`src/include/EzUI/Carousel.h`
- 实现文件：`src/sources/Carousel.cpp`
- PictureBox 增强：`src/sources/PictureBox.cpp`（`size-mode` 属性支持）
- 测试 htm：`bin/ui/htm/carouselBrowser.htm`
- 测试窗口：`src/demo/helloWorld/main.cpp` 中的 `CarouselBrowser` 类
- 测试图片：`bin/ui/images/carousel1.jpg`、`carousel2.jpg`、`carousel3.png`
