# WPF 初始化 — 文件模板

生成时将 `{Namespace}`、`{AppName}` 替换为实际项目名。

---

## app.manifest

```xml
<?xml version="1.0" encoding="utf-8"?>
<assembly manifestVersion="1.0" xmlns="urn:schemas-microsoft-com:asm.v1">
    <assemblyIdentity version="1.0.0.0" name="{AppName}.app"/>
    <trustInfo xmlns="urn:schemas-microsoft-com:asm.v2">
        <security>
            <requestedPrivileges xmlns="urn:schemas-microsoft-com:asm.v3">
                <requestedExecutionLevel level="asInvoker" uiAccess="false"/>
            </requestedPrivileges>
        </security>
    </trustInfo>
    <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1">
        <application>
            <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}"/>
        </application>
    </compatibility>
    <application xmlns="urn:schemas-microsoft-com:asm.v3">
        <windowsSettings>
            <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2</dpiAwareness>
            <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">true/pm</dpiAware>
        </windowsSettings>
    </application>
</assembly>
```

csproj 必须引用：`<ApplicationManifest>app.manifest</ApplicationManifest>`。

---

## Helpers/DpiTextRenderingHelper.cs

每个独立 `Window` 构造函数里调用 `DpiTextRenderingHelper.Attach(this)`。不要在 XAML 写死 `TextFormattingMode`。

```csharp
using System.Windows;
using System.Windows.Media;

namespace {Namespace}.Helpers;

/// <summary>
/// 100% 缩放用 Display（小字锐利），高于 100% 用 Ideal（矢量排版）。跨屏 DpiChanged 时自动切换。
/// </summary>
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
```

---

## appsettings.json

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    }
  },
  "Email": {
    "SmtpHost": "",
    "SmtpPort": 587,
    "UseSsl": true,
    "UserName": "",
    "Password": "",
    "SenderName": "{AppName}",
    "SenderAddress": ""
  }
}
```

---

## Common/AppPaths.cs

```csharp
using System.IO;

namespace {Namespace}.Common;

public static class AppPaths
{
    public const string AppName = "{AppName}";

    public static string AppDataRoot { get; } = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        AppName);

    public static string LogsDirectory { get; } = Path.Combine(AppDataRoot, "logs");
    public static string LogFilePath { get; } = Path.Combine(LogsDirectory, "log-.txt");

    public static void EnsureCreated()
    {
        Directory.CreateDirectory(AppDataRoot);
        Directory.CreateDirectory(LogsDirectory);
    }
}
```

---

## App.xaml

```xml
<Application x:Class="{Namespace}.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <materialDesign:BundledTheme BaseTheme="Light"
                                             PrimaryColor="DeepPurple"
                                             SecondaryColor="Lime" />
                <ResourceDictionary Source="pack://application:,,,/MaterialDesignThemes.Wpf;component/Themes/MaterialDesign3.Defaults.xaml" />
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

---

## App.xaml.cs

```csharp
using System.Windows;
using {Namespace}.Common;
using {Namespace}.DependencyInjection;
using {Namespace}.Views;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;

namespace {Namespace};

public partial class App : Application
{
    private IHost? _host;

    public static IServiceProvider Services =>
        ((App)Current)._host?.Services
        ?? throw new InvalidOperationException("Host 尚未初始化。");

    protected override async void OnStartup(StartupEventArgs e)
    {
        AppPaths.EnsureCreated();

        _host = Host.CreateDefaultBuilder()
            .UseContentRoot(AppContext.BaseDirectory)
            .ConfigureAppConfiguration((_, config) =>
            {
                config.SetBasePath(AppContext.BaseDirectory);
                config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
            })
            .UseSerilog((context, services, loggerConfiguration) =>
            {
                loggerConfiguration
                    .ReadFrom.Configuration(context.Configuration)
                    .ReadFrom.Services(services)
                    .Enrich.FromLogContext()
                    .WriteTo.Console()
                    .WriteTo.File(
                        path: AppPaths.LogFilePath,
                        rollingInterval: RollingInterval.Day,
                        retainedFileCountLimit: 31,
                        shared: true,
                        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}");
            })
            .ConfigureServices((context, services) =>
            {
                services.Add{AppName}Services(context.Configuration);
            })
            .Build();

        await _host.StartAsync();
        Log.Information("应用启动，日志目录：{LogDir}", AppPaths.LogsDirectory);

        _host.Services.GetRequiredService<MainWindow>().Show();
        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        Log.Information("应用退出。");
        if (_host is not null)
        {
            await _host.StopAsync(TimeSpan.FromSeconds(5));
            _host.Dispose();
        }
        await Log.CloseAndFlushAsync();
        base.OnExit(e);
    }
}
```

---

## DependencyInjection/ServiceCollectionExtensions.cs

```csharp
using {Namespace}.Services.Email;
using {Namespace}.ViewModels;
using {Namespace}.Views;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace {Namespace}.DependencyInjection;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection Add{AppName}Services(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddOptions<EmailOptions>()
            .Bind(configuration.GetSection(EmailOptions.SectionName));
        services.AddSingleton<IEmailService, EmailService>();
        services.AddTransient<MainViewModel>();
        services.AddSingleton<MainWindow>();
        return services;
    }
}
```

---

## ViewModels/ViewModelBase.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace {Namespace}.ViewModels;

public abstract partial class ViewModelBase : ObservableObject;
```

---

## ViewModels/MainViewModel.cs（Lottie 首页）

```csharp
using System.IO;
using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.Logging;

namespace {Namespace}.ViewModels;

public partial class MainViewModel : ViewModelBase
{
    private readonly ILogger<MainViewModel> _logger;

    [ObservableProperty]
    private string _title = "{AppName}";

    public string AnimationPath { get; } = Path.Combine(
        AppContext.BaseDirectory, "Assets", "{LottieFileName}.json");

    public MainViewModel(ILogger<MainViewModel> logger)
    {
        _logger = logger;
        _logger.LogInformation("MainViewModel 已创建，动画路径：{AnimationPath}", AnimationPath);
    }
}
```

## ViewModels/MainViewModel.cs（欢迎页）

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.Logging;

namespace {Namespace}.ViewModels;

public partial class MainViewModel : ViewModelBase
{
    private readonly ILogger<MainViewModel> _logger;

    [ObservableProperty]
    private string _title = "{AppName}";

    [ObservableProperty]
    private string _greeting = "欢迎使用 {AppName}";

    public MainViewModel(ILogger<MainViewModel> logger)
    {
        _logger = logger;
    }

    [RelayCommand]
    private void Refresh()
    {
        Greeting = $"刷新于 {DateTime.Now:HH:mm:ss}";
        _logger.LogInformation("执行刷新命令。");
    }
}
```

---

## Views/MainWindow.xaml（Lottie 首页）

```xml
<Window x:Class="{Namespace}.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:{Namespace}.ViewModels"
        xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes"
        xmlns:lottie="clr-namespace:LottieSharp.WPF;assembly=LottieSharp"
        TextElement.Foreground="{DynamicResource MaterialDesignBody}"
        Background="{DynamicResource MaterialDesignPaper}"
        FontFamily="{materialDesign:MaterialDesignFont}"
        TextOptions.TextRenderingMode="Auto"
        UseLayoutRounding="True"
        WindowStartupLocation="CenterScreen"
        Title="{Binding Title}" Height="450" Width="800">
    <Grid>
        <lottie:LottieAnimationView HorizontalAlignment="Center"
                                    VerticalAlignment="Center"
                                    Width="360" Height="360"
                                    AutoPlay="True" RepeatCount="-1"
                                    FileName="{Binding AnimationPath}" />
    </Grid>
</Window>
```

## Views/MainWindow.xaml（欢迎页）

```xml
<Window x:Class="{Namespace}.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:{Namespace}.ViewModels"
        xmlns:materialDesign="http://materialdesigninxaml.net/winfx/xaml/themes"
        TextElement.Foreground="{DynamicResource MaterialDesignBody}"
        Background="{DynamicResource MaterialDesignPaper}"
        FontFamily="{materialDesign:MaterialDesignFont}"
        TextOptions.TextRenderingMode="Auto"
        UseLayoutRounding="True"
        WindowStartupLocation="CenterScreen"
        Title="{Binding Title}" Height="450" Width="800">
    <Grid>
        <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
            <TextBlock Text="{Binding Greeting}"
                       Style="{StaticResource MaterialDesignHeadline5TextBlock}"
                       Margin="0,0,0,24" />
            <Button Content="刷新"
                    Command="{Binding RefreshCommand}"
                    Style="{StaticResource MaterialDesignRaisedButton}" />
        </StackPanel>
    </Grid>
</Window>
```

---

## Views/MainWindow.xaml.cs

```csharp
using System.Windows;
using {Namespace}.Helpers;
using {Namespace}.ViewModels;

namespace {Namespace}.Views;

public partial class MainWindow : Window
{
    public MainWindow(MainViewModel viewModel)
    {
        InitializeComponent();
        DpiTextRenderingHelper.Attach(this);
        DataContext = viewModel;
    }
}
```

---

## Services/Email/

标准三文件：`EmailOptions.cs`（`SectionName = "Email"`）、`IEmailService.cs`、`EmailService.cs`（MailKit + `IOptions<EmailOptions>` + `ILogger`）。实现与 AceReader 项目一致，命名空间改为 `{Namespace}.Services.Email`。

---

## AssemblyInfo.cs

```csharp
using System.Windows;

[assembly: ThemeInfo(
    ResourceDictionaryLocation.None,
    ResourceDictionaryLocation.SourceAssembly)]
```
