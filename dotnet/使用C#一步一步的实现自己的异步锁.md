# 使用 C# 一步一步的实现自己的异步锁

> 在多线程编程中,锁是确保线程安全的关键工具,用于防止多个线程同时访问共享资源导致的竞态条件.在 C# 中,`lock` 语句是同步编程中的常见选择,但当我们转向异步编程(`async/await`)时,传统的同步锁可能会引发问题,例如线程阻塞或死锁.特别是在高并发场景,如 ASP.NET Core 应用或实时数据处理系统,异步锁成为不可或缺的工具.

> 本文将从基础概念开始,逐步引导您实现一个高效的异步锁 `AsyncLock`,并深入分析其设计理念、实现细节和应用场景.作为一名精通 .NET 平台的 C# 工程师,我将结合丰富的编程经验,带您从浅到深理解异步锁的实现过程,并展示如何在实际项目中应用它.

## 第一步:理解锁的基础知识

锁(Lock)是一种同步机制,用于控制对共享资源的访问,确保在任意时刻只有一个线程或任务可以操作该资源.在 C# 中,`lock` 语句基于 `Monitor` 类实现,提供了一种简单的方式来实现互斥访问.以下是一个同步代码的示例:

```csharp
private readonly Lock _syncLock = new();
private int _counter = 0;

public void Increment()
{
    lock (_syncLock)
    {
        _counter++;
    }
}
```

在这个例子中,`lock` 确保 `_counter` 的更新是线程安全的,防止多个线程同时修改导致数据不一致.然而,这种同步锁在异步编程中存在局限性.

## 第二步:同步锁在异步代码中的问题

在异步编程中,`async/await` 模式允许任务在等待 I/O 操作(如网络请求或文件读写)时释放线程,从而提高系统效率.然而,如果在异步方法中使用 `lock`,可能会导致问题.考虑以下错误示例:

```csharp
private readonly Lock _syncLock = new();
private int _counter = 0;

public async Task IncrementAsync()
{
    lock (_syncLock)
    {
        await Task.Delay(100); // 模拟异步操作
        _counter++;
    }
}
```

在这个例子中,`lock` 会阻塞当前线程,而 `await` 试图在同一线程上继续执行,可能导致死锁或性能下降.例如,在 ASP.NET Core 中,线程阻塞可能耗尽线程池资源,降低服务器的响应能力.为了解决这个问题,我们需要一种专门为异步代码设计的锁——异步锁.

## 第三步:认识 SemaphoreSlim

`SemaphoreSlim` 是 .NET 提供的一个轻量级同步原语,专为单进程内的同步设计.它支持异步等待方法 `WaitAsync`,非常适合异步编程环境 [SemaphoreSlim Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim).通过将 `SemaphoreSlim` 的初始计数和最大计数设置为 1,我们可以将其用作互斥锁(mutex),确保一次只有一个任务可以进入临界区.

`SemaphoreSlim` 的关键方法包括:

- `WaitAsync`:异步等待信号量,获取锁.
- `Release`:释放信号量,允许其他任务获取锁.
- `CurrentCount`:获取当前可用的信号量计数.

通过利用 `SemaphoreSlim`,我们可以构建一个高效的异步锁.

## 第四步:设计异步锁

在实现 `AsyncLock` 之前,我们需要明确其设计目标:

- **非阻塞**:使用 `WaitAsync` 实现异步等待,不阻塞线程.
- **互斥**:一次只允许一个任务持有锁.
- **易用性**:提供类似 `lock` 的 API,配合 `using` 语句自动释放锁.
- **不可重入**:为简化实现和避免死锁,不支持同一任务多次获取锁.
- **性能优化**:通过缓存任务减少资源分配.
- **资源管理**:支持 `IDisposable` 接口,确保正确释放资源.

## 第五步:逐步实现 AsyncLock

让我们一步步构建 `AsyncLock` 类,逐步添加功能并解释每个部分的实现原理.

### 步骤 1:定义类和字段

首先,创建一个 `AsyncLock` 类,继承 `IDisposable` 接口,并添加一个 `SemaphoreSlim` 字段,初始计数和最大计数均为 1,以实现互斥锁.

```csharp
public sealed class AsyncLock : IDisposable
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);
}
```

`SemaphoreSlim(1, 1)` 确保一次只有一个任务可以获取锁,类似于一个二进制信号量.

### 步骤 2:实现基本的 LockAsync 方法

`LockAsync` 方法是获取锁的核心方法,它应该异步等待锁并返回一个可释放的对象.我们使用 `SemaphoreSlim.WaitAsync` 来实现异步等待,并返回一个 `IDisposable` 对象,用于在操作完成后释放锁.

```csharp
public async Task<IDisposable> LockAsync()
{
    await _semaphore.WaitAsync();
    return new Release(this);
}
```

这里,`Release` 是一个内部类,用于释放锁.我们稍后会定义它.

### 步骤 3:实现 Release 结构

为了提高性能,我们将 `Release` 实现为一个只读结构(`struct`),而不是类,因为它仅持有对 `AsyncLock` 的引用,且生命周期短暂.`Release` 实现 `IDisposable` 接口,在 `Dispose` 方法中释放信号量.

```csharp
private readonly struct Release : IDisposable
{
    private readonly AsyncLock _asyncLock;

    internal Release(AsyncLock asyncLock)
    {
        _asyncLock = asyncLock;
    }

    public void Dispose()
    {
        _asyncLock._semaphore.Release();
    }
}
```

通过将 `Release` 设计为结构,我们减少了内存分配的开销.`using` 语句可以确保在离开作用域时自动调用 `Dispose`,释放锁.

### 步骤 4:优化 LockAsync 方法

在某些情况下,锁可能立即可用(即 `WaitAsync` 同步完成).为了避免每次都创建新的 `Task`,我们可以缓存一个已完成的 `Task<Release>`,用于同步获取锁的场景.

```csharp
private readonly Task<Release> _cachedReleaseTask;

public AsyncLock()
{
    _cachedReleaseTask = Task.FromResult(new Release(this));
}

public Task<Release> LockAsync()
{
    var waitTask = _semaphore.WaitAsync();
    return waitTask.IsCompletedSuccessfully
        ? _cachedReleaseTask
        : LockAsyncInternal(waitTask);
}

private async Task<Release> LockAsyncInternal(Task waitTask)
{
    await waitTask.ConfigureAwait(false);
    return new Release(this);
}
```

`Task.FromResult` 创建一个已完成的 `Task`,减少了同步场景下的任务分配开销.`ConfigureAwait(false)` 确保在异步等待后不强制恢复到原始上下文,进一步优化性能.

### 步骤 5:添加取消支持

为了支持取消操作,我们在 `LockAsync` 方法中添加 `CancellationToken` 参数,允许任务在等待锁时被取消.

```csharp
public Task<Release> LockAsync(CancellationToken cancellationToken = default)
{
    var waitTask = _semaphore.WaitAsync(cancellationToken);
    return waitTask.IsCompletedSuccessfully
        ? _cachedReleaseTask
        : LockAsyncInternal(waitTask);
}
```

如果 `cancellationToken` 被取消,`WaitAsync` 会抛出 `OperationCanceledException`,符合异步编程的预期行为.

### 步骤 6:实现 Dispose 方法

`AsyncLock` 需要实现 `IDisposable` 接口,以确保正确释放 `SemaphoreSlim` 资源.我们添加一个 `_disposed` 标志,防止重复处置.

```csharp
private bool _disposed;

public void Dispose()
{
    if (!_disposed)
    {
        _semaphore.Dispose();
        _disposed = true;
    }
}
```

为了防止在已处置的对象上执行操作,我们在 `LockAsync` 和其他方法中检查 `_disposed` 状态.

```csharp
public Task<Release> LockAsync(CancellationToken cancellationToken = default)
{
    ObjectDisposedException.ThrowIf(_disposed, this);
    var waitTask = _semaphore.WaitAsync(cancellationToken);
    return waitTask.IsCompletedSuccessfully
        ? _cachedReleaseTask
        : LockAsyncInternal(waitTask);
}
```

### 步骤 7:添加状态属性

为了方便调试和监控,我们添加两个属性:`IsHeld` 和 `WaitingCount`.

- **IsHeld**:检查锁是否被持有,通过 `SemaphoreSlim.CurrentCount is 0` 判断.
- **WaitingCount**:提供等待任务数量的近似值.由于 `SemaphoreSlim` 不直接暴露等待队列大小,我们简化为返回 1(锁被持有)或 0(锁空闲).

```csharp
public bool IsHeld
{
    get
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return _semaphore.CurrentCount is 0;
    }
}

public int WaitingCount
{
    get
    {
        if (_disposed)
        {
            return 0;
        }
        return _semaphore.CurrentCount is 0 ? 1 : 0;
    }
}
```

### 步骤 8:实现辅助方法

为了提高易用性,我们添加两个辅助方法,允许在获取锁后直接执行异步操作,并自动释放锁.

```csharp
public async Task LockAsync(Func<Task> action, CancellationToken cancellationToken = default)
{
    ObjectDisposedException.ThrowIf(_disposed, this);
    ArgumentNullException.ThrowIfNull(action);
    await _semaphore.WaitAsync(cancellationToken).ConfigureAwait(false);
    try
    {
        await action().ConfigureAwait(false);
    }
    finally
    {
        _semaphore.Release();
    }
}

public async Task<TResult> LockAsync<TResult>(Func<Task<TResult>> func, CancellationToken cancellationToken = default)
{
    ObjectDisposedException.ThrowIf(_disposed, this);
    ArgumentNullException.ThrowIfNull(func);
    await _semaphore.WaitAsync(cancellationToken).ConfigureAwait(false);
    try
    {
        return await func().ConfigureAwait(false);
    }
    finally
    {
        _semaphore.Release();
    }
}
```

这些方法简化了常见的使用模式,避免了手动管理 `Release` 对象的需要.

### 步骤 9:完善 Release 结构

为了确保在 `AsyncLock` 已处置的情况下不会尝试释放锁,我们在 `Release` 结构的 `Dispose` 方法中添加检查.

```csharp
public readonly struct Release : IDisposable
{
    private readonly AsyncLock? _asyncLockToRelease;

    internal Release(AsyncLock? asyncLockToRelease)
    {
        _asyncLockToRelease = asyncLockToRelease;
    }

    public void Dispose()
    {
        if (_asyncLockToRelease is not null && !_asyncLockToRelease._disposed)
        {
            _asyncLockToRelease._semaphore.Release();
        }
    }
}
```

## 第六步:优化与设计选择

在实现 `AsyncLock` 的过程中,我们做出了一些关键的设计选择:

1. **不可重入锁**:选择不可重入设计以简化实现并降低死锁风险.相比支持重入的锁(如某些库中的实现),不可重入锁避免了跟踪任务标识和重入计数的复杂性.开发者可以通过重构代码避免递归获取锁.
2. **缓存任务**:通过 `_cachedReleaseTask` 优化同步获取锁的场景,减少任务分配的开销.
3. **异步等待**:使用 `ConfigureAwait(false)` 提高性能,避免不必要的上下文切换.
4. **取消支持**:通过 `CancellationToken` 支持操作取消,适合需要快速响应的场景.
5. **资源管理**:通过 `IDisposable` 和 `_disposed` 标志确保资源正确释放.

## 第七步:应用场景与使用示例

`AsyncLock` 在以下场景中尤为有用:

| 场景             | 描述                                                                   |
| ---------------- | ---------------------------------------------------------------------- |
| **Web 应用程序** | 在 ASP.NET Core 中,保护共享资源(如内存缓存)免受并发请求的竞争条件影响. |
| **I/O 操作**     | 确保对非线程安全的文件或网络资源的异步操作独占访问.                    |
| **内存缓存**     | 保护 Redis 或本地字典等缓存,防止数据不一致.                            |

以下是一个更复杂的示例,展示如何在 ASP.NET Core 中使用 `AsyncLock` 保护内存缓存:

```csharp
public class CacheService
{
    private readonly AsyncLock _lock = new();
    private readonly Dictionary<string, string> _cache = new();

    public async Task UpdateCacheAsync(string key, string value)
    {
        using (await _lock.LockAsync())
        {
            _cache[key] = value;
            await Task.Delay(50); // 模拟异步操作
        }
    }

    public async Task<string> GetCacheAsync(string key)
    {
        using (await _lock.LockAsync())
        {
            return _cache.TryGetValue(key, out var value) ? value : null;
        }
    }
}
```

这个示例展示了如何在高并发环境中保护内存缓存,确保数据一致性.

## 第八步:注意事项与最佳实践

在使用 `AsyncLock` 时,需要注意以下几点:

- **避免递归获取锁**:由于不可重入,同一任务多次调用 `LockAsync` 会导致死锁.应重构代码以避免这种情况.
- **最小化锁持有时间**:尽量缩短临界区代码的执行时间,以减少锁争用.
- **正确处置**:确保 `AsyncLock` 在适当的时机被处置,避免在任务持有或等待锁时过早释放.
- **使用 using 语句**:始终使用 `using` 语句确保 `Release` 对象被正确处置.

最佳实践包括:

- 使用 `CancellationToken` 支持操作取消.
- 监控 `IsHeld` 和 `WaitingCount` 属性,优化高争用场景.
- 在临界区内避免执行耗时操作.

## 结论

通过以上步骤,我们成功实现了一个高效、易用的异步锁 `AsyncLock`.它利用 `SemaphoreSlim` 提供了非阻塞的互斥访问,适用于现代 .NET 应用程序中的异步编程场景.无论是保护 Web 应用中的共享资源,还是确保 I/O 操作的线程安全,`AsyncLock` 都是一个强大的工具.希望这篇文章能帮助您深入理解异步锁的实现原理,并在实际项目中自信地应用它.

## 其他

- **参考资料**:
  - [SemaphoreSlim Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim)
  - [C# Asynchronous Programming](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)
  - [Monitor Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.monitor)

本文完整代码已经包含在 GitHub EasilyNET 仓库中,您可以在 [AsyncLock.cs](https://github.com/joesdu/EasilyNET/blob/main/src/EasilyNET.Core/Threading/AsyncLock.cs) 找到它.同时本文也同步更新到 GitHub 仓库中,便于脱离公众号查看,毕竟公众号不支持 markdown 格式的文章,显示不太友好.
