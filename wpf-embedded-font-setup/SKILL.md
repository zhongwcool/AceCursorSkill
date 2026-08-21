---
name: wpf-embedded-font-setup
description: Embed custom .ttf fonts into a WPF app and use them via FontFamily resources. Bundles ready-to-copy ttf files (Inter, Noto Sans SC, JetBrains Mono in Regular/Medium/SemiBold/Bold) under assets/fonts/ plus their verified Win32FamilyName values, so no font download is needed. Use when a WPF project needs to bundle fonts so they work without system installation, when setting up FontFamily/pack URI resources in App.xaml, when adding extra font weights (Medium/SemiBold/Bold) or when FontWeight looks blurry / synthesized, when fonts don't render / fall back to system default, or when the user asks how to add, embed, or apply custom fonts in WPF/XAML.
---

# WPF 嵌入式自定义字体接入

把 `.ttf` 字体打包进 WPF 程序集，通过 `App.xaml` 的 `FontFamily` 资源在全局或局部使用。字体随程序分发，**不依赖目标机器是否安装该字体**。

## 字体文件：本 skill 已自带，直接拷

`assets/fonts/` 下已备好 12 个 ttf，无需再去下载，直接拷进目标项目：

```powershell
# 在项目根目录执行；<skill> 替换为本 skill 所在目录
New-Item -ItemType Directory -Force -Path "Resources\Fonts" | Out-Null
Copy-Item "<skill>\assets\fonts\*.ttf" -Destination "Resources\Fonts"
```

自带清单（Regular / Medium / SemiBold / Bold 各一档）：

| 字体 | 用途 | 单档体积 |
| --- | --- | --- |
| `Inter_18pt-*.ttf` | 英文正文 | 0.33 MB |
| `NotoSansSC-*.ttf` | 中文正文（Inter 缺字时回退） | 10 MB |
| `JetBrainsMono-*.ttf` | 等宽，数字/代码/时间对齐 | 0.1–0.26 MB |

合计约 42 MB，其中 Noto Sans SC 四档占 40 MB。**按实际用到的字重裁剪**：只用 Regular + SemiBold 的话，删掉两档中文就能省 20 MB。

三套字体均为 SIL OFL 1.1，允许随程序分发；对外发布时把 `OFL.txt` 一并带上（[Inter](https://github.com/rsms/inter) / [Noto Sans SC](https://fonts.google.com/noto/specimen/Noto+Sans+SC) / [JetBrains Mono](https://www.jetbrains.com/lp/mono/)）。

## 三步接入

### 1. 字体文件设为 Resource

把 `.ttf` 放进 `Resources/Fonts/`，在 `.csproj` 中**每个文件**改成 `Resource`（不是 `Content`/`Embedded Resource`）：

```xml
<ItemGroup>
    <None Remove="Resources\Fonts\JetBrainsMono-Regular.ttf"/>
    <Resource Include="Resources\Fonts\JetBrainsMono-Regular.ttf"/>
    <None Remove="Resources\Fonts\JetBrainsMono-Bold.ttf"/>
    <Resource Include="Resources\Fonts\JetBrainsMono-Bold.ttf"/>
</ItemGroup>
```

只引用**真实存在**的文件；按需打包字重（粗体发虚通常是因为缺对应字重的 ttf，WPF 做了合成加粗）。

Regular / Medium / SemiBold / Bold 四档足够覆盖界面需求，不必再堆 Light / Thin / Black / ExtraBold 或 `*NL-*`（no-ligature）变体——所以自带清单里也没放这些。

### 2. 在 App.xaml 声明 FontFamily 资源

```xml
<Application.Resources>
    <ResourceDictionary>
        <!-- 等宽字体（数字/时间对齐）：含 Regular 与 Bold -->
        <FontFamily x:Key="MonoFont">
            pack://application:,,,/Resources/Fonts/#JetBrains Mono
        </FontFamily>

        <!-- 等宽 Medium：独立 Win32 族名，必须单独一个 Key -->
        <FontFamily x:Key="MonoFontMedium">
            pack://application:,,,/Resources/Fonts/#JetBrains Mono Medium
        </FontFamily>

        <!-- 正文：英文 Inter，缺字（中文）自动回退 Noto Sans SC；含 Regular 与 Bold -->
        <FontFamily x:Key="GlobalFont">
            pack://application:,,,/Resources/Fonts/#Inter 18pt 18pt,
            pack://application:,,,/Resources/Fonts/#Noto Sans SC
        </FontFamily>

        <!-- Medium 正文：表单输入、列表主文案，比 Regular 有存在感又不过重 -->
        <FontFamily x:Key="GlobalFontMedium">
            pack://application:,,,/Resources/Fonts/#Inter 18pt 18pt Medium,
            pack://application:,,,/Resources/Fonts/#Noto Sans SC Medium
        </FontFamily>

        <!-- SemiBold 正文：标题强调，避免合成粗体发虚 -->
        <FontFamily x:Key="GlobalFontBold">
            pack://application:,,,/Resources/Fonts/#Inter 18pt 18pt SemiBold,
            pack://application:,,,/Resources/Fonts/#Noto Sans SC SemiBold
        </FontFamily>
    </ResourceDictionary>
</Application.Resources>
```

pack URI 关键规则（最易踩坑）：

```
pack://application:,,,/Resources/Fonts/#字体族名称
                       └─ 文件夹路径 ─┘ └─ 字体内部 Family，不是文件名 ─┘
```

- `#` 前是**文件夹**（不写具体 ttf 文件名，WPF 扫整个文件夹）。
- `#` 后必须是字体内部的 **Win32FamilyName（GDI 名）**，不是文件名、也不是普通 Family。
  例：`Inter_18pt-Regular.ttf` 的 Win32Family 是 `Inter 18pt 18pt`（看起来重复但正确）；`Inter_18pt-SemiBold.ttf` 是 `Inter 18pt 18pt SemiBold`；`JetBrainsMono-Regular.ttf` 是 `JetBrains Mono`。
- 逗号分隔多个字体 = **回退链**，前者缺字（如 CJK）时用后者渲染。
- 用相对路径 `/Resources/Fonts/` 而非写程序集名，可避免多环境下 AssemblyName 不同导致引用失败。

### 字重怎么归族：Bold 混在 Regular 里，Medium/SemiBold 各自独立

这是最反直觉的一点，**不能靠文件名推断**。下表是对 `assets/fonts/` 里那 12 个文件实测的结果，用自带字体时照抄即可、不必再跑脚本：

| ttf 文件 | Win32FamilyName |
| --- | --- |
| `*-Regular.ttf` | `Inter 18pt 18pt` / `Noto Sans SC` / `JetBrains Mono` |
| `*-Bold.ttf` | **与 Regular 完全相同** |
| `*-Medium.ttf` | `... Medium`（独立族） |
| `*-SemiBold.ttf` | `... SemiBold`（独立族，但 JetBrains Mono 例外，仍归 `JetBrains Mono`） |

推论出两条用法，别混：

- **Bold 不需要单独 Key。** Regular 与 Bold 落在同一族里，直接在 Regular 的 Key 上加 `FontWeight="Bold"`，WPF 会挑真正的 Bold 字形，不是合成加粗：

```xml
<TextBlock FontFamily="{DynamicResource GlobalFont}" FontWeight="Bold" Text="真 Bold 字形" />
```

- **Medium / SemiBold 必须单独 Key。** 它们自成一族，在 Regular 的 Key 上写 `FontWeight="Medium"` 命中不到那个 ttf——`Inter 18pt 18pt` 族里只有 400 和 700 两个字面，请求 500 会退到最近的那个，拿不到真正的 Medium。要走 `GlobalFontMedium` / `GlobalFontBold` 这类专用 Key：

```xml
<TextBox FontFamily="{DynamicResource GlobalFontMedium}" FontSize="14" />
```

Medium 族里再叠 `FontWeight="Medium"` 是**安全**的：该族唯一字面的 `usWeightClass` 正好是 500，属精确命中，不会合成。控件样式本身带 `FontWeight="Medium"`（如 Material 按钮）时不必特意去掉。但别在 Medium 族上叠 `FontWeight="Bold"`，那个族里没有 700 字面，才会真的合成加粗发虚。用脚本可以直接读出字面的权重值核对：

```powershell
$g.Weight.ToOpenTypeWeight()   # Medium=500 / SemiBold=600 / Bold=700
```

回退链里两侧字重要对齐：`GlobalFontMedium` 必须是 `Inter ... Medium` 配 `Noto Sans SC Medium`，否则中英文混排时中文字重会跳。

### 用脚本读出准确的字体族名（强烈推荐，避免填错）

`#` 后填错会静默回退到系统字体且不报错。用**自带清单以外**的字体时（或字体换了版本），先跑一遍这个脚本，一次性列出整个文件夹里每个 ttf 的真实族名，再照抄进 `App.xaml`：

```powershell
Add-Type -AssemblyName PresentationCore
Get-ChildItem "Resources\Fonts\*.ttf" | ForEach-Object {
    $g = New-Object System.Windows.Media.GlyphTypeface ($_.FullName)
    "{0,-34} => {1}" -f $_.Name, ($g.Win32FamilyNames.Values -join ' | ')
}
```

输出里**族名相同的多个文件属于同一族**（靠 `FontWeight` 区分），族名带后缀的要各自建 Key，判断依据见上一节。

### 3. 使用

全局默认（窗口根节点，子控件继承）：

```xml
<Window ...
        TextElement.FontFamily="{DynamicResource GlobalFont}"
        TextOptions.TextFormattingMode="Ideal">
```

单控件指定：

```xml
<TextBlock Text="{Binding Price, StringFormat='{}{0:F2}'}"
           FontFamily="{DynamicResource MonoFont}" />
```

代码中：

```csharp
textBlock.FontFamily = (FontFamily)Application.Current.Resources["MonoFont"];
```

## 排查表（字体没生效几乎都在这里）

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 仍是系统默认字体 | `#` 后 Win32Family 名写错（大小写/空格） | 用上面脚本读真实 Win32Family |
| 仍是系统默认字体 | ttf 的 Build Action 不是 `Resource` | 改 `Resource` 重新编译 |
| 编译报错/找不到字体 | csproj 引用了不存在的 ttf | 删多余引用或补文件 |
| 中文显示方块/缺字 | 英文字体不含 CJK 且无回退 | `FontFamily` 追加中文字体做回退 |
| 粗体发虚/模糊 | 缺粗体字重 ttf，WPF 合成加粗 | 打包对应字重 ttf；Bold 与 Regular 同族，加 `FontWeight="Bold"` 即可 |
| `FontWeight="Medium"` 没效果/发虚 | Medium 是独立族，Regular 族里命中不到 | 改用 `GlobalFontMedium` 这类专用 Key |
| 个别控件不跟随全局字体（中文尤其明显） | 控件样式自己设了 `FontFamily`，覆盖了继承值 | 见下方「第三方控件库会覆盖字体」 |
| 中英文混排字重不一致 | 回退链两侧字重不对齐 | Medium 族只配 Medium 回退，SemiBold 只配 SemiBold |
| 用了某 Key 没定义 | `DynamicResource` 找不到 Key | 确认每个用到的 Key 都在 App.xaml 定义 |

> `DynamicResource` 找不到 Key **不会报错**，只会静默回退。排查优先核对两个字符串：资源 Key 名、`#` 后的 Win32Family 名。

## 第三方控件库会覆盖字体（MaterialDesignThemes 实例）

在根节点设了 `TextElement.FontFamily` 也不代表所有控件都跟随：**控件样式里的 `FontFamily` Setter 优先于继承值**。

MaterialDesignThemes 就是典型——它自带 Roboto，按钮等样式设了 `FontFamily="{DynamicResource MaterialDesignFont}"`（即 `#Roboto`）。Roboto 没有 CJK 字形，于是中文按钮文字回退到系统字体（微软雅黑），和页面其余用 Noto Sans SC 的文字并排时字形、字重明显不一致，而英文界面上却看不出问题，很容易漏掉。

修法二选一：

- **改单个/一类按钮**：建一个 `BasedOn` 原样式的派生样式，覆盖字体三项，再在用到的地方引用。可控，不影响其他页面：

```xml
<Style x:Key="RaisedButtonStyle" TargetType="Button"
       BasedOn="{StaticResource MaterialDesignRaisedButton}">
    <Setter Property="FontFamily" Value="{DynamicResource GlobalFontMedium}" />
    <Setter Property="FontWeight" Value="Medium" />
    <Setter Property="FontSize" Value="14" />
</Style>
```

- **全局改**：在 `App.xaml` 的 `MergedDictionaries` **之后**，用同名 Key 重定义 `MaterialDesignFont` 覆盖掉 Roboto，所有 Material 控件一起生效。但注意这个 Key 是通用正文字体，只能指向 Regular 族；那些样式里带 `FontWeight="Medium"` 的控件会因为 Regular 族没有 500 字面而落回 400，比原设计略细。

在自定义样式里写字体时用 `DynamicResource`：`Styles.xaml` 这类被合并的字典在解析时还看不到 `App.Resources` 里的字体 Key，`StaticResource` 会直接失败。

两种按钮并排时，除了 `FontFamily` 还要对齐 `FontSize`——库样式通常显式设了 14，而自定义样式不设就继承祖先（WPF 默认 12），只统一字体仍会一大一小。

## 为什么嵌入而非系统字体

跨机器一致、部署即用；等宽字体（JetBrains Mono）保证数字列对齐。代价是每个 ttf 数 MB，按需只打包用到的字重。
