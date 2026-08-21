---
name: wpf-dpi-static-check
description: >-
  Statically audit a WPF project for missing Per-Monitor DPI scaffolding
  (app.manifest PerMonitorV2, UseLayoutRounding on each Window,
  DpiTextRenderingHelper.Attach, hardcoded TextFormattingMode=Ideal) and
  overlay dialogs that Window-level checks miss (MaterialDesign DialogHost,
  Popup, DropShadowEffect on text). Use only when the user explicitly asks
  to run a static check, DPI 静态检查, 检查窗口 DPI, 检查对话框, DialogHost,
  遮罩弹窗, 检查 UseLayoutRounding, or to find missed DPI setup. Do not use
  when adding a new Window unless the user asked for a check, and do not
  use for full multi-monitor layout adaptation (see wpf-multi-dpi-adaptation).
---

# WPF DPI 脚手架静态检查

只做仓库内 Grep，**不编译、不启动应用**。漏了不报错，只能靠这次检查发现。

用户没点名要检查时不要跑。完整跨屏适配见 `wpf-multi-dpi-adaptation`。

## 检查进度

```
- [ ] 1. 列出全部顶层 Window，并给对话框分类（独立 Window / DialogHost 遮罩 / Popup）
- [ ] 2. app.manifest 是否声明 PerMonitorV2，csproj 是否引用 ApplicationManifest
- [ ] 3. 每个独立 Window 根节点是否有 UseLayoutRounding="True"
- [ ] 4. 每个独立 Window 构造函数是否调用 DpiTextRenderingHelper.Attach(this)
- [ ] 5. 是否存在写死的 TextFormattingMode="Ideal"
- [ ] 6. 遮罩/Popup 对话框：Effect 是否包住文字、是否用 Viewbox 撑开
- [ ] 7. 按差集补漏，向用户列出通过 / 漏设（含对话框分类结论）
```

## 步骤 1：列出 Window，并给对话框分类

先搜 `*.xaml` 根元素 `<Window`，记下路径和 `x:Class`。再搜对话框实现，**不要假设对话框都是 Window**：

| 形态 | 怎么认 | 只查 Window 能否覆盖 |
| --- | --- | --- |
| 独立窗口 | `ShowDialog()` / 根元素就是 `<Window` 的登录窗、确认窗 | **能**。按步骤 3–4 做差集。 |
| 层内遮罩 | `materialDesign:DialogHost`、`DialogHost.Show`、主窗口变暗 + 居中卡片（截图这类） | **不能**。卡片是宿主 Window 的子树，脚手架会继承，典型问题是阴影/Viewbox，见步骤 6。 |
| Popup | `<Popup` | **不能**。Popup 另开 HWND，宿主 Window 的 `UseLayoutRounding` 到不了 Popup 内容。 |

`UserControl` / `Page` 本身不是 Window：若它只是某页内容则跳过；若它是 DialogHost 的 `DialogContent` 或 Popup 的 Child，归入步骤 6，不要当成独立 Window 去要 `Attach`。

建议搜索：

```
<Window
ShowDialog
DialogHost
<Popup
```

## 步骤 2：PerMonitorV2（项目级，只查一处）

同时满足才算通过：

- `app.manifest` 的 `windowsSettings` 里有 `dpiAwareness` = `PerMonitorV2`
- `.csproj` 有 `<ApplicationManifest>app.manifest</ApplicationManifest>`（或等价路径）

缺一处则整进程跨屏都会糊（含遮罩对话框），优先补。模板见 `wpf-project-init` 的 `templates.md`。

## 步骤 3–4：每个独立 Window 对两项

只对步骤 1 里「独立窗口」做差集；DialogHost 内容不要列进这项漏设。

| 项 | 搜什么 | 漏了的样子 |
| --- | --- | --- |
| 像素对齐 | 该 XAML 的 `<Window` 开标签上有 `UseLayoutRounding="True"` | 根节点没有此属性 |
| 文字模式 | 对应 code-behind 构造函数里有 `DpiTextRenderingHelper.Attach(this)` | 没有 Attach，或项目没有 `DpiTextRenderingHelper` |

建议搜索：

```
UseLayoutRounding
DpiTextRenderingHelper
```

`UseLayoutRounding` 必须落在各 **Window 根节点**。Helper 不存在时从 `wpf-project-init/templates.md` 拷 `Helpers/DpiTextRenderingHelper.cs`，再给每个独立 Window 构造函数补 `Attach(this)`。不要在 XAML 写死 `TextFormattingMode`。

宿主 Window 过了 3–4，只说明 DialogHost 能继承脚手架，**不代表**遮罩卡片本身没问题。

## 步骤 5：禁止写死 Ideal

搜索：

```
TextFormattingMode
```

任何 XAML 的 `TextOptions.TextFormattingMode="Ideal"`（或对子级 Set 成 Ideal）都算失败：局部值会盖掉 Window 上的动态切换。DialogHost 内容上写死同样算。删掉这些局部赋值。

## 步骤 6：遮罩 / Popup（Window 检查盖不住的部分）

主窗口变暗、中间一块带阴影的卡片，就是这一类。对步骤 1 标出的 DialogHost 内容、`*Dialog*` 用户控件、`<Popup` 子树做下面几项。

搜索：

```
DropShadowEffect
BitmapCache
CacheMode
Viewbox
```

**Effect 包住文字**（最常见）：`DropShadowEffect` / `Effect=` 挂在**含有** `TextBlock` / `TextBox` / `Button` 的祖先上 → 整棵子树进中间位图，ClearType 没了，对话框里那行字会糊。修法：阴影只挂在**没有文字的兄弟背景层**（`Border`），文字层与之平级。完整示例见 `wpf-multi-dpi-adaptation` 的 5.2。`BitmapCache` 同理。

**Viewbox 包住对话框**：任意小数倍缩放，卡片会又大又空、字发虚。报出来；含文字的常驻对话框不要用 Viewbox，改固定 DIP + 弹性留白。细节见适配 skill 第 2 层。

**Popup 根节点**：在 Popup 的 Child 根上补 `UseLayoutRounding="True"`（宿主 Window 的设不到这里）。

写死过大的 `Width`/`Height` 导致内容顶对齐、下方大片空白：列进报告，**不要在本检查里猜着改尺寸**；那是布局问题，交给适配 skill 或业务 XAML。

## 步骤 7：修复与汇报

- 漏 `UseLayoutRounding` / `Attach`：只补**独立 Window**（以及 Popup 的 Child 根）。
- 漏 Helper 类：按 init 模板新增。
- 写死 Ideal：删除。
- DialogHost 卡片 Effect 包文字：拆成背景层 + 文字层。
- 不顺手改弹性布局、断点、业务文案。

汇报时按分类写，避免把「DialogHost 没 Attach」当成漏设：

- 独立 Window：通过 / 漏设文件
- 层内遮罩：宿主 Window 是否已过 3–4；Effect / Viewbox 命中文件
- Popup：Child 是否有 `UseLayoutRounding`；Effect 命中

脚手架和对话框 Effect 都过了仍发虚 → 停，改走 `wpf-multi-dpi-adaptation`。
