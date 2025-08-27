# 深入理解 C# 中 async 和 await 的工作原理

> 在现代 .NET 应用程序中，异步编程是提升性能和可扩展性的核心技术。async 和 await 关键字作为 C# 异步编程的基石，自 C# 5.0 引入以来，已彻底改变了开发者编写并发代码的方式。本文基于微软官方博客文章《How Async/Await Really Works in C#》，深入探讨 async/await 的历史演进、设计理念、关键技术、实现原理及使用方式，旨在帮助开发者全面理解异步编程的底层机制并在实际项目中灵活应用。
>
> 本文将从异步编程的历史开始，逐步深入到编译器的内部实现，揭示 async/await 如何通过状态机、上下文流动和调度器集成来简化异步编程。同时，我们将通过大量实际例子来加深理解，并探讨性能优化、最佳实践以及常见陷阱。

## 第一步：异步编程的历史演进与挑战

异步编程并非新鲜事物，其发展历程反映了 .NET 平台对并发和性能的不断追求。理解 async/await 的工作原理，首先需要回顾其前身模式，这些模式虽然功能强大，但也暴露出诸多局限性。让我们从最原始的异步编程模式开始，逐步演进到现代的 async/await。

### 1. **APM（Asynchronous Programming Model）模式**

#### 原理详解

APM 模式，又称 Begin/End 模式，是 .NET Framework 1.0 引入的异步编程标准。对于任何同步操作 `DoStuff`，APM 要求实现者提供一对 BeginDoStuff 和 EndDoStuff 方法：

```csharp
class Handler
{
    public int DoStuff(string arg);

    public IAsyncResult BeginDoStuff(string arg, AsyncCallback? callback, object? state);
    public int EndDoStuff(IAsyncResult asyncResult);
}
```

- **BeginDoStuff**：启动异步操作，接受参数、回调委托和状态对象。返回 IAsyncResult 接口，用于跟踪操作状态。
- **EndDoStuff**：完成异步操作，获取结果并处理异常。
- **IAsyncResult**：核心接口，包含 AsyncState（状态对象）、AsyncWaitHandle（等待句柄）、IsCompleted（完成状态）和 CompletedSynchronously（同步完成标志）。

#### 优势与挑战

- **优势**：

  - 支持真正的异步执行，避免阻塞线程池。
  - 提供细粒度的控制，可以检查操作状态和等待完成。
  - 适用于需要长时间运行的操作，如网络 I/O 或文件操作。

- **挑战**：
  - **回调地狱（Callback Hell）**：多个异步操作组合时，代码嵌套层级深，难以阅读和维护。
  - **栈溢出风险（Stack Dives）**：如果操作同步完成，回调可能在同一栈帧中执行，导致递归调用栈过深。
  - **异常处理复杂**：异常可能在 Begin 或 End 方法中抛出，需要分别处理。
  - **组合困难**：难以实现复杂的异步控制流，如顺序执行、并行执行或条件分支。

#### 详细使用方式与例子

```csharp
// 同步版本
public int DoStuff(string arg)
{
    // 模拟耗时操作
    Thread.Sleep(1000);
    return arg.Length;
}

// APM 版本
public IAsyncResult BeginDoStuff(string arg, AsyncCallback? callback, object? state)
{
    // 启动异步操作
    return Task.Run(() =>
    {
        int result = DoStuff(arg);
        callback?.Invoke(new MyAsyncResult(result, state));
    }).ContinueWith(_ => new MyAsyncResult(DoStuff(arg), state));
}

public int EndDoStuff(IAsyncResult asyncResult)
{
    var myResult = (MyAsyncResult)asyncResult;
    return myResult.Result;
}

// 使用示例
handler.BeginDoStuff("test", iar =>
{
    try
    {
        Handler handler = (Handler)iar.AsyncState!;
        int i = handler.EndDoStuff(iar);
        Console.WriteLine($"Result: {i}");
    }
    catch (Exception e)
    {
        Console.WriteLine($"Error: {e.Message}");
    }
}, handler);
```

这个例子展示了 APM 的基本用法，但想象一下如果有多个操作需要顺序执行，代码会变得多么复杂。

#### 栈溢出问题示例

```csharp
// 危险的递归调用可能导致栈溢出
void ReadNext()
{
    stream.BeginRead(buffer, 0, buffer.Length, iar =>
    {
        if (stream.EndRead(iar) > 0)
        {
            ReadNext(); // 如果每次都同步完成，会导致栈溢出
        }
    }, null);
}
```

### 2. **EAP（Event-Based Asynchronous Pattern）模式**

#### 原理详解

.NET Framework 2.0 引入的 EAP 模式使用事件驱动的方式处理异步操作，旨在简化异步编程：

```csharp
class Handler
{
    public void DoStuffAsync(string arg, object? userToken);
    public event DoStuffEventHandler? DoStuffCompleted;
}

public delegate void DoStuffEventHandler(object sender, DoStuffEventArgs e);

public class DoStuffEventArgs : AsyncCompletedEventArgs
{
    public DoStuffEventArgs(int result, Exception? error, bool canceled, object? userToken)
        : base(error, canceled, userToken) => Result = result;

    public int Result { get; }
}
```

- **DoStuffAsync**：启动异步操作，接受参数和用户令牌。
- **DoStuffCompleted**：事件在操作完成时触发。
- **AsyncCompletedEventArgs**：提供结果、异常和取消状态。

#### 优势与挑战

- **优势**：

  - 引入 SynchronizationContext，支持 UI 线程调度。
  - 事件模型更直观，易于理解。
  - 支持取消操作。

- **挑战**：
  - 事件模型复杂，难以组合多个异步操作。
  - 内存泄漏风险：忘记注销事件处理器。
  - 难以实现复杂的控制流。

#### 详细使用方式与例子

```csharp
// EAP 实现示例
class FileHandler
{
    public void ReadFileAsync(string path, object? userToken)
    {
        Task.Run(() =>
        {
            try
            {
                string content = File.ReadAllText(path);
                OnReadFileCompleted(new ReadFileEventArgs(content, null, false, userToken));
            }
            catch (Exception ex)
            {
                OnReadFileCompleted(new ReadFileEventArgs(null, ex, false, userToken));
            }
        });
    }

    public event ReadFileEventHandler? ReadFileCompleted;

    protected virtual void OnReadFileCompleted(ReadFileEventArgs e)
    {
        ReadFileCompleted?.Invoke(this, e);
    }
}

// 使用示例
var handler = new FileHandler();
handler.ReadFileCompleted += (sender, e) =>
{
    if (e.Error != null)
    {
        Console.WriteLine($"Error: {e.Error.Message}");
    }
    else
    {
        Console.WriteLine($"Content: {e.Content}");
    }
};
handler.ReadFileAsync("example.txt", null);
```

### 3. **Tasks 的诞生与统一**

.NET Framework 4.0 引入的 Task 类型标志着异步编程的重大进步。Task 作为异步操作的统一表示，提供了延续（Continuation）的概念，使得异步编程更加直观和可组合。

#### Task 的核心特性

- **统一表示**：所有异步操作都可以用 Task 或 Task<TResult> 表示。
- **延续支持**：通过 ContinueWith 方法支持操作完成后执行后续逻辑。
- **组合操作**：Task.WhenAll、Task.WhenAny 等方法支持复杂的异步组合。
- **异常处理**：Task 可以捕获和传播异常。
- **取消支持**：通过 CancellationToken 支持取消操作。

#### 例子：Task 的基本使用

```csharp
// 创建 Task
Task task = Task.Run(() => DoWork());

// 延续
task.ContinueWith(t =>
{
    if (t.IsFaulted)
    {
        Console.WriteLine($"Error: {t.Exception.Message}");
    }
    else
    {
        Console.WriteLine("Work completed");
    }
});

// 组合
Task<Task<string>>[] tasks = urls.Select(url => Task.Run(() => DownloadString(url))).ToArray();
Task.WhenAll(tasks).ContinueWith(t =>
{
    foreach (var task in t.Result)
    {
        Console.WriteLine(task.Result);
    }
});
```

Task 的引入解决了 APM 和 EAP 的许多问题，但延续的回调风格仍然不够直观，这就是 async/await 诞生的背景。

## 第二步：迭代器如何启发 async/await

有趣的是，async/await 的设计灵感来自于 C# 2.0 的迭代器。迭代器允许将一个方法转换为状态机，实现协程（Coroutine）。这种模式为异步编程提供了一种全新的思路。

### 迭代器的基本原理

考虑以下迭代器方法：

```csharp
public static IEnumerable<int> Fib()
{
    int prev = 0, next = 1;
    yield return prev;
    yield return next;

    while (true)
    {
        int sum = prev + next;
        yield return sum;
        prev = next;
        next = sum;
    }
}
```

编译器将其转换为状态机，每次调用 MoveNext() 时恢复执行状态：

```csharp
// 编译器生成的代码（简化版）
private sealed class <Fib>d__0 : IEnumerable<int>, IEnumerator<int>
{
    private int <>1__state;
    private int <>2__current;
    private int <prev>5__1;
    private int <next>5__2;

    public <Fib>d__0(int <>1__state)
    {
        this.<>1__state = <>1__state;
    }

    public bool MoveNext()
    {
        switch (<>1__state)
        {
            case 0:
                <>1__state = -1;
                <prev>5__1 = 0;
                <next>5__2 = 1;
                <>2__current = <prev>5__1;
                <>1__state = 1;
                return true;
            case 1:
                <>1__state = -1;
                <>2__current = <next>5__2;
                <>1__state = 2;
                return true;
            case 2:
                <>1__state = -1;
                <>2__current = <prev>5__1 + <next>5__2;
                <prev>5__1 = <next>5__2;
                <next>5__2 = <>2__current;
                <>1__state = 2;
                return true;
            default:
                return false;
        }
    }

    public int Current => <>2__current;
    object IEnumerator.Current => Current;
    public void Dispose() { }
    public void Reset() { throw new NotSupportedException(); }
    public IEnumerator<int> GetEnumerator() => this;
}
```

### 与 async/await 的关联

async/await 本质上是迭代器的异步版本：

- **yield return** 对应 **await**：暂停执行并返回控制。
- **MoveNext()** 对应 **状态机恢复**：从暂停点继续执行。
- **状态机**：跟踪局部变量和执行位置。

这种设计允许 async 方法在 await 点暂停，并在异步操作完成时恢复，而无需显式回调。

#### 详细例子：异步迭代器

```csharp
// 异步版本的斐波那契数列生成器
public static async IAsyncEnumerable<int> FibAsync()
{
    int prev = 0, next = 1;
    yield return prev;
    yield return next;

    while (true)
    {
        await Task.Delay(100); // 模拟异步操作
        int sum = prev + next;
        yield return sum;
        prev = next;
        next = sum;
    }
}

// 使用异步迭代器
await foreach (int number in FibAsync())
{
    Console.WriteLine(number);
    if (number > 100) break;
}
```

这个例子展示了迭代器如何自然地扩展到异步领域。

## 第三步：async/await 的编译器转换

async/await 的魔力在于编译器的智能转换。编译器将开发者编写的直观代码转换为复杂的状态机实现。这个过程涉及多个关键组件。

### 1. **状态机生成详解**

#### 原理

编译器将 async 方法重写为状态机结构体，实现 IAsyncStateMachine 接口。状态机跟踪当前执行位置和局部变量。

#### 实现细节

```csharp
[AsyncStateMachine(typeof(<CopyStreamToStreamAsync>d__0))]
public Task CopyStreamToStreamAsync(Stream source, Stream destination)
{
    <CopyStreamToStreamAsync>d__0 stateMachine = default;
    stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
    stateMachine.source = source;
    stateMachine.destination = destination;
    stateMachine.<>1__state = -1;
    stateMachine.<>t__builder.Start(ref stateMachine);
    return stateMachine.<>t__builder.Task;
}
```

#### 状态机结构

```csharp
private struct <CopyStreamToStreamAsync>d__0 : IAsyncStateMachine
{
    public int <>1__state;                    // 当前状态
    public AsyncTaskMethodBuilder <>t__builder; // 构建器
    public Stream source;                     // 参数
    public Stream destination;                // 参数
    private byte[] <buffer>5__1;             // 局部变量
    private TaskAwaiter <>u__1;              // awaiter
    private TaskAwaiter<int> <>u__2;         // awaiter

    public void MoveNext()
    {
        // 状态机逻辑
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine)
    {
        <>t__builder.SetStateMachine(stateMachine);
    }
}
```

### 2. **ExecutionContext 的流动**

#### 原理详解

ExecutionContext 负责在异步操作间流动环境数据（如 AsyncLocal<T>）。编译器确保上下文在 await 点正确捕获和恢复。

#### 工作流程

1. **捕获**：在 await 之前捕获当前 ExecutionContext。
2. **存储**：将上下文与异步操作关联。
3. **恢复**：在异步操作完成时恢复上下文。

#### 例子：AsyncLocal 的使用

```csharp
private static AsyncLocal<string> _userContext = new AsyncLocal<string>();

public async Task ProcessRequestAsync()
{
    _userContext.Value = "User123";
    await DoSomeWorkAsync();
    // _userContext.Value 仍然是 "User123"
}

public async Task DoSomeWorkAsync()
{
    Console.WriteLine($"User: {_userContext.Value}"); // 输出 "User123"
    await Task.Delay(100);
    Console.WriteLine($"User: {_userContext.Value}"); // 仍然输出 "User123"
}
```

#### 优势

- **线程安全**：环境数据在异步执行中保持一致。
- **性能优化**：.NET Core 中 ExecutionContext 变为不可变，减少复制开销。

### 3. **SynchronizationContext 的集成**

#### 原理详解

SynchronizationContext 抽象了调度器，支持 UI 线程等特定环境的调度。await 默认捕获当前 SynchronizationContext 并在其中恢复执行。

#### 工作流程

1. **捕获**：await 时捕获当前 SynchronizationContext。
2. **调度**：异步操作完成后，在捕获的上下文中恢复执行。

#### 例子：UI 线程调度

```csharp
// WPF 应用中
private async void button1_Click(object sender, RoutedEventArgs e)
{
    button1.IsEnabled = false;
    var result = await Task.Run(() => ComputeMessage());
    button1.Content = result;  // 在 UI 线程执行
    button1.IsEnabled = true;
}
```

#### 自定义 SynchronizationContext 例子

```csharp
public class CustomSynchronizationContext : SynchronizationContext
{
    private readonly BlockingCollection<(SendOrPostCallback, object?)> _queue = new();
    private readonly Thread _thread;

    public CustomSynchronizationContext()
    {
        _thread = new Thread(Run);
        _thread.Start();
    }

    public override void Post(SendOrPostCallback d, object? state)
    {
        _queue.Add((d, state));
    }

    private void Run()
    {
        while (true)
        {
            var (callback, state) = _queue.Take();
            callback(state);
        }
    }
}
```

### 4. **状态机字段详解**

状态机结构体包含多个关键字段：

- **<>1\_\_state**：当前状态，指示执行位置。

  - -1：初始状态
  - 0, 1, 2...：await 点索引
  - -2：完成状态

- **<>t\_\_builder**：构建器，负责创建和完成 Task。

  - AsyncTaskMethodBuilder：用于 Task
  - AsyncTaskMethodBuilder<TResult>：用于 Task<TResult>
  - AsyncVoidMethodBuilder：用于 async void

- **局部变量提升**：如 buffer 等变量在 await 点间保持状态。

- **Awaiter 字段**：存储 await 表达式的结果。

#### 例子：复杂状态机

```csharp
public async Task<int> ComplexAsync()
{
    int x = 1;
    await Task.Delay(100);
    int y = 2;
    await Task.Delay(100);
    return x + y;
}

// 生成的状态机字段
private struct <ComplexAsync>d__0 : IAsyncStateMachine
{
    public int <>1__state;
    public AsyncTaskMethodBuilder<int> <>t__builder;
    private int <x>5__1;      // 提升的局部变量
    private int <y>5__2;      // 提升的局部变量
    private TaskAwaiter <>u__1;
    private TaskAwaiter <>u__2;
    // ...
}
```

## 第四步：性能优化与最佳实践

async/await 的性能不断优化，从 .NET Framework 到 .NET Core 发生了显著改进。

### 1. **.NET Framework vs .NET Core 对比**

| 特性                 | .NET Framework                        | .NET Core                           |
| -------------------- | ------------------------------------- | ----------------------------------- |
| **ExecutionContext** | 可变，频繁复制，每次 await 分配新对象 | 不可变，捕获开销低，池化优化        |
| **状态机**           | 每次 await 分配新对象                 | 池化 AsyncStateMachineBox，减少分配 |
| **性能**             | 高分配压力，GC 负担重                 | 近零分配（优化场景），更好的扩展性  |

### 2. **池化机制详解**

.NET Core 引入池化 AsyncStateMachineBox，避免重复分配：

```csharp
private class AsyncStateMachineBox<TStateMachine> : Task<VoidTaskResult>, IAsyncStateMachineBox
{
    public TStateMachine? StateMachine;
    public ExecutionContext? Context;
    private Action? _moveNextAction;

    // 池化实现
    private static readonly ConcurrentBag<AsyncStateMachineBox<TStateMachine>> _pool = new();

    public static AsyncStateMachineBox<TStateMachine> RentFromCache()
    {
        if (_pool.TryTake(out var box))
        {
            return box;
        }
        return new AsyncStateMachineBox<TStateMachine>();
    }

    public void ReturnToCache()
    {
        StateMachine = default;
        Context = null;
        _moveNextAction = null;
        _pool.Add(this);
    }
}
```

#### 池化效果

- **减少分配**：重用状态机对象
- **降低 GC 压力**：减少临时对象的创建
- **提高性能**：尤其在高并发场景下

### 3. **ValueTask 的优势与使用**

ValueTask 避免 Task 分配，适合高频同步完成场景：

```csharp
public class BufferReader
{
    private readonly byte[] _buffer = new byte[1024];
    private int _position;

    public ValueTask<int> ReadAsync()
    {
        if (_position < _buffer.Length)
        {
            // 同步完成，无分配
            return new ValueTask<int>(_buffer[_position++]);
        }
        else
        {
            // 异步路径
            return ReadFromNetworkAsync();
        }
    }

    private async Task<int> ReadFromNetworkAsync()
    {
        // 模拟网络读取
        await Task.Delay(10);
        _position = 0;
        return _buffer[_position++];
    }
}
```

#### ValueTask vs Task

- **Task**：总是分配对象，适合长时间运行的操作
- **ValueTask**：避免分配，适合快速同步完成的操作
- **注意**：ValueTask 只能被消费一次

### 4. **ConfigureAwait 的深入理解**

```csharp
public async Task<string> GetDataAsync()
{
    var data = await DownloadDataAsync().ConfigureAwait(false);
    return ProcessData(data); // 在后台线程执行
}
```

#### ConfigureAwait(false) 的作用

- **避免死锁**：在 UI 应用中防止死锁
- **提高性能**：减少线程切换
- **适用场景**：库代码，纯计算任务

#### 例子：死锁场景

```csharp
// 可能死锁的代码
public async Task<string> GetDataAsync()
{
    var data = await DownloadDataAsync(); // 捕获 SynchronizationContext
    return ProcessData(data); // 在 UI 线程执行，如果 UI 线程被阻塞，会死锁
}

// 修复
public async Task<string> GetDataAsync()
{
    var data = await DownloadDataAsync().ConfigureAwait(false); // 不捕获上下文
    return ProcessData(data); // 在后台线程执行
}
```

## 第五步：应用场景与使用方式

async/await 适用于各种异步场景，以下是详细的实际应用例子。

### 1. **I/O 操作**

```csharp
public async Task<string> DownloadContentAsync(string url)
{
    using var client = new HttpClient();
    var response = await client.GetAsync(url);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
}

// 使用示例
var content = await DownloadContentAsync("https://example.com");
Console.WriteLine(content);
```

### 2. **并发处理**

```csharp
public async Task<List<string>> DownloadMultipleAsync(IEnumerable<string> urls)
{
    var tasks = urls.Select(url => DownloadContentAsync(url));
    var results = await Task.WhenAll(tasks);
    return results.ToList();
}

// 并行处理
public async Task ProcessInParallelAsync()
{
    var task1 = ProcessDataAsync("data1");
    var task2 = ProcessDataAsync("data2");
    var task3 = ProcessDataAsync("data3");

    await Task.WhenAll(task1, task2, task3);
    Console.WriteLine("All processing completed");
}
```

### 3. **UI 响应性**

```csharp
private async void button1_Click(object sender, EventArgs e)
{
    button1.Enabled = false;
    button1.Text = "Processing...";

    try
    {
        var result = await ComputeAsync();
        button1.Text = result;
    }
    catch (Exception ex)
    {
        button1.Text = $"Error: {ex.Message}";
    }
    finally
    {
        button1.Enabled = true;
    }
}
```

### 4. **重试机制**

```csharp
public async Task<T> RetryAsync<T>(Func<Task<T>> operation, int maxRetries)
{
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (i < maxRetries - 1)
        {
            Console.WriteLine($"Attempt {i + 1} failed: {ex.Message}");
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i))); // 指数退避
        }
    }
    throw new Exception("All retries failed");
}

// 使用
var result = await RetryAsync(() => DownloadContentAsync("https://example.com"), 3);
```

### 5. **超时控制**

```csharp
public async Task<T> WithTimeoutAsync<T>(Task<T> task, TimeSpan timeout)
{
    var delayTask = Task.Delay(timeout);
    var completedTask = await Task.WhenAny(task, delayTask);

    if (completedTask == delayTask)
    {
        throw new TimeoutException("Operation timed out");
    }

    return await task; // 任务已完成
}

// 使用
try
{
    var result = await WithTimeoutAsync(DownloadContentAsync("https://example.com"), TimeSpan.FromSeconds(10));
    Console.WriteLine(result);
}
catch (TimeoutException ex)
{
    Console.WriteLine(ex.Message);
}
```

### 6. **取消操作**

```csharp
public async Task<string> DownloadWithCancellationAsync(string url, CancellationToken cancellationToken)
{
    using var client = new HttpClient();
    using var response = await client.GetAsync(url, cancellationToken);
    return await response.Content.ReadAsStringAsync();
}

// 使用
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    var result = await DownloadWithCancellationAsync("https://example.com", cts.Token);
    Console.WriteLine(result);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was cancelled");
}
```

## 第六步：注意事项与最佳实践

### 1. **避免 async void**

```csharp
// 错误示例
public async void ProcessData()
{
    await Task.Delay(1000);
    // 如果抛出异常，会导致应用程序崩溃
}

// 正确示例
public async Task ProcessDataAsync()
{
    await Task.Delay(1000);
}
```

### 2. **正确使用 ConfigureAwait**

```csharp
// 在库代码中使用
public async Task<string> GetDataAsync()
{
    var data = await DownloadDataAsync().ConfigureAwait(false);
    return ProcessData(data);
}
```

### 3. **最小化锁持有时间**

```csharp
// 错误示例
public async Task ProcessItemsAsync()
{
    lock (_lock)
    {
        await Task.Delay(1000); // 锁持有期间执行异步操作
    }
}

// 正确示例
public async Task ProcessItemsAsync()
{
    List<Item> items;
    lock (_lock)
    {
        items = GetItems(); // 快速获取数据
    }

    await ProcessItemsAsync(items); // 异步处理
}
```

### 4. **异常处理**

```csharp
public async Task<string> SafeDownloadAsync(string url)
{
    try
    {
        using var client = new HttpClient();
        var response = await client.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
    catch (HttpRequestException ex)
    {
        throw new DownloadException($"Failed to download {url}", ex);
    }
    catch (Exception ex)
    {
        throw new DownloadException($"Unexpected error downloading {url}", ex);
    }
}
```

### 5. **性能监控**

```csharp
public class AsyncPerformanceMonitor
{
    private readonly ConcurrentDictionary<string, long> _operationCounts = new();
    private readonly ConcurrentDictionary<string, long> _totalTimes = new();

    public async Task<T> MonitorAsync<T>(string operationName, Func<Task<T>> operation)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            var result = await operation();
            _operationCounts.AddOrUpdate(operationName, 1, (_, count) => count + 1);
            _totalTimes.AddOrUpdate(operationName, stopwatch.ElapsedMilliseconds,
                (_, time) => time + stopwatch.ElapsedMilliseconds);
            return result;
        }
        catch
        {
            _operationCounts.AddOrUpdate(operationName, 1, (_, count) => count + 1);
            _totalTimes.AddOrUpdate(operationName, stopwatch.ElapsedMilliseconds,
                (_, time) => time + stopwatch.ElapsedMilliseconds);
            throw;
        }
    }

    public void PrintStatistics()
    {
        foreach (var kvp in _operationCounts)
        {
            var averageTime = _totalTimes[kvp.Key] / (double)kvp.Value;
            Console.WriteLine($"{kvp.Key}: {kvp.Value} operations, average {averageTime}ms");
        }
    }
}
```

### 6. **避免常见的陷阱**

#### 陷阱 1：忘记 await

```csharp
// 错误
public async Task ProcessAsync()
{
    Task.Delay(1000); // 忘记 await
    Console.WriteLine("Done");
}

// 正确
public async Task ProcessAsync()
{
    await Task.Delay(1000);
    Console.WriteLine("Done");
}
```

#### 陷阱 2：过度使用 async

```csharp
// 过度使用
public async Task<int> GetLengthAsync(string text)
{
    await Task.Yield(); // 不必要的异步
    return text.Length;
}

// 优化
public int GetLength(string text)
{
    return text.Length;
}
```

#### 陷阱 3：Task.Run 滥用

```csharp
// 错误：不必要的线程切换
public async Task<string> ProcessDataAsync(string data)
{
    return await Task.Run(() => ProcessData(data)); // 浪费资源
}

// 正确：直接使用
public async Task<string> ProcessDataAsync(string data)
{
    return ProcessData(data);
}
```

## 第七步：调试与诊断

### 1. **异步堆栈跟踪**

```csharp
// 启用异步堆栈跟踪
AppContext.SetSwitch("System.Diagnostics.StackTrace.AppDomain", true);

// 或者在项目文件中设置
<PropertyGroup>
  <UseAsyncStackTrace>true</UseAsyncStackTrace>
</PropertyGroup>
```

### 2. **使用调试器**

- **并行堆栈窗口**：查看异步调用链
- **任务窗口**：监控正在运行的任务
- **线程窗口**：查看线程状态

### 3. **性能分析工具**

- **Visual Studio Performance Profiler**：分析异步操作的性能
- **dotnet-trace**：收集运行时跟踪
- **Application Insights**：监控生产环境性能

### 4. **自定义诊断**

```csharp
public class AsyncDiagnosticListener : IObserver<DiagnosticListener>
{
    public void OnNext(DiagnosticListener value)
    {
        if (value.Name == "System.Threading.Tasks.Task")
        {
            value.Subscribe(new TaskDiagnosticObserver());
        }
    }

    public void OnError(Exception error) { }
    public void OnCompleted() { }
}

public class TaskDiagnosticObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnNext(KeyValuePair<string, object?> value)
    {
        switch (value.Key)
        {
            case "Task.Start":
                Console.WriteLine($"Task started: {value.Value}");
                break;
            case "Task.Completed":
                Console.WriteLine($"Task completed: {value.Value}");
                break;
        }
    }

    public void OnError(Exception error) { }
    public void OnCompleted() { }
}
```

## 第八步：高级模式与扩展

### 1. **自定义 Awaiter**

```csharp
public class CustomAwaiter<T> : INotifyCompletion
{
    private T _result;
    private Action _continuation;
    private bool _completed;

    public bool IsCompleted => _completed;

    public T GetResult() => _result;

    public void OnCompleted(Action continuation)
    {
        _continuation = continuation;
        if (_completed)
        {
            continuation();
        }
    }

    public void Complete(T result)
    {
        _result = result;
        _completed = true;
        _continuation?.Invoke();
    }
}

public class CustomAwaitable<T>
{
    public CustomAwaiter<T> GetAwaiter() => new CustomAwaiter<T>();
}
```

### 2. **异步流（IAsyncEnumerable）**

```csharp
public async IAsyncEnumerable<int> GenerateNumbersAsync(int count)
{
    for (int i = 0; i < count; i++)
    {
        await Task.Delay(100);
        yield return i;
    }
}

// 使用
await foreach (int number in GenerateNumbersAsync(10))
{
    Console.WriteLine(number);
}
```

### 3. **Channels**

```csharp
public class ProducerConsumerExample
{
    private readonly Channel<int> _channel = Channel.CreateUnbounded<int>();

    public async Task ProduceAsync()
    {
        for (int i = 0; i < 10; i++)
        {
            await _channel.Writer.WriteAsync(i);
            await Task.Delay(100);
        }
        _channel.Writer.Complete();
    }

    public async Task ConsumeAsync()
    {
        await foreach (int item in _channel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Consumed: {item}");
        }
    }

    public async Task RunAsync()
    {
        var producer = ProduceAsync();
        var consumer = ConsumeAsync();
        await Task.WhenAll(producer, consumer);
    }
}
```

## 结论

async/await 通过编译器状态机转换、ExecutionContext 流动和 SynchronizationContext 集成，实现了简洁而强大的异步编程模型。从 APM 到 EAP，再到 Tasks 和 async/await，.NET 异步编程不断演进，性能不断优化。理解其工作原理有助于开发者编写更高效、可维护的异步代码。

在实际项目中，合理运用 async/await 不仅能提升应用性能，还能显著改善用户体验。希望本文能帮助您深入掌握这一核心技术，并在实践中灵活应用。

## 参考资料

- [How Async/Await Really Works in C#](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/)
- [Async/Await Best Practices](https://learn.microsoft.com/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [Understanding the Whys, Whats, and Whens of ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
- [C# Async Streams](https://learn.microsoft.com/dotnet/csharp/tutorials/generate-consume-asynchronous-stream)
- [Channels in .NET](https://learn.microsoft.com/dotnet/core/extensions/channels)
- [Performance Improvements in .NET 6](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-6/)
