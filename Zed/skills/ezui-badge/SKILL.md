---
name: ezui-badge
description: 指导如何使用EZUI框架的角标（Badge）功能，包括htm属性设置、C++ API调用、样式自定义等。
---

# EZUI 角标（Badge）功能指南

## 概述

EZUI 的角标功能允许在任意控件右上角绘制徽标，支持数字、文字、自定义颜色、大小和位置偏移。角标形状根据内容自适应：1-2个字符显示为正圆形，3个及以上字符自动拉伸为圆角矩形（胶囊形）。

## 实现原理

- `Control` 类中新增 `Badge m_badge` 私有成员
- `Badge` 结构体包含：`Text`、`BackColor`、`ForeColor`、`Size`、`OffsetX`、`OffsetY`、`Visible`
- 在 `OnPaintBefore` 的边框绘制后、偏移弹出前调用 `OnBadgePaint` 虚方法进行绘制
- `OnBadgePaint` 可被子类重写自定义角标外观

## HTM 中使用

### 标签属性方式

直接在控件标签上设置以下属性：

```html
<label id="myBtn" text="消息" badge-text="3"></label>
```

### 支持的属性

| 属性 | 说明 | 默认值 | 示例 |
|------|------|--------|------|
| `badge-text` | 角标文字（空字符串则不显示） | `""` | `badge-text="99+"` |
| `badge-color` | 角标背景颜色 | `red` | `badge-color="rgb(0,100,200)"` |
| `badge-fore-color` | 角标文字颜色 | `white` | `badge-fore-color="rgb(255,255,200)"` |
| `badge-size` | 角标直径（px） | `16` | `badge-size="20"` |
| `badge-offset-x` | 水平偏移（正数向左移） | `0` | `badge-offset-x="10"` |
| `badge-offset-y` | 垂直偏移（正数向下移） | `0` | `badge-offset-y="10"` |

### 完整示例

```html
<vbox id="mainLayout" dock="fill">
    <!-- 基础数字角标 -->
    <label id="badge1" text="消息" badge-text="3" badge-size="20"></label>

    <!-- 拉伸角标（3字符以上变胶囊形） -->
    <label id="badge2" text="通知" badge-text="99+" badge-size="20"></label>

    <!-- 自定义颜色 -->
    <label id="badge3" text="VIP" badge-text="VIP" badge-color="rgb(150,50,200)" badge-size="20"></label>

    <!-- 自定义大小和偏移 -->
    <label id="badge4" text="大角标" badge-text="9" badge-size="28" badge-offset-x="5" badge-offset-y="5"></label>
</vbox>
```

## C++ API

```cpp
// 获取角标数据引用（可直接修改字段）
Badge& GetBadge();

// 设置角标文字（自动显示，空文字隐藏）
void SetBadge(const UIString& text);

// 设置角标文字及颜色
void SetBadge(const UIString& text, const Color& backColor, const Color& foreColor);

// 显式控制角标可见性
void SetBadgeVisible(bool visible);
```

### C++ 使用示例

```cpp
auto* btn = FindControl(L"myBtn");

// 简单设置数字
btn->SetBadge(L"99+");

// 设置带颜色
btn->SetBadge(L"!", Color::Red, Color::White);

// 微调位置
btn->GetBadge().OffsetX = 5;
btn->GetBadge().OffsetY = 2;

// 修改大小
btn->GetBadge().Size = 20;

// 隐藏角标
btn->SetBadgeVisible(false);
```

## 重写角标绘制

子类可重写 `OnBadgePaint` 实现自定义角标外观：

```cpp
class MyControl : public ezui::Label {
protected:
    void OnBadgePaint(PaintEventArgs& e) override {
        if (!GetBadge().Visible || GetBadge().Text.empty()) return;

        auto& g = e.Graphics;
        // 自定义绘制逻辑
        // ...
    }
};
```

## 注意事项

1. 角标默认位置在控件**右上角**，通过 `OffsetX`（正数左移）和 `OffsetY`（正数下移）微调
2. 角标文字不超过2个字符时为正圆形，3个及以上自动变为圆角矩形（胶囊形）
3. `badge-text=""` 或 `SetBadge("")` 时角标不显示（`Visible` 自动设为 `false`）
4. 角标不参与布局计算，仅作为绘制叠加层
5. 角标不受 `ControlStyle` 的状态切换影响（hover/active 不会改变角标样式）
