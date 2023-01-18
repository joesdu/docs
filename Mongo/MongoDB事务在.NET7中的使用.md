## MongoDB 事务在.NET7 中的使用

- 背景
  - 在前边的文章中,我们详细的介绍了 MongoDB 的安装, CRUD, 聚合管道等操作.已经对 MongoDB 有了初步的了解.
  - 本文的话就着重于在.NET 中如何使用 MongoDB 事务做一个简要的描述.
- MongoDB 事务概述
  - MongoDB 单文档原生支持原子性,也具备事务的特性,但是我们说起事务,通常是指在多文档中的实现,因此,MongoDB 在 4.0 版本支持了多文档事务,4.0 对应于复制集的多表、多行,后续又在 4.2 版本支持了分片集的多表、多行事务操作.(该段话从互联网复制)
  - 从上面的文字中我们已经看出新版的 MongoDB 肯定是支持的非常好了,毕竟我最开始用 MongoDB 的时候已经 4.4 了,现在更是 6.x 版本了.
- 事务四大特性(ACID)
  - 原子性(Atomicity):事务必须是原子工作单元,对于其数据修改,要么全执行,要么全不执行.类似于 Redis 中我通常使用 Lua 脚本来实现多条命令操作的原子性.
  - 一致性(Consistency):事务在完成时,必须使所有的数据都保持一致状态.
  - 隔离性(Isolation):由并发事务所做的修改必须与任何其他并发事务所作的修改隔离(简而言之:一个事务执行过程中不应受其它事务影响).
  - 持久性(Durability):事务完成之后,对于系统的影响是永久性的.
  - 上述几句话也是从互联网复制.
- MongoDB 事务 API 的注意点.
  - 推荐.使用针对 MongoDB 部署版本更新的 MongoDB 驱动程序.对于 MongoDB 4.2 部署(副本集和分片集群上的事务,客户端必须使用为 MongoDB 4.2 更新的 MongoDB 驱动程序.
  - 使用驱动程序时,事务中的每个操作必须与会话相关联(即将会话传递给每个操作).
  - 事务中的操作使用事务级读关注,事务级写关注和事务级读偏好.
  - 在 MongoDB 4.2 及更早版本中,你无法在事务中创建集合.如果在事务内部运行会导致文档插入的写操作(例如 insert 或带有 upsert: true 的更新操作),必须在已存在的集合上才能执行.
  - 从 MongoDB 4.4 开始,你可以隐式或显式地在事务中创建集合.但是,必须使用针对 4.4 更新的 MongoDB 驱动程序.有关详细信息,请参阅在事务中创建集合和索引.
  - 上面这部分内容也是从互联网摘抄过来的.因为写的很好,所以我就不做更改了,只是驱动这方面我们一般都是使用最新版的驱动.而我的 MongoDB 版本是 6.x 所以支持方面没问题.
- 上面的内容参考了几篇文章我一一列出来.
  - [中文版](https://mongoing.com/archives/81286) : https://mongoing.com/archives/81286
  - [官网版](https://www.mongodb.com/docs/manual/core/transactions-in-applications) : https://www.mongodb.com/docs/manual/core/transactions-in-applications
  - [腾讯社区](https://cloud.tencent.com/developer/article/1590862) : https://cloud.tencent.com/developer/article/1590862

---

- 其实写了那么多,我们并不怎么关心那么复杂的概念.
- 接下来我们直接使用代码来体现 MongoDB 的事务特点.
- 按照老惯例打开我们的 MongoCRUD 项目,(后期是否需要给他改个名?改成 MongoSample 吧)
- [Github 地址](https://github.com/joesdu/MongoCRUD) : https://github.com/joesdu/MongoCRUD
- 😔,又到了起名环节,以及数据结构如何体现.由于事务会涉及到多个集合,所以这次我们可能需要 2+ 个实体对象.
- 想不出来什么好的结构了,就用 Cat 和 Dog 吧,刚好他们也是我非常喜欢的两种动物.也希望大家有养毛孩子的好好爱护他们.

- 我们在 Models 文件夹中新建一个类,Animal.cs 由于这里我们更注重 MongoDB 的使用,所以项目结构和数据结构没有太多讲究,因此我将 Cat 和 Dog 均写在了这个文件中,代码如下:

```csharp
public class Animal
{
    /// <summary>
    /// 数据ID
    /// </summary>
    public string Id { get; set; } = string.Empty;
    /// <summary>
    /// 序号
    /// </summary>
    public int No { get; set; }
    /// <summary>
    /// 名字
    /// </summary>
    public string Name { get; set; } = string.Empty;
    /// <summary>
    /// 描述
    /// </summary>
    public string Description { get; set; } = string.Empty;
}
/// <summary>
/// 可爱猫咪
/// </summary>
public sealed class Cat : Animal { }
/// <summary>
/// 可爱狗狗
/// </summary>
public sealed class Dog : Animal { }
```

- 新增了类型,所以还是需要调整一下 DbContext.cs 以及新增一个控制器来体现事务的操作,避免和之前的混合在一起,造成理解上的困难.

```csharp
// 首先在DbContext.cs中写入刚写的两个类型.
/// <summary>
/// 可爱猫猫
/// </summary>
public IMongoCollection<Cat> Cat => Database.GetCollection<Cat>("cute.cat");
/// <summary>
/// 可爱狗狗
/// </summary>
public IMongoCollection<Dog> Dog => Database.GetCollection<Dog>("cute.dog");
```

- 创建新的控制器 TransactionController.cs 并填入一些初始代码.

```csharp
[Route("api/[controller]"), ApiController]
public class TransactionController : ControllerBase
{
    private readonly DbContext _db;
    public TransactionController(DbContext db)
    {
        _db = db;
    }

    private readonly FilterDefinitionBuilder<Cat> _cbf = Builders<Cat>.Filter;
    private readonly UpdateDefinitionBuilder<Cat> _cbu = Builders<Cat>.Update;
    private readonly FilterDefinitionBuilder<Dog> _dbf = Builders<Dog>.Filter;
    private readonly UpdateDefinitionBuilder<Dog> _dbu = Builders<Dog>.Update;
}
```

- 这次我们也不初始化数据了,直接在一个 API 中将事务,数据添加,修改和删除一起干了.
- 比如我们先批量添加 100 个 猫咪 🐱 和 狗狗 🐕,然后将序号大于 50 的猫咪名称改成 Tom,狗狗的名称改成 Spike,然后将序号小于 10 的猫咪和狗狗都删掉.
- Tom 和 Spike 都有了,下次给 Jerry 也安排上 😁(猫和老鼠角色)
- 由于我们的 MongoDB 集群环境大于 4.4 版本.所以我们可以直接插入数据,并不需要事先创建集合.

```csharp
[HttpPost]
public async Task Demo()
{
    var cats = new List<Cat>();
    var dogs = new List<Dog>();
    // 这个地方的语法是从Kotlin中学来的,使用CustomIntEnumeratorExtension.cs实现
    foreach (var index in ..99)
    {
        cats.Add(new()
        {
            No = index,
            Name = $"Cat-{index}",
            Description = "Tom Cat"
        });
        dogs.Add(new()
        {
            No = index,
            Name = $"Dog-{index}",
            Description = "Spike Dog"
        });
    }
    // 后期为了模拟异常,这里用一个try catch
    // 这里就开始要用事务了.先获取 session
    var session = await _db.Client.StartSessionAsync();
    try
    {
        // 这里记住一定要开始事务,不然也不行.
        session.StartTransaction();
        // 这里的第一个参数一定要传,不然就不会使用事务.
        await _db.Cat.InsertManyAsync(session, cats);
        await _db.Dog.InsertManyAsync(session, dogs);
        _ = await _db.Cat.UpdateManyAsync(session, _cbf.Gt(c => c.No, 50), _cbu.Set(c => c.Name, "Tom"));
        _ = await _db.Dog.UpdateManyAsync(session, _dbf.Gt(c => c.No, 50), _dbu.Set(c => c.Name, "Spike"));
        _ = await _db.Cat.DeleteManyAsync(session, c => c.No < 10);
        _ = await _db.Dog.DeleteManyAsync(session, c => c.No < 10);
        // 完成事务的操作后提交事务.
        await session.CommitTransactionAsync();
    }
    catch (Exception)
    {
        // 若是发生异常,退出事务
        await session.AbortTransactionAsync();
    }
}
```

- 执行上边的接口后,去数据库查看数据,发现均是按照我们的预期操作.

```json
// Cat数据
...
// 40
{
    "_id": ObjectId("63c7d456ac7bee95e4ca923d"),
    "no": NumberInt("49"),
    "name": "Cat-49",
    "description": "Tom Cat"
}
// 41
{
    "_id": ObjectId("63c7d456ac7bee95e4ca923e"),
    "no": NumberInt("50"),
    "name": "Cat-50",
    "description": "Tom Cat"
}
// 42
{
    "_id": ObjectId("63c7d456ac7bee95e4ca923f"),
    "no": NumberInt("51"),
    "name": "Tom",
    "description": "Tom Cat"
}
// 43
{
    "_id": ObjectId("63c7d456ac7bee95e4ca9240"),
    "no": NumberInt("52"),
    "name": "Tom",
    "description": "Tom Cat"
}
...
// Gog数据
...
// 40
{
    "_id": ObjectId("63c7d456ac7bee95e4ca92a1"),
    "no": NumberInt("49"),
    "name": "Dog-49",
    "description": "Spike Dog"
}
// 41
{
    "_id": ObjectId("63c7d456ac7bee95e4ca92a2"),
    "no": NumberInt("50"),
    "name": "Dog-50",
    "description": "Spike Dog"
}
// 42
{
    "_id": ObjectId("63c7d456ac7bee95e4ca92a3"),
    "no": NumberInt("51"),
    "name": "Spike",
    "description": "Spike Dog"
}
// 43
{
    "_id": ObjectId("63c7d456ac7bee95e4ca92a4"),
    "no": NumberInt("52"),
    "name": "Spike",
    "description": "Spike Dog"
}
```

- 最后我们试试删除所有数据,和修改一些数据,并在程序中抛出异常,观察事务失败的情况.
- 先将序号小于 50 的猫咪和狗狗.修改名称,Cat 改成 Spike,Dog 改成 Tom,然后再将所有数据全删除.

```csharp
[HttpPost]
public async Task WithError()
{
    var session = await _db.Client.StartSessionAsync();
    try
    {
        // 这里记住一定要开始事务,不然也不行.
        session.StartTransaction();
        _ = await _db.Cat.UpdateManyAsync(session, _cbf.Lte(c => c.No, 50), _cbu.Set(c => c.Name, "Spike"));
        _ = await _db.Dog.UpdateManyAsync(session, _dbf.Lte(c => c.No, 50), _dbu.Set(c => c.Name, "Tom"));
        _ = await _db.Cat.DeleteManyAsync(session, _ => true);
        _ = await _db.Dog.DeleteManyAsync(session, _ => true);
        throw new("error");
        // 完成事务的操作后提交事务.
        await session.CommitTransactionAsync();
    }
    catch (Exception)
    {
        // 若是发生异常,退出事务
        await session.AbortTransactionAsync();
    }
}
```

- 执行后,再去数据库查看数据,会发现并没有什么变化.
- 到这里,事务的使用,和异常后取消事务都展示了,并测试了集群环境中的事务支持是非常 OK 的,所以说 NoSQL 数据库没有事务的说法也不成立了.
- 不过需要注意的是事务只支持副本集集群和分片集群,单机的话写这样的代码可能不会报错,但是和不用事务的效果应该一样的.也就是说后续的操作失败了,之前的操作并不会回退.造成数据异常.
- 事务的简单使用就介绍到这里了,想详细了解它的原理和一些细节的小伙伴推荐看看官网,并且每个版本发生变化的时候,就看下变更日志.
- C# 源码也已经同步到 GitHub,有需要的可以参考和关注一下.
