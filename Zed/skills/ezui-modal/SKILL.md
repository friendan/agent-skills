---
name: ezui-modal
description: 指导如何使用EZUI框架的Modal模态对话框，包括静态方法调用、htm布局拼接、回调处理、踩坑记录等。
---

# EZUI Modal 模态对话框指南

## 概述

Modal 模态对话框，类似 Bootstrap 的 `.modal`，用于在页面顶部显示需要用户交互的对话框内容。支持标题、内容、确定/取消按钮、回调通知、点击遮罩关闭等功能。

基于 `LayeredWindow` 实现（同 Toast），不依赖主窗口控件树。

## 实现原理

- `ModalPopup` 继承 `LayeredWindow`，`WS_EX_NOACTIVATE | WS_EX_LAYERED | WS_EX_TOOLWINDOW`
- 窗口大小由 `ModalOptions` 的 `Width`/`Height` 控制
- 内容通过 htm 字符串动态构建，用 `UIManager::LoadXml` + `SetupUI` 加载
- 半透明遮罩是 htm 中一个 `rgba(0,0,0,0.4)` 背景的 vbox，覆盖整个窗口
- 确定/取消按钮绑定 `EventHandler`，点击后触发回调并异步销毁窗口
- 异步销毁同 Toast 机制：`PostMessage(WM_GUI_TOAST_CLEANUP)`

## 使用方式

### 简单提示

```cpp
Modal::Show(L"操作成功！");
Modal::Show(L"提示", L"文件已保存。");
```

### 确认对话框

```cpp
Modal::Show(L"确认删除", L"确定要删除这条记录吗？",
    ModalOptions().Kind(ControlKind::Danger).ShowCancel(true),
    [](bool ok) {
        if (ok) {
            // 用户点击了确定
        } else {
            // 用户点击了取消或遮罩
        }
    });
```

### 完整选项

```cpp
Modal::Show(L"标题", L"内容",
    ModalOptions()
        .Kind(ControlKind::Primary)   // 标题栏颜色
        .ShowCancel(true)              // 是否显示取消按钮
        .OkText(L"保存")               // 确定按钮文字
        .CancelText(L"放弃")           // 取消按钮文字
        .Width(500)                    // 窗口宽度
        .Height(300)                   // 窗口高度
        .ClickBackdropToClose(false),  // 点击遮罩是否关闭
    [](bool ok) {
        // 回调
    });
```

## ModalOptions API

| 方法 | 说明 | 默认值 |
|------|------|--------|
| `Title(text)` | 标题文字 | `""` |
| `Text(text)` | 内容文字 | `""` |
| `Kind(kind)` | 标题栏颜色 | `Primary` |
| `ShowCancel(bool)` | 是否显示取消按钮 | `false` |
| `OkText(text)` | 确定按钮文字 | `"确定"` |
| `CancelText(text)` | 取消按钮文字 | `"取消"` |
| `Width(w)` | 窗口宽度 | `420` |
| `Height(h)` | 窗口高度 | `200` |
| `ClickBackdropToClose(bool)` | 点击遮罩关闭 | `true` |

## HTM 布局结构

动态拼接的 htm 结构如下：

```
vbox#modalRoot (dock=fill)
  vbox#modalBackdrop (dock=fill, rgba黑色半透明)
    spacer (height=star)
    hlayout#modalCenter (halign=center, 白色背景圆角)
      vbox (dock=fill)
        hlayout (kind, 标题栏)
          label#modalTitle (标题文字)
          spacer (width=star)
        vbox (内容区)
          label#modalText (内容文字)
        hlayout (按钮栏)
          spacer (width=star)
          button#modalCancelBtn (取消)
          button#modalOkBtn (确定)
    spacer (height=star)
```

## 关联代码

| 文件 | 内容 |
|------|------|
| `src/include/EzUI/Modal.h` | Modal 和 ModalOptions 类声明 |
| `src/sources/Modal.cpp` | ModalPopup 窗口类和 Modal 静态方法实现 |
| `src/demo/helloWorld/main.cpp` | Modal 测试按钮和回调示例 |

## 踩坑记录

### 坑1：`UIString` 没有 `utf8()` 方法

`ui_text::String` 本身就是 `std::string`（UTF-8 编码），没有 `utf8()` 转换方法。

**解决**：直接用 `const std::string& utf8 = text;` 获取引用。

### 坑2：`SetLocation` 接受 `Point` 而非两个 int

`LayeredWindow::SetLocation` 的参数是 `const Point&`，不是 `(int x, int y)`。

**解决**：用 `SetLocation(Point(x, y))`。

### 坑3：htm 字符串拼接中的 `%d` 对应问题

`snprintf` 中如果用 `int` 变量对应 `%d`，确保变量类型为 `int`，而不是 `size_t` 或 `float`。

**解决**：从 `Opts.GetWidth()` 等返回值是 `int`，直接传递没问题。

### 坑4：按钮文字动态设置

如果需要动态修改按钮文字，必须在 htm 中用 `text` 属性预先设置，或者在控件查找后调用 `SetText`。

**解决**：在 `BuildHtm` 中直接用 `snprintf` 拼入最终文字，避免二次查找。

### 坑5：异步销毁

在 `WndProc(WM_CLOSE)` 中不能直接 `delete this` 或 `DestroyWindow`，会崩溃。

**解决**：与 Toast 相同机制：`ShowWindow(SW_HIDE)` + `PostMessage(WM_GUI_TOAST_CLEANUP)`。
