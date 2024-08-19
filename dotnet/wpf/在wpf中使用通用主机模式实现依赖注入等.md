### 在 WPF 程序中使用通用主机 IHost

> 熟悉.NET Web Api 开发的小伙伴肯定知道依赖注入,日志,配置文件等功能.可谓是非常的舒服,整个开发配合 C# 优雅的语法与.NET 的性能,写代码也没什么心理压力.最近在公司上班需要制作一个 WPF 程序,那么这些东西能否带入到 WPF 呢?

既然已经写了这篇文章,那么上面的问题答案肯定是可以的.接下来就进入正题直接介绍如何做.

- 在 WPF 项目中若是我们不自己去实现那么就得依赖第三方或者.NET 为我们提供的功能,这里我选则使用 IHost 通用主机.

了解.NET Core 开发的小伙伴肯定对这个东西也是熟悉的很.所以不多说什么.接下来直接进入代码.

#### 创建项目

- 使用 VS 创建一个基于.NET8 的 WPF 项目, .NET9 也行,反正都一样.但必须得是 .NET 的而不是旧的 .Net Framework 的.
- 别的名称以及保存路径根据自己的需要来就行,我这里名称就起 WpfAutoDISample

#### 安装依赖包

- 为了使用 IHost 以及方便使用,我们需要安装几个依赖包
- EasilyNET.AutoDependencyInjection 这是我和小伙伴合作维护的一个用来实现服务模块化注入的一个库,感兴趣的可以去看看.
- Microsoft.Extensions.Hosting 这个不用多说
- CommunityToolkit.Mvvm 这是为了实现 MVVM 的工具库

#### 改造项目以支持 IHost

- 改造项目的启动方式.

  > WPF 默认直接从 App 类进行启动,并且没有显示的 Main 函数,所以这里我们需要调整下项目,让 Main 函数为我们自己的.当然也可以使用添加 Program.cs 类的方式来创建,这里就不用这种方式,直接写在 App 里.

- 在项目的 .csproj 文件中添加如下内容.

```xml
<ItemGroup>
	<ApplicationDefinition Remove="App.xaml" />
	<Page Include="App.xaml" />
</ItemGroup>
```

- 这个配置的作用是将 App.xaml 从应用程序定义文件中移除,并将其重新包含为一个普通的页面文件.这是为了改变 App.xaml 的处理方式以解决特定的构建问题.
- 同时我们后期的程序加载页面的方式也会发生变化,所以这里一并将 App.xaml 文件也修改一下.移除掉其中的 `StartupUri="MainWindow.xaml"`

```xml
<Application x:Class="WpfAutoDISample.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:WpfAutoDISample">
             <!--删除掉这一句-->
             <!--StartupUri="MainWindow.xaml"-->
    <Application.Resources>
        <ResourceDictionary>

        </ResourceDictionary>
    </Application.Resources>
</Application>
```

- 为我们的程序添加 Main 函数.打开 `App.xaml.cs` 文件,并在其中写入 Main 函数.该函数必须使用 `[STAThread]` 标记,并且只能是 static 类型,不能使用异步的版本.我已经试过了不行.

```csharp
/// <summary>
/// Main入口函数
/// </summary>
/// <param name="args"></param>
/// <returns></returns>
[STAThread]
public static void Main(string[] args)
{
}
```

- 准备工作做好后,则可以开始我们的主要任务.
- 首先添加一个 `appsettings.json` 用于做配置文件,起别的名字也行,只是这个名字比较亲切而已.我这里简单的在里面定义一下日志的配置就行.其他后期根据需要添加.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Trace"
    }
  }
}
```

- 然后添加我们的 IHost 通用主机

```csharp
private static IHostBuilder CreateHostBuilder(params string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration(c =>
        {
            c.SetBasePath(AppContext.BaseDirectory);
            c.AddJsonFile("appsettings.json", false, false);
        })
        .ConfigureLogging((_, lb) =>
        {
            lb.ClearProviders();
            // 这里可以配置日志
            lb.AddDebug();
        })
        .ConfigureServices(sc =>
        {
            // 在这里配置服务
            sc.AddApplicationModules<AppServiceModules>();
            sc.AddSingleton(_ => Current.Dispatcher);
        });
```

- 由于`AppServiceModules.cs`中暂时没有需要服务配置,这里直接新建一个文件用来占位,当然也可以不要.

```csharp
[DependsOn(typeof(DependencyAppModule))]
internal sealed class AppServiceModules : AppModule;
```

- 为了展示效果,我们给`MainWindow.xaml`中添加一点元素.

```xml
<Window x:Class="WpfAutoDISample.MainWindow"
                 xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                 xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                 xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
                 xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
                 xmlns:viewModels="clr-namespace:WpfAutoDISample.ViewModels"
                 d:DataContext="{d:DesignInstance viewModels:MainWindowModel}"
                 mc:Ignorable="d"
                 Title="MainWindow" Height="600" Width="800" Background="DarkGray"
                 WindowStartupLocation="CenterScreen">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="100"></RowDefinition>
            <RowDefinition Height="*"></RowDefinition>
        </Grid.RowDefinitions>
        <TextBlock FontSize="32" HorizontalAlignment="Center" VerticalAlignment="Center" Text="{Binding Message}" Width="300" Height="63"/>
        <Button Grid.Row="1" Margin="351,0,319,228" FontSize="24" Click="OnBtnClick" Content="ClickMe" VerticalAlignment="Bottom" />
    </Grid>
</Window>
```

- 在上面的窗体中,我们添加了一个文本,和一个按钮,并使用了 Binding 将文本的数据绑定到了 ViewModel 的 Message 属性上面.

- 接下来为`MainWindow.xaml.cs`添加如下特性.以便我们的依赖注入库能自动注入该类.

```csharp
[DependencyInjection(ServiceLifetime.Singleton, AddSelf = true)]
public partial class MainWindow
{
    /// <inheritdoc />
    public MainWindow(MainWindowModel mwm)
    {
        InitializeComponent();
        DataContext = mwm;
    }

    private async void OnBtnClick(object sender, RoutedEventArgs e)
    {
        if (DataContext is not MainWindowModel vm) return;
        vm.Message = "Clicked";
        await Task.CompletedTask;
    }
}
```

- 同样我们也为 MainWindow 的 ViewModel 设置同样的自动注入.

```csharp
[DependencyInjection(ServiceLifetime.Singleton, AddSelf = true)]
public partial class MainWindowModel : ObservableObject
{
    /// <summary>
    /// Message
    /// </summary>
    [ObservableProperty]
    private string message = "Hello WPF!";
}
```

- 当一切写好后,我们再回到 `App.xaml.cs` 的 Main 函数中完成我们的代码.

```csharp
[STAThread]
public static void Main(string[] args)
{
    // 创建一个通用主机
    using var host = CreateHostBuilder(args).Build();
    host.InitializeApplication();
    host.StartAsync().ConfigureAwait(true).GetAwaiter().GetResult();

    // 配置主窗口
    var app = new App();
    app.InitializeComponent();
    app.MainWindow = host.Services.GetRequiredService<MainWindow>();
    app.MainWindow.Visibility = Visibility.Visible;
    app.Run();
    host.StopAsync().ConfigureAwait(true).GetAwaiter().GetResult();
}
```

- 完成上面的操作后,我们直接 F5 启动,那么一切正常的情况下,应该是能够正常启动,并打开我们窗体.
- 上面的`App.xaml.cs`中的代码还太简单,并不方便我们后续在其他地方的使用,所以我们为他添加一些公共的属性.

```csharp
private IHost AppHost { get; set; }

/// <summary>
/// Services
/// </summary>
public static IServiceProvider Services => CurrentAppHost.Services;

/// <summary>
/// 获取当前AppHost
/// </summary>
private static IHost CurrentAppHost => (Current as App)?.AppHost ?? throw new InvalidOperationException("无法获取AppHost，当前Application实例不是App类型。");
```

- 同样,添加了东西后,Main 函数还需进一步修改一下,并为其添加启动和退出事件,便于程序启动启动服务以及退出后释放资源.

```csharp
[STAThread]
public static void Main(string[] args)
{
    // 创建一个通用主机
    using var host = CreateHostBuilder(args).Build();
    host.InitializeApplication();
    // 配置主窗口
    var app = new App();
    app.InitializeComponent();
    app.AppHost = host;
    app.MainWindow = CurrentAppHost.Services.GetRequiredService<MainWindow>();
    app.MainWindow.Visibility = Visibility.Visible;
    app.Run();
}

protected override async void OnExit(ExitEventArgs e)
{
    await AppHost.StopAsync().ConfigureAwait(false);
    AppHost.Dispose();
    Shutdown();
    base.OnExit(e);
}

protected override async void OnStartup(StartupEventArgs e)
{
    await AppHost.StartAsync().ConfigureAwait(false);
    base.OnStartup(e);
}
```

- 到此本文的内容就结束了,代码稍后会同步到 GitHub ,虽然这些内容已经满足基本的需求,但是还有一些没有解决,比如全局异常捕获.当程序内容增多后,启动较慢的时候如何新增一个启动窗口,类似 VS 和大多数软件那样.这些后面再开新的文章来写.
- 代码地址: https://github.com/joesdu/WpfAutoDISample
