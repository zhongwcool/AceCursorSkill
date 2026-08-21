---
name: wpf-project-init
description: >-
  Initialize a .NET WPF desktop project with Generic Host DI, Serilog (AppData
  logs), CommunityToolkit.Mvvm, MaterialDesignThemes, LottieSharp, MailKit,
  and Per-Monitor DPI foundation (app.manifest PerMonitorV2 + UseLayoutRounding +
  DpiTextRenderingHelper). Use when creating a new WPF app, bootstrapping
  project infrastructure, or when the user asks to set up WPF with dependency
  injection, Serilog, MVVM, Material Design, or high-DPI / 多分辨率基础配置.
---

# WPF 项目初始化

将空白或新建的 WPF 项目升级为带完整基础设施的 MVVM 应用。参考实现：AceReader。

## 前置确认

从用户消息或项目名推断 `{AppName}`（PascalCase，如 `AceReader`）和 `{Namespace}`（通常与项目名相同）。

若项目不存在，先创建：

```bash
dotnet new wpf -n {AppName} -f net10.0-windows
```

若已有 WPF 项目，先读 `*.csproj` 确认 `TargetFramework` 与目录结构，避免覆盖用户业务代码。

## 工作流清单

```
- [ ] 1. 添加 NuGet 包
- [ ] 2. 创建目录与基础文件（含 app.manifest + DpiTextRenderingHelper）
- [ ] 3. 改造 App.xaml / App.xaml.cs（Host + Serilog + DI）
- [ ] 4. 迁移 MainWindow 到 Views/，接入 MVVM + DPI 窗口属性
- [ ] 5. 配置 appsettings.json 与 csproj（含 ApplicationManifest）
- [ ] 6. 更新 .gitignore（.idea/）
- [ ] 7. 生成简易 README.md
- [ ] 8. dotnet build 验证（0 错误）
```

## 1. NuGet 包

在项目目录执行（版本取 NuGet 最新稳定版即可）：

```bash
dotnet add package Microsoft.Extensions.Hosting
dotnet add package Serilog.Extensions.Hosting
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Settings.Configuration
dotnet add package CommunityToolkit.Mvvm
dotnet add package MaterialDesignThemes
dotnet add package LottieSharp
dotnet add package MailKit
```

## 2. 目标目录结构

```
{AppName}/
├── App.xaml / App.xaml.cs
├── app.manifest             # PerMonitorV2，必做
├── appsettings.json
├── AssemblyInfo.cs
├── Common/AppPaths.cs
├── Helpers/DpiTextRenderingHelper.cs
├── DependencyInjection/ServiceCollectionExtensions.cs
├── Services/Email/          # EmailOptions, IEmailService, EmailService
├── ViewModels/              # ViewModelBase, MainViewModel
├── Views/                   # MainWindow.xaml(.cs)
├── Assets/                  # Lottie JSON（可选）
└── Resources/               # 图标等
```

删除根目录旧的 `MainWindow.xaml(.cs)`（若存在），视图统一放 `Views/`。

## 3. 核心约定

| 项 | 约定 |
|----|------|
| DI 入口 | `Host.CreateDefaultBuilder()`，`App.Services` 暴露 `IServiceProvider` |
| 服务注册 | `DependencyInjection/ServiceCollectionExtensions.cs`，方法名 `Add{AppName}Services` |
| 日志目录 | `%LOCALAPPDATA%\{AppName}\logs\`，按天滚动，保留 31 天 |
| 主窗口 | 从 DI 解析 `MainWindow`，**不用** `StartupUri` |
| MVVM | `CommunityToolkit.Mvvm` 源生成器：`[ObservableProperty]`、`[RelayCommand]` |
| UI 主题 | MaterialDesign 5.x：`BundledTheme` + `MaterialDesign3.Defaults.xaml` |
| 配置 | `appsettings.json` → `CopyToOutputDirectory: PreserveNewest` |
| DPI | `app.manifest` 声明 `PerMonitorV2`；每个独立 `Window` 设 `UseLayoutRounding="True"` 并调用 `DpiTextRenderingHelper.Attach(this)`。**不要在 XAML 写死 `TextFormattingMode="Ideal"`**。后续发虚排查见 `wpf-multi-dpi-adaptation` skill。 |

`AppPaths.AppName` 必须与 `{AppName}` 一致。

## 4. csproj 补充项

```xml
<PropertyGroup>
  <ApplicationManifest>app.manifest</ApplicationManifest>
</PropertyGroup>

<ItemGroup>
  <None Update="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
  <!-- Lottie 首页时添加 -->
  <Content Include="Assets\{lottie-file}.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>

<ItemGroup>
  <Page Update="Views\MainWindow.xaml">
    <Generator>MSBuild:Compile</Generator>
    <XamlRuntime>Wpf</XamlRuntime>
    <SubType>Designer</SubType>
  </Page>
</ItemGroup>
```

`ApplicationIcon` 仅在 `Resources\main.ico` 存在时配置，否则省略以免编译失败。

## 5. 首页模式（二选一）

**默认（Lottie 占位）**：用户指定 JSON 文件名（如 `man_and_robot.json`）时：
- 放入 `Assets/`
- `MainViewModel.AnimationPath` = `Path.Combine(AppContext.BaseDirectory, "Assets", "{file}.json")`
- `MainWindow.xaml` 使用 `LottieAnimationView`：`xmlns:lottie="clr-namespace:LottieSharp.WPF;assembly=LottieSharp"`，`AutoPlay="True"`，`RepeatCount="-1"`

**简易欢迎页**：无 Lottie 资源时，用 MaterialDesign 文案 + 刷新按钮占位。

## 6. .gitignore

若未忽略 JetBrains 配置，追加：

```gitignore
# JetBrains Rider / IntelliJ
.idea/
```

## 7. README

生成根目录 `README.md`：项目简介、技术栈表、环境要求、`dotnet build/run`、目录结构、日志路径、配置说明。

## 8. 验证

```bash
dotnet build {AppName}/{AppName}.csproj
```

可选运行数秒，确认日志写入 `%LOCALAPPDATA%\{AppName}\logs\`。

## 命名替换规则

创建文件时全局替换：

| 占位符 | 示例 |
|--------|------|
| `{AppName}` | AceReader |
| `{Namespace}` | AceReader |
| `{AddServicesMethod}` | AddAceReaderServices |

## 完整文件模板

各文件的完整代码见 [templates.md](templates.md)。按模板生成后做命名替换，再根据首页模式调整 `MainWindow` / `MainViewModel`。

## 注意事项

- 不提交 `.idea/`、`.env`、密钥
- 邮件服务依赖 `appsettings.json` 的 `Email` 节点，默认留空
- Lottie `FileName` 绑定绝对路径；资源必须复制到输出目录
- 保持改动最小，不添加用户未请求的业务逻辑
- DPI 基础三项（manifest / UseLayoutRounding / DpiTextRenderingHelper）属于脚手架，不是可选优化；Viewbox、DropShadow、小数倍缩放等长尾问题仍按 `wpf-multi-dpi-adaptation` 排查，不要把整份适配文档塞进 init
