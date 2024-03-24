### MongoDB 在.NET 中使用 Linq 3 的数组操作

> 最近由于没有上班,写代码的量比以前少多了,并且很多新的东西并没有及时跟进.所以这个变化还是前几天群里看到有人讨论 MongoDB 的时候发现的.于是我也去 Jira 上看了下更新内容.本文简短的介绍下差异.

**数据结构**

- 这里还是用以前哆啦 A 梦的例子中的数据结构.

```json
{
    "_id": ObjectId("660007c9fe3f76f29b685406"),
    "name": "野比家",
    "members": [
        {
            "_id": ObjectId("660007c9fe3f76f29b685402"),
            "index": NumberInt("0"),
            "name": "野比大助",
            "age": NumberInt("40"),
            "gender": "男",
            "birthday": "1943-01-24"
        },
        {
            "_id": ObjectId("660007c9fe3f76f29b685403"),
            "index": NumberInt("1"),
            "name": "野比玉子",
            "age": NumberInt("34"),
            "gender": "女",
            "birthday": "1941-09-30"
        },
        {
            "_id": ObjectId("660007c9fe3f76f29b685404"),
            "index": NumberInt("2"),
            "name": "野比大雄",
            "age": NumberInt("10"),
            "gender": "男",
            "birthday": "1964-08-07"
        },
        {
            "_id": ObjectId("660007c9fe3f76f29b685405"),
            "index": NumberInt("3"),
            "name": "哆啦A梦",
            "age": NumberInt("1"),
            "gender": "男",
            "birthday": "2112-09-03"
        },
        {
            "_id": ObjectId("660007cefe3f76f29b685407"),
            "index": NumberInt("4"),
            "name": "哆啦美",
            "age": NumberInt("1"),
            "gender": "女",
            "birthday": "2114-12-02"
        }
    ]
}
```

在 C#中的数据结构

```csharp
using EasilyNET.Core.Enums;

namespace MongoCRUD.Models;
public class FamilyInfo
{
    /// <summary>
    /// 数据ID
    /// </summary>
    public string Id { get; set; } = string.Empty;

    /// <summary>
    /// 名称
    /// </summary>
    public string Name { get; set; } = string.Empty;

    /// <summary>
    /// 家庭成员
    /// </summary>
    public List<Person> Members { get; set; } = [];
}
// ReSharper disable once ClassNeverInstantiated.Global
public class Person
{
    /// <summary>
    /// 数据ID
    /// </summary>
    public string Id { get; set; } = string.Empty;

    /// <summary>
    /// 和例子中的数据结构同步
    /// </summary>
    public int Index { get; set; }

    /// <summary>
    /// Name
    /// </summary>
    public string Name { get; set; } = string.Empty;

    /// <summary>
    /// 年龄
    /// </summary>
    public int Age { get; set; }

    /// <summary>
    /// 性别
    /// </summary>
    public EGender Gender { get; set; } = EGender.男;

    /// <summary>
    /// 临时决定,使用.Net 6新增类型保存生日,同时让例子变得丰富,明白如何将MongoDB不支持的数据类型序列化
    /// </summary>
    public DateOnly Birthday { get; set; }
}
```

在过去的例子中我们是希望将哆啦美的名字更新成日文的形式.在 MongoDB 数组的操作中,在原来的 Linq2 的版本中可以直接使用[-1]的形式来表示当前的元素,所以有如下代码可以实现更新操作.

```csharp
[Route("api/[controller]"), ApiController]
public class MongoArrayController(DbContext db) : ControllerBase
{
    private readonly UpdateDefinitionBuilder<FamilyInfo> _bu = Builders<FamilyInfo>.Update;

    [HttpPut("UpdateOne")]
    public async Task UpdateOneElement()
    {
        // 这里我们举得例子是将哆啦美的名字变更为日文名字.
        // 这里我们假设查询参数同样是通过参数传入的,所以我们写出了如下代码.
        await db.FamilyInfo.UpdateOneAsync(c => (c.Name == "野比家") & c.Members.Any(s => s.Index == 4),
            _bu.Set(c => c.Members.[-1].Name, "ドラミ"));
    }
}
```

为了方便演示,所以我将代码直接写在了控制器中,可以看到我们的查询条件为:

```csharp
c => (c.Id == "660007c9fe3f76f29b685406") & c.Members.Any(s => s.Index == 4)
```

这里使用了我之前的开源库,所以实体类中的 Id 属性可以自动和 ObjectId 进行互相转换.有兴趣的可以查看下.回到正题.其中 `c.Members.Any(s => s.Index == 4)` 则可以查询出属于哆啦美的元素.接下来使用 `_bu.Set(c => c.Members.[-1].Name, "ドラミ")` 即可更新.

上面是基于 Linq 2 的用法,虽然目前 MongoDB 的驱动还是可以手动切换到 Linq 2 的版本,但是还是推荐大家新项目使用 Linq 3.接下来介绍下官方新增的 3 种函数.

> -1 was not a good choice as way to represent the positional operator.<br><br>
> In LINQ3 we are going to be using custom extension methods instead to represent the three different kinds of positional operators available in update specifications:

上面是官方人员的原文.简单的意思就是: -1 不是表示位置运算符的好选择.在 LINQ3 中,我们将使用自定义扩展方法来表示更新规范中可用的三种不同类型的位置运算符:

```csharp
x.A.FirstMatchingElement() => "A.$"
x.A.AllElements() => "A.$[]"
x.A.AllMatchingElements("identifier") => "A.$[identifier]"
```

<details> 
<summary style="font-size: 14px">FirstMatchingElement</summary>

正如字面意思,FirstMatchingElement 方法的含义表示第一个匹配的元素.所以上面的例子可以改写为如下的形式:

```csharp
[Route("api/[controller]"), ApiController]
public class MongoArrayController(DbContext db) : ControllerBase
{
    private readonly UpdateDefinitionBuilder<FamilyInfo> _bu = Builders<FamilyInfo>.Update;

    [HttpPut("UpdateOne")]
    public async Task UpdateOneElement()
    {
        // 这里我们举得例子是将哆啦美的名字变更为日文名字.
        // 这里我们假设查询参数同样是通过参数传入的,所以我们写出了如下代码.
        await db.FamilyInfo.UpdateOneAsync(c => (c.Name == "野比家") & c.Members.Any(s => s.Index == 4),
            _bu.Set(c => c.Members.FirstMatchingElement().Name, "ドラミ"));
    }
}
```

</details>

<details> 
<summary style="font-size: 14px">AllElements</summary>

正如字面意思,AllElements 方法的含义表示匹配的所有元素.比如我们希望将上述数据中的所有 Name 字段改成:ドラミ,则可以将查询条件改一下.

```csharp
[Route("api/[controller]"), ApiController]
public class MongoArrayController(DbContext db) : ControllerBase
{
    private readonly UpdateDefinitionBuilder<FamilyInfo> _bu = Builders<FamilyInfo>.Update;

    [HttpPut("UpdateOne")]
    public async Task UpdateOneElement()
    {
        // 这里我们举得例子是将哆啦美的名字变更为日文名字.
        // 这里我们假设查询参数同样是通过参数传入的,所以我们写出了如下代码.
        await db.FamilyInfo.UpdateOneAsync(c => (c.Name == "野比家"),
            _bu.Set(c => c.Members.AllElements().Name, "ドラミ"));
    }
}
```

</details>

---

最后的 **AllMatchingElements** 函数我目前确实也没有找出使用方法,我看了官网的内容有 ArrayFilters 的 UpdateOption 的选项,但是在.NET 的 C# 代码中目前还没有测试出来应该如何使用.不过我已经在社区提问了,等待大佬给我一个例子.同样,若是有谁知道也可以给我一个例子.不过我不是希望那种拼接字符串的方式,或者利用 BsonDocument 的方式,因为里面有很多魔法字符串,写代码我是很不建议使用魔法字符,因为若是后期发生变化的话,将及其难以维护和调整.

这次调整,我也将以前的 MongoCRUD 库的代码进行了更新,同时将原始的配置调整为 EasilyNET 库引用的形式.最后附上两个库的地址.

https://github.com/joesdu/MongoCRUD | https://github.com/EasilyNET/EasilyNET
