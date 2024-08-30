## 为 WPF 程序添加更多实用小技巧

> 上一篇关于 WPF 的文章写了后,我们创建了一个简单的 WPF 程序.并为其添加了 IHost 通用主机,并制作了一个简单的 demo 程序.但是其中还有不少的问题,想让我们的程序健壮以及代码设计合理,还需更多的优化,这篇文章就继续来分享我在项目中的一些做法.

#### 添加全局异常处理

> 未处理的异常通常是致命的,在 WebApi 这种程序中有全局的异常过滤器或者使用中间件的方式来处理全局异常,让程序即使发生异常也不至于崩溃退出.保证其他功能能正常使用.那桌面程序也一样.当发生未处理的异常时候,可能会毫无征兆的闪退,或者卡死崩溃等情况,用户体验极差.

那么针对这种问题,我们就会想能否有一种方式能让 WPF 程序也能全局捕获未处理的异常,来避免程序闪退等问题.接下来就是实现的原理以及代码.

##### 原理

在 WPF 这种桌面程序中,异常主要来自于如下三个方面

| 来源                                 | 触发条件                                        | 场景                                                                            |
| ------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------------------------- |
| 应用程序域范围内未捕获的异常         | 当应用程序域中的任何线程抛出未捕获的异常时触发  | 适用于捕获所有未处理的异常,包括那些在非 UI 线程中发生的异常                     |
| WPF 应用程序中 UI 线程中未捕获的异常 | 当 WPF 应用程序的 UI 线程抛出未捕获的异常时触发 | 适用于捕获在 UI 线程中发生的异常,例如按钮点击事件处理程序中的异常               |
| 并行库(TPL)中未捕获的异常            | 当任务并行库中的任务抛出异常且未被观察到时触发  | 适用于捕获异步任务中发生的异常,例如使用 async 和 await 关键字的异步方法中的异常 |

针对这 3 种异常.NET WPF 中也提供了 3 个事件来处理.`AppDomain.CurrentDomain.UnhandledException`,`DispatcherUnhandledException`以及`TaskScheduler.UnobservedTaskException`.

接下来就是使用代码来实现针对全局异常的处理.在 `App.xaml.cs` 中重写 `OnStartup` 函数代码,在其中注册这三种异常的事件以及启动我们之前的 `IHost`.

```csharp
// 程序退出时执行资源的释放和关闭操作
protected override async void OnExit(ExitEventArgs e)
{
    AppDomain.CurrentDomain.UnhandledException -= CurrentDomainUnhandledException;
    DispatcherUnhandledException -= AppDispatcherUnhandledException;
    TaskScheduler.UnobservedTaskException -= TaskSchedulerOnUnobservedTaskException;
    await AppHost.StopAsync().ConfigureAwait(false);
    AppHost.Dispose();
    // 这里不要忘记释放
    WinApis._mutex.ReleaseMutex();
    Shutdown();
    base.OnExit(e);
}
// 程序启动时候的时候执行的操作
protected override async void OnStartup(StartupEventArgs e)
{
    AppDomain.CurrentDomain.UnhandledException += CurrentDomainUnhandledException;
    DispatcherUnhandledException += AppDispatcherUnhandledException;
    TaskScheduler.UnobservedTaskException += TaskSchedulerOnUnobservedTaskException;
    // 在 OnStart 中启动 IHost 通用主机
    await AppHost.StartAsync().ConfigureAwait(false);
    base.OnStartup(e);
}

private void CurrentDomainUnhandledException(object sender, UnhandledExceptionEventArgs e)
{
    HandleException(e.ExceptionObject as Exception);
}

private void TaskSchedulerOnUnobservedTaskException(object? sender, UnobservedTaskExceptionEventArgs e)
{
    HandleException(e.Exception);
    e.SetObserved();
}

private void AppDispatcherUnhandledException(object sender, DispatcherUnhandledExceptionEventArgs e)
{
    HandleException(e.Exception);
    e.Handled = true;
}

private void HandleException(Exception? exception)
{
    if (exception is null) return;
    _logger.LogError(exception, "Unhandled exception occurred.");
    //MessageBox.Show("发生未处理的异常，程序将继续运行。", "错误", MessageBoxButton.OK, MessageBoxImage.Error);
}
```

通过上述代码即可实现捕获全局异常,但是这样在调试的时候就不太方便了,有些异常可能不会被抛出来,所以在我们的服务配置中,配置日志的地方还需加上`AddDebug`这样可以在 VS 的调试窗口输出错误信息.

```csharp
Host.CreateDefaultBuilder(args)
    .ConfigureLogging((_, lb) =>
    {
        // 添加输出到调试控制台
        lb.AddDebug();
    })
```

#### 让 APP 只能单例运行

> 我们的 WPF 程序有时候会访问和缓存一些本地数据,如 LiteDB 或者 SQLite 这样的本地数据库,但是这种数据库往往支持单实例访问,比如我们经常遇到某些文件被别的程序编辑,就打不开或者无法修改的问题.所以为了避免我们程序自己多实例竞争导致的文件占用,需要让我们的 APP 只能有一个实例运行,当再次点击快捷方式启动程序的时候将之前的实例置顶,并退出新打开的 APP.

为了让 WPF 程序也能实现这样的功能,我们则需要用到 Windows 的 API 了.利用 `user32.dll`中的`SetForegroundWindow`,`ShowWindowAsync`和`IsIconic`这三个函数.这三个函数分别能将指定窗口设置为前台窗口,设置窗口的显示状态和判断窗口是否最小化,同时我们还需要使用到`System.Threading.Mutex`这个类,这个能确保我们的程序只运行一个实例,用于在多个线程或进程之间协调对共享资源的访问.接下来上代码.

- 导入或定义好我们需要的函数和类型
- 这里需要注意 `GlobalMutexName` 需要按需调整我们程序本身的程序集名称.当然也可以通过反射的方式来获取.
- `private static readonly string? GlobalMutexName = Assembly.GetExecutingAssembly().GetName().Name;`
- 我这里新增一个名为 `WinApis.cs` 的类文件来写这些代码

```csharp
// 这里需要按需调整为我们程序的程序集名称
// private const string GlobalMutexName = "WpfAutoDISample";
private static readonly string? GlobalMutexName = Assembly.GetExecutingAssembly().GetName().Name;

private const int SW_RESTORE = 9;
private static Mutex _mutex;
/// <summary>
/// 将指定窗口设置为前台窗口。
/// </summary>
/// <param name="hWnd">窗口句柄。</param>
/// <returns>如果窗口被成功设置为前台窗口，则返回true；否则返回false。</returns>
[LibraryImport("user32.dll", EntryPoint = "SetForegroundWindow")]
[return: MarshalAs(UnmanagedType.Bool)]
private static partial bool SetForegroundWindow(IntPtr hWnd);
/// <summary>
/// 设置窗口的显示状态。
/// </summary>
/// <param name="hWnd">窗口句柄。</param>
/// <param name="nCmdShow">指定窗口的显示状态的标志。可以是SW_RESTORE等值。</param>
/// <returns>如果窗口显示状态被成功设置，则返回true；否则返回false。</returns>
[LibraryImport("user32.dll", EntryPoint = "ShowWindowAsync")]
[return: MarshalAs(UnmanagedType.Bool)]
private static partial bool ShowWindowAsync(IntPtr hWnd, int nCmdShow);
/// <summary>
/// 判断指定窗口是否最小化。
/// </summary>
/// <param name="hWnd">窗口句柄。</param>
/// <returns>如果窗口最小化，则返回true；否则返回false。</returns>
[LibraryImport("user32.dll", EntryPoint = "IsIconic")]
[return: MarshalAs(UnmanagedType.Bool)]
private static partial bool IsIconic(IntPtr hWnd);
```

- 实现我们的`EnsureSingleInstance`和`BringExistingInstanceToFront`函数

```csharp
/// <summary>
/// 确保应用程序只运行一个实例
/// </summary>
/// <returns></returns>
internal static void EnsureSingleInstance(out bool createdNew)
{
    _mutex = new(true, GlobalMutexName, out createdNew);
}

/// <summary>
/// 查找已运行的实例并将其前置
/// </summary>
internal static void BringExistingInstanceToFront()
{
    // 查找已经运行的实例的主窗口
    var currentProcess = Process.GetCurrentProcess();
    foreach (var process in Process.GetProcessesByName(currentProcess.ProcessName))
    {
        if (process.Id == currentProcess.Id) continue;
        var hWnd = process.MainWindowHandle;
        if (IsIconic(hWnd))
        {
            _ = ShowWindowAsync(hWnd, SW_RESTORE);
        }
        _ = SetForegroundWindow(hWnd);
        break;
    }
}
```

- 调整 `Main` 函数来确保我们程序启动时进行检查

```csharp
[STAThread]
public static void Main(string[] args)
{
    WinApis.EnsureSingleInstance(out var createdNew);
    if (!createdNew)
    {
        WinApis.BringExistingInstanceToFront();
        // 退出当前实例
        return;
    }
    // ...其他代码
}
```

- 通过上述代码即可实现我们想要的功能.

#### 为我们的程序加上类似 VS 的启动界面

> 当添加了 IHost 后,我们程序启动时干的事情更多了,而且后期加入了更多的功能,或者迫于公司业务别的依赖库引入了 Autofac,并且我们需要使用 Autofac 后,程序将会变得第一次启动时间超级长,反正我是用了领导写的库后,不得不用 Autofac.从本来秒起的程序变成了第一次启动可能超过一分钟.

- 基于上述原因,我需要为我的程序添加一个启动界面来提升体验.
- 接下来添加一个新的窗体.就叫他 `Loading` 吧.
- 通过 VS 的向导添加 `Loading` 后,我们为了界面好看来对他的效果和样式做一些修改.代码如下:

```xml
<Window x:Class="WpfAutoDISample.Loading"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        WindowStartupLocation="CenterScreen"
        WindowStyle="None"
        AllowsTransparency="True"
        Title="Loading" Height="378" Width="672">
    <Window.Background>
        <ImageBrush ImageSource="pack://application:,,,/Assets/Image/LoadingBackground.png"
                    TileMode="FlipXY"
                    Stretch="None"
                    AlignmentX="Center"
                    AlignmentY="Center"/>
    </Window.Background>
    <Window.Resources>
        <Style x:Key="DotStyle" TargetType="Ellipse">
            <Setter Property="Width" Value="5"/>
            <Setter Property="Height" Value="5"/>
            <Setter Property="Fill" Value="WhiteSmoke"/>
        </Style>
        <Storyboard x:Key="LoadingAnimation" RepeatBehavior="Forever">
            <DoubleAnimationUsingKeyFrames Storyboard.TargetProperty="(UIElement.RenderTransform).(RotateTransform.Angle)">
                <EasingDoubleKeyFrame KeyTime="0:0:0" Value="0"/>
                <EasingDoubleKeyFrame KeyTime="0:0:1" Value="360"/>
            </DoubleAnimationUsingKeyFrames>
        </Storyboard>
        <FontFamily x:Key="HuaWenXingKaiFont">
            <![CDATA[pack://application:,,,/WpfAutoDISample;component/Assets/Fonts/#华文行楷]]>
        </FontFamily>
    </Window.Resources>
    <Grid Background="Transparent">
        <StackPanel HorizontalAlignment="Right" VerticalAlignment="Bottom" Orientation="Horizontal" Margin="0,0,20,10">
            <TextBlock Text="正在启动,请稍后" FontFamily="{StaticResource HuaWenXingKaiFont}" Margin="0,0,20,0" FontSize="20" Foreground="WhiteSmoke" FontWeight="DemiBold" VerticalAlignment="Center"/>
            <Canvas Width="50" Height="50">
                <Canvas.RenderTransform>
                    <RotateTransform CenterX="25" CenterY="25"/>
                </Canvas.RenderTransform>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="22.5" Canvas.Top="5"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="35" Canvas.Top="10"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="40" Canvas.Top="22.5"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="35" Canvas.Top="35"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="22.5" Canvas.Top="40"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="10" Canvas.Top="35"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="5" Canvas.Top="22.5"/>
                <Ellipse Style="{StaticResource DotStyle}" Canvas.Left="10" Canvas.Top="10"/>
                <Canvas.Triggers>
                    <EventTrigger RoutedEvent="FrameworkElement.Loaded">
                        <BeginStoryboard Storyboard="{StaticResource LoadingAnimation}"/>
                    </EventTrigger>
                </Canvas.Triggers>
            </Canvas>
        </StackPanel>
    </Grid>
</Window>

```

- 不想截图了,针对上面的代码做一点解释.
- 首先我们为该窗体添加了一个 `LoadingBackground.png` 的背景图,然后在窗体右下加添加了一行提示文本以及一个类似于 Windows11 的加载动画.但是我是持续转动,没 Windows11 那么优雅 😅
- 保持 `Loading.xaml.cs` 中的代码为默认代码即可,不需要改变.
- 接下来调整我们的`Main`函数让他能在启动的时候显示加载页面.并在启动完成后关闭该页面.

```csharp
/// <summary>
/// Main入口函数
/// </summary>
/// <param name="args"></param>
/// <returns></returns>
[STAThread]
public static void Main(string[] args)
{
    WinApis.EnsureSingleInstance(out var createdNew);
    if (!createdNew)
    {
        WinApis.BringExistingInstanceToFront();
        // 退出当前实例
        return;
    }
    var loading = new Loading();
    loading.Show();
    // ...其他代码
    loading.Close();
}
```

- 上面的代码倒是能够正常显示加载界面,并在程序加载完成后自动关闭.但是我们都知道,若是没有意外的时候就会出现意外了.会发现我们写的动画,根本没用转动.所以就产生了下一个问题来修复这个问题.

#### 修复 Loading 页面动画未生效

> 上面我们的动画无效果的根本原因就是我们所有的操作都是在主线程进行的,并且启动 IHost 主机这种耗时的操作也是,导致 UI 线程被阻塞,所以我们要才用异步的方式来启动 IHost 主机,并确保主线程正常处理 UI 线程的渲染.接下来调整代码:

- 由于 `CreateHostBuilder`函数在上一篇文章中已经有所展示,这里就省略掉了.
- 这里我们需要使用 `Dispatcher` 和异步操作,可以确保 UI 线程不会被阻塞,从而使动画能够正常播放.
- `Dispatcher` 是 WPF 中的一个类,用于管理线程的工作队列.它确保 UI 线程上的操作是线程安全的.

```csharp
/// <summary>
/// 异步初始化主机
/// </summary>
/// <param name="args"></param>
/// <returns></returns>
private static Task<IHost> InitializeHostAsync(string[] args)
{
    return Task.Factory.StartNew(() =>
    {
        // 创建一个通用主机
        var host = CreateHostBuilder(args).Build();
        host.InitializeApplication();
        return host;
    }, CancellationToken.None, TaskCreationOptions.DenyChildAttach, TaskScheduler.Default);
}

/// <summary>
/// 处理主机初始化的结果
/// </summary>
/// <param name="task"></param>
/// <param name="loading"></param>
/// <returns></returns>
private static async Task HandleHostInitializationAsync(Task<IHost> task, Loading loading)
{
    if (task.Exception is not null)
    {
        // 处理异常
        MessageBox.Show($"应用程序启动时发生错误: {task.Exception.InnerException?.Message}", "错误", MessageBoxButton.OK, MessageBoxImage.Error);
        loading.Close();
        return;
    }
    var host = await task.ConfigureAwait(false);
    // 配置主窗口
    var app = new App()
    {
        MainWindow = host.Services.GetRequiredService<MainWindow>()
    };
    app.AppHost = host;
    app.MainWindow.Visibility = Visibility.Visible;
    // 关闭加载窗口
    loading.Close();
    app.Run();
}

/// <summary>
/// 显示加载窗体
/// </summary>
/// <param name="loading"></param>
private static void ShowLoadingScreen(out Loading loading)
{
    loading = new();
    loading.Show();
}
```

- 最后我们还需要调整 `Main` 函数,这是我们最终完整的 Main

```csharp
[STAThread]
public static void Main(string[] args)
{
    WinApis.EnsureSingleInstance(out var createdNew);
    if (!createdNew)
    {
        WinApis.BringExistingInstanceToFront();
        // 退出当前实例
        return;
    }
    ShowLoadingScreen(out var loading);
    // 获取当前线程的Dispatcher对象
    var dispatcher = Dispatcher.CurrentDispatcher;
    InitializeHostAsync(args).ContinueWith(async t =>
            await dispatcher.InvokeAsync(async () =>
                await HandleHostInitializationAsync(t, loading).ConfigureAwait(false)),
        TaskScheduler.Default);
    // 启动WPF消息循环
    Dispatcher.Run();
}
```

- 经过上述调整后 Loading 的动画即可正常显示了.

#### 使用新的 Program.cs 类来启动程序而不是 App.xaml.cs

> WPF 程序默认从 `App.xaml.cs` 中启动,前面我们为其添加了 Main 函数并调整了程序的默认启动方式.让他支持从我们自定义的 Main 函数启动,但是我们目前为止所有代码均写在 App.xaml.cs 中的,这导致我们部分和程序初始化无关的代码也在这里面,并且后期可能会有更多的内容写进了,比如全局资源的一些事件,比如我的项目中就对 `ComboBox` 进行了 UI 上的调整,导致产生了一些事件需要写在 App 类中,让我感觉代码有点丑.所以有了这个需求.接下来就是改造时刻.

- 新建 Program.cs 文件.将 Main 函数,确保程序单例的代码迁移过去.让 App.xaml.cs 更纯粹.
- 首先调整 App 类的构造函数,让部分参数在实例化的时候传入.

```csharp
public App(ref IHost host)
{
    InitializeComponent();
    _host = host;
    _logger = Services.GetRequiredService<ILogger<App>>();
    MainWindow = Services.GetRequiredService<MainWindow>();
    MainWindow.Visibility = Visibility.Visible;
}
private IHost _host { get; }
private ILogger<App> _logger { get; }
```

- HandleHostInitializationAsync 函数的代码

```csharp
/// <summary>
/// 处理主机初始化的结果
/// </summary>
/// <param name="task"></param>
/// <param name="loading"></param>
/// <returns></returns>
private static async Task HandleHostInitializationAsync(Task<IHost> task, Loading loading)
{
    if (task.Exception is not null)
    {
        // 处理异常
        MessageBox.Show($"应用程序启动时发生错误: {task.Exception.InnerException?.Message}", "错误", MessageBoxButton.OK, MessageBoxImage.Error);
        loading.Close();
        return;
    }
    var host = await task.ConfigureAwait(false);
    // 配置主窗口
    var app = new App(ref host, ref WinApis._mutex);
    // 关闭加载窗口
    loading.Close();
    app.Run();
}
```

- 别的代码几乎不用动,仅需将 Main 和其相关的内容移动到 `Program.cs` 中即可,这里就不展示代码了,不然内容太长了,稍后我会将代码同步到 GitHub 仓库.感兴趣的可以前往查看.

- 最后附上 GitHub 地址: https://github.com/joesdu/WpfAutoDISample
