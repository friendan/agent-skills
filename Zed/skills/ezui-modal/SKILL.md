---
name: ezui-modal
description: 指导如何使用EZUI框架的Modal模态对话框，包括静态方法调用、htm布局拼接、回调处理、踩坑记录等。
---

# EZUI Modal 模态对话框指南

## 概述

Modal 模态对话框，类似 Bootstrap 的 `.modal`，用于在页面顶部显示需要用户交互的对话框内容。支持标题、内容、确定/取消按钮、回调通知等功能。

基于 `LayeredWindow` 实现（同 Toast），不依赖主窗口控件树。

## 实现原理

- `ModalPopup` 继承 `LayeredWindow`，`WS_EX_NOACTIVATE | WS_EX_LAYERED | WS_EX_TOOLWINDOW | WS_EX_TOPMOST`
- 窗口大小由 `ModalOptions` 的 `Width`/`Height` 控制
- 内容通过 htm 字符串动态构建，用 `UIManager::LoadXml` + `SetupUI` 加载
- 确定/取消按钮绑定 `EventHandler`，点击后触发回调并异步销毁窗口
- 异步销毁同 Toast 机制：`PostMessage(WM_GUI_TOAST_CLEANUP)`
- owner 传 `NULL`，不依赖父窗口（避免点击按钮时影响父窗口状态）

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
            // 用户点击了取消
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
        .ClickBackdropToClose(false),  // 点击遮罩是否关闭（当前未实现遮罩，功能预留）
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
| `ClickBackdropToClose(bool)` | 点击遮罩关闭（预留） | `true` |

## HTM 布局结构

动态拼接的 htm 结构如下：

```xml
<vbox id="modalRoot" dock="fill" style="background-color: rgb(255,255,255);">
  <vbox dock="fill" style="padding: 0px;">
    <hlayout height="44" kind="%s" style="border-radius: 0;" action="title">
      <spacer width="16"></spacer>
      <label id="modalTitle" text="%s" font-size="14"
             style="font-weight: bold; color: rgb(255,255,255);"
             valign="middle" halign="left" action="title"></label>
      <spacer width="star"></spacer>
    </hlayout>
    <vbox dock="fill" style="padding: 20px 16px;" action="title">
      <label id="modalText" text="%s" font-size="13"
             style="color: rgb(60,60,60);" valign="top" event="none"></label>
    </vbox>
    <hlayout height="44" style="padding-right: 12px; border-top: 1px solid rgb(230,230,230);">
      <spacer width="star"></spacer>
      <button id="modalCancelBtn" text="%s" kind="secondary"
              width="72" height="30" style="border-radius: 4px;"></button>
      <spacer width="8"></spacer>
      <button id="modalOkBtn" text="%s" kind="%s"
              width="72" height="30" style="border-radius: 4px;"></button>
      <spacer width="12"></spacer>
    </hlayout>
  </vbox>
</vbox>
```

有取消按钮的版本包含 `modalCancelBtn` 和 `modalOkBtn` 两个按钮，无取消按钮的版本只包含 `modalOkBtn`。

## 关联代码

| 文件 | 内容 |
|------|------|
| `src/include/EzUI/Modal.h` | Modal 和 ModalOptions 类声明 |
| `src/sources/Modal.cpp` | ModalPopup 窗口类和 Modal 静态方法实现 |
| `src/sources/LayeredWindow.cpp` | LayeredWindow 基类的 Paint 实现（含 alpha 修复） |
| `src/demo/helloWorld/main.cpp` | Modal 测试按钮和回调示例 |

## 踩坑记录

### 坑1：`UIString` 没有 `utf8()` 方法

`ui_text::String` 本身就是 `std::string`（UTF-8 编码），没有 `utf8()` 转换方法。

**解决**：直接用 `const std::string& utf8 = text;` 获取引用。

### 坑2：`SetLocation` 接受 `Point` 而非两个 int

`LayeredWindow::SetLocation` 的参数是 `const Point&`，不是 `(int x, int y)`。

**解决**：用 `SetLocation(Point(x, y))`。

### 坑3：htm 拼接中的 `%s` 顺序必须与参数严格一致

`snprintf` 格式化时，若有取消按钮传 6 个参数（kn, title, text, cancel, ok, kn），若无取消按钮传 5 个参数（kn, title, text, ok, kn）。`%s` 数量必须匹配。

**解决**：使用两根模板字符串 `kModalHtmFmt` / `kModalHtmFmtNoCancel`，分离管理。

### 坑4：`snprintf` 定长缓冲区的截断风险

`char buf[4096]` + `snprintf` 分段拼接时，如果 title/text 较长可能导致截断，`n += snprintf(...)` 在截断后 `n > sizeof(buf)` 导致后续 `buf + n` 越界。

**解决**：先 `snprintf(nullptr, 0, ...)` 计算精确长度，再分配 `std::string` 并写入。

### 坑5：`EscapeHtml` 返回的临时 `std::string` 悬空指针

```cpp
const char* cancelEscaped = EscapeHtml(cancelText).c_str(); // 临时对象在此销毁！
```

三目运算符中 `EscapeHtml()` 返回的临时 `std::string` 在完整表达式结束时销毁，`.c_str()` 变成野指针。

**解决**：先存为具名 `std::string` 变量，再取 `.c_str()`。

### 坑6：owner 传入 `GetActiveWindow()` 导致点击按钮时主窗口最小化

传入了真实 owner 后，`CloseModal` 中 `ShowWindow(SW_HIDE)` 隐藏 owned 窗口时触发系统异常行为。

**解决**：owner 传 `NULL`，用 `WS_EX_TOPMOST` 保持置顶。

### 坑7：`LayeredWindow` 使用 Direct2D 绘制后 alpha 通道不正确

`Bitmap::Earse()` 用 memset 将 alpha 清 0，Direct2D 写入后 alpha 值未被正确设置为 255，导致 `UpdateLayeredWindow` 的 `AC_SRC_ALPHA` 显示全透明。

**解决**：在 `LayeredWindow::Paint()` 中用 GDI `FillRect` 画白底 + 遍历像素设 alpha=255，而非依赖 `Earse` + Direct2D 的 alpha 写入。
