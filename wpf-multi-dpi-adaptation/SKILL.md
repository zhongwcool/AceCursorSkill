---
name: wpf-multi-dpi-adaptation
description: Make a WPF desktop app automatically adapt to different monitor resolutions and DPI scaling, including dragging the window between monitors with different scaling (100% ↔ 150%). Use when a WPF app looks blurry or wrong-sized on high-DPI/4K screens, when UI breaks after moving the window to another monitor, when text is still blurry in some cases even after PerMonitorV2 is enabled (small text blurry at 100% scaling, blurry text inside DropShadowEffect popups, blurry scaled controls or Viewbox content, stretched bitmaps), when the user mentions Per-Monitor DPI, PerMonitorV2, dpiAwareness, app.manifest, TextFormattingMode, 分辨率适配, 多显示器, 高分屏, DPI 缩放, 发虚, 模糊, or wants a layout that scales across resolutions.
---

# WPF 多分辨率 / 多显示器 DPI 自动适配

让 WPF 桌面应用在**不同分辨率、不同 DPI 缩放的显示器之间切换/拖动时自动适配**，不发虚、不错位、不溢出。

核心认知：**真正的"跨屏自动适配"不是靠手写检测分辨率再重排，而是靠多层机制配合**。手写代码只负责"窗口变宽/变窄时增减模块"这种响应式优化。按下面五层逐项落地即可。

| 层级 | 机制 | 解决的问题 |
|------|------|-----------|
| 1 系统级 | `app.manifest` 声明 `PerMonitorV2` | 跨不同 DPI 显示器拖动时自动重新缩放、不模糊 |
| 2 框架级 | WPF DIP 矢量渲染 + 弹性布局 + `MinSize` | 不同分辨率按比例伸缩 |
| 3 应用级 | `SizeChanged` 响应式断点（宽则显示、窄则隐藏/折叠） | 宽屏/窄屏布局自适应 |
| 4 细节级 | 多字重 TTF + `UseLayoutRounding` + 导出位图 DpiScale | 高分屏文字/图片清晰 |
| 5 长尾级 | Ideal/Display 按 DPI 切换、Effect 与文字分离、禁小数缩放、界面位图多倍图 | 前四层做完后"某些情况仍发虚"的残余问题 |

## 落地清单

```
适配进度：
- [ ] 第 1 层：app.manifest 启用 PerMonitorV2（最关键，缺它跨屏必模糊）
- [ ] 第 2 层：窗口用弹性布局 + MinSize + UseLayoutRounding
- [ ] 第 3 层：需要的页面加 SizeChanged 响应式断点
- [ ] 第 4 层：字体/位图高清处理
- [ ] 第 5 层：PerMonitorV2 之后仍发虚的长尾原因逐项排查（见下文）
- [ ] 验证：拖到不同缩放的显示器、改系统缩放、改分辨率
```

---

## 第 1 层（必做）：app.manifest 启用 Per-Monitor V2

这是"切换不同 DPI 显示器自动适配"的**根本**。没有它，跨屏拖动一定模糊或尺寸错乱。

`.csproj` 里确认引用了 manifest：

```xml
<PropertyGroup>
  <ApplicationManifest>app.manifest</ApplicationManifest>
</PropertyGroup>
```

`app.manifest` 加入（`windowsSettings` 段）：

```xml
<application xmlns="urn:schemas-microsoft-com:asm.v3">
  <windowsSettings>
    <!-- Per-Monitor V2 DPI Awareness -->
    <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2</dpiAwareness>
    <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">true/pm</dpiAware>
  </windowsSettings>
</application>
```

声明后：窗口被拖到另一台不同缩放比例的显示器时，Windows 发 `WM_DPICHANGED`，**.NET Core / .NET 5+ 的 WPF 框架内建支持**会自动重算渲染缩放并重绘，保持物理尺寸一致、文字锐利。**通常无需自己处理 `WM_DPICHANGED`**。

> 注意：.NET Framework 4.x 对 PerMonitorV2 支持不完整，跨屏可能仍需额外处理或升级到 .NET Core+。本 skill 默认 .NET Core / .NET 5+。

---

## 第 2 层：窗口用弹性布局（分辨率无关）

WPF 以**设备无关单位 DIP（1/96 英寸）矢量渲染**，只要不写死像素绝对定位，分辨率变化会自动缩放。要点：

**窗口根元素属性**（顶层 `Window`）：

```xml
<Window WindowStartupLocation="CenterScreen"
        MinHeight="790" Height="900" MinWidth="1350" Width="1350"
        TextOptions.TextRenderingMode="Auto"
        UseLayoutRounding="True">
```

- `TextFormattingMode` **不要在 XAML 里写死**，在 code-behind 按 DPI 动态设置（见 5.1）。
- `MinHeight`/`MinWidth`：低分辨率下的兜底，防止内容被压垮。
- `UseLayoutRounding="True"`：把布局对齐到像素边界，**消除高分屏上 1px 错位/发虚**。
- `WindowStartupLocation="CenterScreen"`：在当前主屏居中弹出。

**布局规则**（贯穿所有页面）：

- 用 `Grid` + `Width="*"` / `Height="*"` 按比例伸缩；用 `Auto` 包裹随内容大小的区域；**避免给容器写死像素宽高**。
- **慎用 `Viewbox`**：它本质是任意小数倍 `ScaleTransform`，内部 `UseLayoutRounding` 的像素对齐会全部失效（对齐到的是缩放前的网格），窗口尺寸稍变就产生 0.87、1.13 之类的非整数缩放，**文字和图标边缘必然发虚**。含文字/图标的常驻 UI（如导航栏）应固定 DIP 尺寸 + 弹性留白；`Viewbox` 只用于纯矢量图形装饰、或确实需要整体等比缩放且能接受轻微发虚的场景。
- 列表高度用 `Grid` 的 `*` 行 + `Stretch` 填满父容器，而非固定 `Height`，使其随窗口伸缩。

---

## 第 3 层：响应式断点（窗口变宽/变窄时增减模块）

类似网页响应式布局：监听 `SizeChanged`，按当前**可视宽度**决定某模块显示/隐藏/折叠。适合"宽屏多列、窄屏少列"。

模式（code-behind）：

```csharp
private const double LeftFixedColumnsWidth = 900;   // 固定区占用
private const double MinPanelWidth = 450;           // 该模块所需最小宽度（断点）

public MyPage()
{
    InitializeComponent();
    Loaded += (_, _) => Dispatcher.BeginInvoke(
        new Action(UpdatePanelVisibilityByWidth), DispatcherPriority.Loaded);
    SizeChanged += (_, _) => Dispatcher.BeginInvoke(
        new Action(UpdatePanelVisibilityByWidth), DispatcherPriority.Loaded);
}

private void UpdatePanelVisibilityByWidth()
{
    var width = MainScrollViewer?.ViewportWidth;
    if (!width.HasValue || double.IsNaN(width.Value) || width.Value <= 0)
        width = MainScrollViewer?.ActualWidth > 0 ? MainScrollViewer.ActualWidth : ActualWidth;

    var remaining = Math.Max(0, width.Value - LeftFixedColumnsWidth);
    var show = remaining >= MinPanelWidth;

    OptionalPanelColumn.Width = show ? new GridLength(remaining) : new GridLength(0);
    // 同步给 ViewModel 控制可见性 / 停用其内部定时器等资源
}
```

要点：
- 用 `Dispatcher.BeginInvoke(..., DispatcherPriority.Loaded)` 延迟到布局完成后再读 `ActualWidth`，避免读到 0。
- 优先用 `ViewportWidth`（滚动视口实际宽），回退到 `ActualWidth`。
- 隐藏时把列宽设为 `GridLength(0)`，并联动停掉该模块的定时器/订阅，省资源。

**对话框/弹窗尺寸跟随主窗口**，避免小屏溢出：

```csharp
var windowHeight = Application.Current.MainWindow?.ActualHeight;
if (!windowHeight.HasValue || windowHeight.Value <= 0)
    windowHeight = SystemParameters.WorkArea.Height;   // 兜底用工作区高度
MaxHeight = Math.Max(500, windowHeight.Value - 140);
```

---

## 第 4 层：高分屏清晰度细节

- **字体**：为每个字重单独嵌入 `.ttf`（Regular/Bold/Medium…），**不要依赖 WPF 合成粗体**（合成粗体在高分屏发虚）。中文用矢量字体（如 Noto Sans SC），等宽场景用 JetBrains Mono 等。
- **`UseLayoutRounding="True"`**（见第 2 层）同样改善文字锐度。
- **导出位图**（截图/导出图片/PDF）时按缩放系数放大渲染，避免位图糊：

```csharp
float dpiScale = 1.5f;                 // 高清系数
var width  = (int)(baseWidth  * dpiScale);
var height = (int)(baseHeight * dpiScale);
using var bitmap = new Bitmap(width, height);
bitmap.SetResolution(96 * dpiScale, 96 * dpiScale);   // 设置高 DPI
// graphics.TextRenderingHint = TextRenderingHint.ClearTypeGridFit; 等高质量渲染参数
```

---

## 第 5 层：PerMonitorV2 之后"某些情况仍发虚"的长尾原因

第 1~4 层做完后若个别场景仍模糊，按下面逐项排查（按出现频率排序）：

### 5.1 `TextFormattingMode` 要按 DPI 动态切换，不能写死 `Ideal`

WPF 实际行为：**100% 缩放（96 DPI）屏上，11~13px 小字用 `Ideal` 渲染偏灰发虚，`Display` 才锐利；125% 及以上 `Ideal` 更清晰**。写死 `Ideal` 的典型症状是"高分屏清晰、拖回普通屏后小字模糊"。正确做法是设在 Window 上并监听 `DpiChanged` 动态切换（该属性可继承，会传遍整棵视觉树，包括 DialogHost 对话框和 Popup）：

```csharp
public static class DpiTextRenderingHelper
{
    public static void Attach(Window window)
    {
        window.Loaded += (_, _) => Apply(window, VisualTreeHelper.GetDpi(window));
        window.DpiChanged += (_, e) => Apply(window, e.NewDpi);
    }

    private static void Apply(Window window, DpiScale dpi)
    {
        TextOptions.SetTextFormattingMode(window,
            dpi.DpiScaleX > 1.0 ? TextFormattingMode.Ideal : TextFormattingMode.Display);
    }
}
// 每个 Window 构造函数里：DpiTextRenderingHelper.Attach(this);
```

注意：**删掉子级 XAML 里所有写死的 `TextOptions.TextFormattingMode="Ideal"`**，否则局部值会覆盖继承，动态切换失效。每个独立 Window（登录窗、弹出窗）都要各自 Attach 并设置 `UseLayoutRounding="True"`——这些属性不会跨 Window 继承。

### 5.2 `Effect`（如 DropShadowEffect）会杀死子树内文字的 ClearType

任何挂了 `Effect` 的元素，其**整个子树先渲染进中间位图纹理，ClearType 自动降级为灰度抗锯齿**，里面的文字看起来"糊了一层"。修法：把阴影挂到一个**不含文字的兄弟背景层**上：

```xml
<Grid>
    <!-- 背景层：只有它挂 Effect -->
    <Border CornerRadius="10" Background="...">
        <Border.Effect><DropShadowEffect BlurRadius="16" ShadowDepth="2"/></Border.Effect>
    </Border>
    <!-- 文字层：与背景层平级，不受 Effect 影响，ClearType 保留 -->
    <StackPanel Margin="12"><TextBlock Text="..."/></StackPanel>
</Grid>
```

`BitmapCache`/`CacheMode` 同理（且默认按 1x 缓存，高分屏更糊；必须用时设 `RenderAtScale`）。

### 5.3 小数倍 `ScaleTransform`/`LayoutTransform` 缩放含文字的控件

`ScaleX="0.75"` 这类小数缩放让边缘落在半像素上（再叠加显示器 150% 就是 0.75×1.5=1.125 倍），控件轮廓和文字必然发虚，`UseLayoutRounding` 在变换内部不生效。修法：**不缩放，直接做小尺寸样式/模板**；确需缩放动画时只在动画瞬间缩放、静止态恢复 1.0。

### 5.4 界面内位图只有单一尺寸

第 4 层讲的是导出位图；**界面内展示的 png/jpg（logo、插图）在 150%/200% 屏会被拉伸变糊**。修法按优先级：改用矢量（Path/字体图标/SVG 转 XAML）；或提供 2x/3x 多倍图按 DPI 切换；至少设 `RenderOptions.BitmapScalingMode="HighQuality"`。反过来，列表里为性能设了 `LowQuality` 的地方确认其中没有位图。

### 5.5 其他

- `SnapsToDevicePixels` 只影响少数场景，主力是 `UseLayoutRounding`；两者都治不了变换内部的错位。
- 需要局部强制 ClearType 时可设 `RenderOptions.ClearTypeHint="Enabled"`（如半透明背景上的文字层）。

---

## 验证清单（务必实测）

仅看代码不算适配成功，按下面三个场景实测：

- [ ] **跨屏拖动**：把窗口从 100% 缩放屏拖到 150%/200% 屏，文字应保持锐利、控件物理尺寸一致、不错位。**再拖回 100% 屏检查小字号文字**（验证 Ideal/Display 动态切换，见 5.1）。
- [ ] **重点看局部**：带阴影的弹窗/卡片内的文字、被缩放过的控件、位图 logo——这些是整窗清晰但局部发虚的高发区（见第 5 层）。
- [ ] **改系统缩放**：在显示设置里把缩放从 100% 改到 150%/175%，重开应用，布局正常、不溢出。
- [ ] **改分辨率**：切到 1366×768 等低分辨率，窗口不超出屏幕、`MinSize` 生效、响应式模块按断点折叠。

## 常见坑

- 只做了第 2/3 层却没加 `PerMonitorV2` → 跨屏拖动必模糊。**第 1 层是前提。**
- 全局写死 `TextFormattingMode="Ideal"` → 100% 缩放屏上小字发灰；按 DPI 动态切换（见 5.1）。
- 在含文字的元素上挂 `DropShadowEffect` → 子树 ClearType 失效；阴影移到无文字的背景层（见 5.2）。
- 用 `Viewbox` / 小数倍 `ScaleTransform` 缩放含文字的 UI → 像素对齐失效必发虚（见 5.3）。
- 独立 Window（登录窗等）漏设 `UseLayoutRounding` / 漏挂 DPI 切换 → 属性不跨 Window 继承，每个窗口都要设。
- 在 `Loaded` 之前读 `ActualWidth`/`ActualHeight` 得到 0 → 用 `DispatcherPriority.Loaded` 延迟。
- 给容器写死像素宽高 / 用 `StackPanel` 包大列表 → 破坏伸缩与虚拟化（列表卡顿另见 `wpf-list-virtualization-debug` skill）。
- 依赖 WPF 合成粗体 → 高分屏发虚；改为各字重独立 TTF。
