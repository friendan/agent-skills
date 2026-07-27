---
name: ezui-layeredwindow-alpha
description: 指导如何修复 EZUI 框架中 LayeredWindow 使用 Direct2D 绘制后 alpha 通道不正确的问题（LayeredWindow 控件显示全透明/鼠标穿透）。
---

# EZUI LayeredWindow Alpha 通道修复

## 问题现象

`LayeredWindow` 子类（如 Modal 弹窗）显示全透明，鼠标点击穿透。使用 `Toast` 等其他 `LayeredWindow` 子类时可能正常，但这是因为具体控件的颜色覆盖了背景。

GDI 直接在 HDC 上绘制可以正常显示（如 `FillRect` 画纯色），但通过 `DXRender`（Direct2D）绘制后 `UpdateLayeredWindow` 调用 `AC_SRC_ALPHA` 时显示全透明。

## 根因

`LayeredWindow::Paint()` 中先调用 `Bitmap::Earse()` 用 `memset` 将 DIB 像素的 ARGB 全部清为 0（alpha=0，完全透明）。然后 `DoPaint` 通过 `DXRender`（`ID2D1DCRenderTarget`）在 `D2D1_ALPHA_MODE_PREMULTIPLIED` 预乘 alpha 模式下绘制控件。

Direct2D 的 `ID2D1DCRenderTarget` 写入 `CreateDIBSection` 创建的 32-bit DIB 后，alpha 通道值不正确，导致 `UpdateLayeredWindow` 使用 `AC_SRC_ALPHA` 时认为像素完全透明。

## 修复方案

在 `LayeredWindow::Paint()` 中，不再依赖 `Earse` 的清零 + Direct2D 的 alpha 写入，而是在 `DoPaint` 之前先用 GDI 绘制白色背景，再强制遍历像素将 alpha 通道设为 0xFF：

```cpp
void LayeredWindow::Paint() {
    if (IsVisible()) {
        Rect invalidateRect;
        BeginPaint(&invalidateRect);
        if ((m_winBitmap && !invalidateRect.IsEmptyArea())) {
            HDC winHDC = m_winBitmap->GetDC();
            // 用 GDI 画白底后再强制设 alpha=255，确保 UpdateLayeredWindow 的 AC_SRC_ALPHA 正确
            {
                RECT rc = { invalidateRect.X, invalidateRect.Y,
                            invalidateRect.GetRight(), invalidateRect.GetBottom() };
                HBRUSH hBrush = ::CreateSolidBrush(RGB(255, 255, 255));
                ::FillRect(winHDC, &rc, hBrush);
                ::DeleteObject(hBrush);
            }
            {
                byte* pixels = m_winBitmap->GetPixel();
                int stride = m_winBitmap->Width() * 4;
                for (int y = invalidateRect.Y; y < invalidateRect.GetBottom(); ++y) {
                    DWORD* p = (DWORD*)(pixels + invalidateRect.X * 4 + y * stride);
                    for (int x = 0; x < invalidateRect.Width; ++x) {
                        p[x] |= 0xFF000000;
                    }
                }
            }
            DoPaint(winHDC, invalidateRect);
            UpdateLayeredWindow(winHDC);
            EndPaint();
        }
    }
}
```

## 修改文件

- `trunk/src/sources/LayeredWindow.cpp` — `Paint()` 方法

## 注意事项

- 此修改不影响 `Bitmap::Earse()` 的其他调用方
- `FillRect` 覆盖了擦除效果，因此可以省略前面的 `Earse` 调用
- 该方法适用于所有 `LayeredWindow` 子类（Modal、Toast、Tooltip 等）
- 如果未来 Direct2D 的 alpha 写入问题被修复，可以还原回原始的 `Earse` + `DoPaint` 流程
