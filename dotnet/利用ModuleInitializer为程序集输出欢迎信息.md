> 某些时候,我们仅仅是开发底层库,我们并不能控制程序启动.所以无法控制程序输出欢迎信息.但是又想在程序启动的时候输出欢迎信息该怎么办?这时候就需要使用到 ModuleInitializer 特性了.

#### ModuleInitializerAttribute

- ModuleInitializer 使库能够在加载时进行预先的一次性初始化,开销最小,并且用户无需显式调用任何内容.
- 当前 statoc 构造函数方法的一个特定痛点是运行时必须对具有静态构造函数的类型使用情况进行额外检查,以便决定是否需要运行静态构造函数.这会增加可衡量的开销.
- 使源生成器能够运行一些全局初始化逻辑,而无需用户显式调用任何内容.

上面是微软官方的文档的介绍.

> 通常我们可能使用该特性的作用是用于预加载一些东西,比如全局的一些配置初始化,或者说别的功能.但是这里我们仅希望在库被加载的时候输出一个 ASCII 图形字符.

- 先看看效果:(这里以 EasilyNET 库作为例子)

```shell
 ________                 _   __          ____  _____  ________  _________
|_   __  |               (_) [  |        |_   \|_   _||_   __  ||  _   _  |
  | |_ \_| ,--.   .--.   __   | |   _   __ |   \ | |    | |_ \_||_/ | | \_|
  |  _| _ `'_\ : ( (`\] [  |  | |  [ \ [  ]| |\ \| |    |  _| _     | |
 _| |__/ |// | |, `'.'.  | |  | |   \ '/ /_| |_\   |_  _| |__/ |   _| |_
|________|\'-;__/[\__) )[___][___][\_:  /|_____|\____||________|  |_____|
                                   \__.'

                       Welcome EasilyNET Project
Ver: 3.24.1025.100
Url: https://github.com/joesdu/EasilyNET
```

接下来就一步一步的展示源码了.

- 首先我们在 EasilyNET.Core 库中创建一个文件.我给他起名: `ModuleInitializer.cs`然后直接填入如下代码.

```csharp
internal static class ModuleInitializer
{
    private const string url = "https://github.com/joesdu/EasilyNET";

    private const string logo = """
                                 ________                 _   __          ____  _____  ________  _________
                                |_   __  |               (_) [  |        |_   \|_   _||_   __  ||  _   _  |
                                  | |_ \_| ,--.   .--.   __   | |   _   __ |   \ | |    | |_ \_||_/ | | \_|
                                  |  _| _ `'_\ : ( (`\] [  |  | |  [ \ [  ]| |\ \| |    |  _| _     | |
                                 _| |__/ |// | |, `'.'.  | |  | |   \ '/ /_| |_\   |_  _| |__/ |   _| |_
                                |________|\'-;__/[\__) )[___][___][\_:  /|_____|\____||________|  |_____|
                                                                   \__.'
                                """;

    private const string welcomeMsg = "Welcome EasilyNET Project";

    private static bool _initialized;
    private static readonly Lock _lock = new();

#pragma warning disable CA2255
    [ModuleInitializer]
#pragma warning restore CA2255
    internal static void Initialize()
    {
        // 使用锁来确保线程安全
        lock (_lock)
        {
            if (_initialized) return;
            var version = Assembly.GetExecutingAssembly().GetName().Version?.ToString() ?? "未知版本";
            // 计算图形的宽度
            var logoWidth = logo.Split('\n').Max(line => Encoding.ASCII.GetBytes(line).Length);
            // 计算欢迎消息前的空格数量
            var padding = ((logoWidth - Encoding.ASCII.GetBytes(welcomeMsg).Length) / 2) - 3;
            // 创建居中的欢迎消息
            var centeredWelcomeMsg = welcomeMsg.PadLeft(padding + welcomeMsg.Length);
            Console.WriteLine($"""
                               {logo}

                               {centeredWelcomeMsg}
                               Ver: \u001b[32m{version}\u001b[0m
                               Url: \u001b[34m{url}\u001b[0m

                               """);
            // 标记为已初始化
            _initialized = true;
        }
    }
}
```

- 上述的输出字符串中中,我们使用了 ANSI 转义序列来设置文本颜色:
  - \u001b[32m :设置文本颜色为绿色.
  - \u001b[34m :设置文本颜色为蓝色.
  - \u001b[0m :重置文本颜色.

通过上面的代码,则会在程序集被加载的时候,获取该程序集本身的版本信息,然后通过 logo 计算出图形宽度,并将欢迎提示语句显示到中央,并在结尾输出版本信息和仓库地址.并通过 ANSI 字符来控制控制台文本的颜色.

- 最后感谢大家的持续关注,由于近期工作没有太多的时间写这个.更新频率会没有那么高.
