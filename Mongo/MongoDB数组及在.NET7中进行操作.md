## MongoDB 数组及在.NET7 中进行操作

- MongoDB 作为非常优秀的一款 NoSQL 文档型数据库,在日常的使用中我发现他不但可以替代传统的关系型数据库,而且性能非常优异,若是使用分片集群,更是离谱.
- 并且在我们的业务系统中实践多年也是没有任何问题.各种业务场景均可以解决.
- 而我们的数据往往是复杂多样的,常常一个实体中会包含 List, Array, 或者 IEnumerable\<T> 类型的数据
- 这类数据在序列化为 JSON 后,往往都是以数组类型的格式.所以 MongoDB 作为 Bson 文档存放数据的数据库中就会有很多数组类型的数据.
- 然而数组类型的数据在操作的时候又和普通的文档不一样,在传统数据库中也不存在这类数据.
- 所以就产生了今天的话题,如何给 MongoDB 的数组元素中新增数据,查询数组中的数据,以及更新和删除.

- 所以本文一共 4 个知识点.(其实还有一个知识点,使用 \$unwind 对数组元素实现分页查询这种操作.)

---

- 首先我们先想一个数据结构出来.由于前边的文章和例子中,我们使用到 Person.cs 这个类,所以这里还是使用他做一个新的类型出来.
- 我看到群里有人发 哆啦 A 梦 的表情包,我就想起了出来一个简要的数据结构.
- 我们用 name 表示野比家,members 表示家庭成员,家庭成员就用之前的 person 对象.Nice!👍,当然文章中的 ID 都是假的,我随便写的.真正插入数据库才是真的 ID.

```json
// 前面的Person对象数据结构
{
    "_id": ObjectId("63c27d0a2c369b078ee1d082"),
    "index": NumberInt("0"),
    "name": "张三",
    "age": NumberInt("20"),
    "gender": "女",
    "birthday": "2023-01-14"
}
// 我们新增的数据对象
{
    "_id": ObjectId("63c27d0a2c369b078ee1d083"),
    "name": "野比家",
    "members": [
        {
            "_id": ObjectId("63c27d0a2c369b078ee1d084"),
            "index": NumberInt("0"),
            "name": "野比大助",
            "age": NumberInt("40"),
            "birthday": "1943-01-24"
        },
        {
            "_id": ObjectId("63c27d0a2c369b078ee1d085"),
            "index": NumberInt("1"),
            "name": "野比玉子",
            "age": NumberInt("34"),
            "birthday": "1941-09-30"
        },
        {
            "_id": ObjectId("63c27d0a2c369b078ee1d086"),
            "index": NumberInt("2"),
            "name": "野比大雄",
            "age": NumberInt("10"),
            "birthday": "1964-08-07"
        },
        {
            "_id": ObjectId("63c27d0a2c369b078ee1d087"),
            "index": NumberInt("3"),
            "name": "哆啦A梦",
            "age": NumberInt("1"),
            "birthday": "2112-09-03"
        }
    ]
}
```

- 其实我也还没有直接用 Navicat 使用查询语句直接处理过数据,都是用的 .NET 程序处理的.
- 既然也没用过,那就边写边学.😁
- 这次就先写 .NET 中 C# 的写法,写完后我有一招可以直接变成 JavaScript 版本的查询语句,以后遇到问题这种办法也是一种排错的好办法.后边再介绍.
- 首先新增了一个数据类型,先创建一个 FamilyInfo.cs 类,按照上边的数据结构完成实体的代码.

```csharp
// FamilyInfo.cs
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
    public List<Person> Members { get; set; } = new();
}
```

- 接下来配置一下 DBContext.cs ,新增如下内容

```csharp
/// <summary>
/// 家庭信息
/// </summary>
public IMongoCollection<FamilyInfo> FamilyInfo => Database.GetCollection<FamilyInfo>("family.info");
```

- 同时为了方便管理,源码中代码结构发生了一点变化,将实体全部移动到 Models 文件夹中了.
- 再创建一个 MongoArrayController.cs 控制器,用来对本文中的例子进行编码.
- 按照老惯例,先将 DBContext 通过构造函数注入.并准备两个常用的对象 \_bf 和 \_bu

```csharp
[Route("api/[controller]"), ApiController]
public class MongoArrayController : ControllerBase
{
    private readonly DbContext _db;
    public MongoArrayController(DbContext db)
    {
        _db = db;
    }

    private readonly FilterDefinitionBuilder<FamilyInfo> _bf = Builders<FamilyInfo>.Filter;
    private readonly UpdateDefinitionBuilder<FamilyInfo> _bu = Builders<FamilyInfo>.Update;
}
```

- 再新建一个 Init 接口用来初始化一个数据.

```csharp
[HttpPost("Init")]
public async Task<FamilyInfo> Init()
{
    var obj = new FamilyInfo()
    {
        Name = "野比家",
        Members = new()
        {
            new()
            {
                Id = ObjectId.GenerateNewId().ToString(),
                Index = 0,
                Name = "野比大助",
                Age = 40,
                Gender = Gender.男,
                Birthday = DateOnly.ParseExact("1943-01-24","yyyy-MM-dd")
            },
            new()
            {
                Id = ObjectId.GenerateNewId().ToString(),
                Index = 1,
                Name = "野比玉子",
                Age = 34,
                Gender = Gender.女,
                Birthday = DateOnly.ParseExact("1941-09-30","yyyy-MM-dd")
            },
            new()
            {
                Id = ObjectId.GenerateNewId().ToString(),
                Index = 2,
                Name = "野比大雄",
                Age = 10,
                Gender = Gender.男,
                Birthday = DateOnly.ParseExact("1964-08-07","yyyy-MM-dd")
            },
            new()
            {
                Id = ObjectId.GenerateNewId().ToString(),
                Index = 3,
                Name = "哆啦A梦",
                Age = 1,
                Gender = Gender.男,
                Birthday = DateOnly.ParseExact("2112-09-03","yyyy-MM-dd")
            }
        }
    };
    await _db.FamilyInfo.InsertOneAsync(obj);
    return obj;
}
```

- 好,数据有了,这下就可以来正文了,首先来讲新增数据.添加一个哆啦美.首先用 JSON 格式展示一下数据.

```json
{
    "_id": ObjectId("63c27d0a2c369b078ee1d088"),
    "index": NumberInt("4"),
    "name": "哆啦美",
    "age": NumberInt("1"),
    "birthday": "2114-12-02"
}
```

- 接下来将这个数据加入到数组中.
- 一般我们的查询条件以及插入的数据均需要参数传入,为了方便这里我们直接写.

```csharp
[HttpPost("Create")]
public async Task<Person> AddOneElement()
{
    var dorami = new Person
    {
        Id = ObjectId.GenerateNewId().ToString(),
        Index = 4,
        Name = "哆啦美",
        Age = 1,
        Gender = Gender.女,
        Birthday = DateOnly.ParseExact("2114-12-02", "yyyy-MM-dd")
    };
    _ = await _db.FamilyInfo.UpdateOneAsync(c => c.Name == "野比家", _bu.Push(c => c.Members, dorami));
    return dorami;
}
```

- 同时我们将 JavaScript 脚本语句也写出来,因为新增比较简单.使用 update 或者 updateOne 函数和 $push 操作符即可
- 首先我们来看看语法

```JavaScript
db_name.collection_name.update(
    <query>,
    {$push:{filed:value}}
);
```

| 参数            | 说明                          |
| --------------- | ----------------------------- |
| db_name         | 数据库名                      |
| collection_name | 集合名                        |
| query           | 必选项,是设置更新的文档的条件 |
| filed           | 需要添加的字段                |
| value           | 需要添加的字段的值            |

```JavaScript
// 根据上边的语法就可以写出如下的脚本.
db.getCollection("family.info").updateOne({
    'name': '野比家'
}, {
    $push: {
        'members': {
            "_id": new ObjectId(),
            "index": NumberInt("4"),
            "name": "哆啦美",
            "age": NumberInt("1"),
            "gender": "女",
            "birthday": "2114-12-02"
        }
    }
});
```

- 新增已经搞定了,接下来就继续搞更新数组中特定的一个数据.同样新建一个函数 UpdateOneElement

```csharp
[HttpPut("UpdateOne")]
public async Task UpdateOneElement()
{
    // 这里我们举的例子是将哆啦美的名字变更为日文名字.
    // 这里我们假设查询参数同样是通过参数传入的,所以我们写出了如下代码.
    _ = await _db.FamilyInfo.UpdateOneAsync(
        c => c.Name == "野比家" & c.Members.Any(s => s.Index == 4),
        _bu.Set(c => c.Members[-1].Name, "ドラミ"));
}
```

- 接下来我们协议下 JavaScript 脚本的代码.由于 C# 代码我们变成了日语名,这里变成中文名测试

```JavaScript
db.getCollection("family.info").updateOne({
    'name': "野比家",
    "members.index": 4
}, {
    $set: {
        "members.$.name": "哆啦美"
    }
});
```

- 注意此处的 \$ 符号 .Net 驱动正是将 -1 替换为了 \$ 符号.
- 妈耶,刚学会 Markdown 文档输入 \$ 符号的问题.🥲 总算是学会了.好了,继续回到正题.接下来我们就写
- 在查资料的过程中还学到一些细节问题,我也顺路搬过来了.

- 变动时多字段问题

  - A.定义的类里面新加了字段而 MongoD 数据库没加,不影响.新加的字段会默认赋值为其类型的默认值
  - B.MongoDB 数据库中的新加了字段而定义的类中没加,会报错.但可通过在类上添加特性 BsonIgnoreExtraElements 来忽略数据库中多出的字段

- 前边展示了更新一个字段的情况,若是更新多个字段也可以写多个 \$set,C#代码中就是多个 Set 函数.也可以将整个对象查询出来,修改字段值后,直接更新整个元素.
- 介绍了更新数组中某个元素后,就可以继续写从数组中删除一个元素了.

```csharp
[HttpDelete("DeleteOne")]
public async Task DeleteOneElement()
{
    _ = await _db.FamilyInfo.UpdateOneAsync(
        c => c.Name == "野比家",
        _bu.PullFilter(c => c.Members, f => f.Index == 4));
}
```

- 先介绍下 JavaScript 脚本语法,这里需要使用 \$pull 操作符从现有数组中移除与指定条件匹配的值或值的所有实例

```json
// 语法
{ $pull: { <field1>: <value|condition>, <field2>: <value|condition>, ... } }
//  说明：
//      如果指定的<condition>数组元素为内嵌文档时，$pull操作符应用<condition>，类似每个数组元素是集合中的文档一样
//      如果指定的<value>去移除数组，$pull仅仅移除满足指定条件的数组元素(精确匹配，包括顺序)
//      如果指定的<value>去移除一个文档，$pull仅仅移除字段和值精确匹配的数组元素素(顺序可以不同)
```

- JavaScript 脚本代码如下:

```JavaScript
db.getCollection("family.info").updateOne({
    'name': "野比家"
}, {
    $pull: {
        'members': {
            'name': "哆啦美"
        }
    }
});
// 其中 members 中的条件还可以使用别的操作符如 $in 等匹配多个元素.
```

- 还有一个 \$pullAll 操作符感兴趣的也可以了解下.这里就不详细展开描述了.[参考链接](https://www.mongodb.com/docs/manual/reference/operator/update/pullAll)
- https://www.mongodb.com/docs/manual/reference/operator/update/pullAll
- 查的话其实都不用说了,直接返回数组成员即可.这里还是写一个例子吧.同时将如何查看 C# 代码生成的查询语句方式一起写这里.
- 注意看 sql 临时变量,将 IFindFluent ToString 后即可查看到生成的 MongoDB 查询语句的条件.所以在调试的时候,可以使用这种办法获取脚本代码.方便在 navicat 中调试.
- 比如这段代码生产的查询语句如下:

```JavaScript
find({ "name" : "野比家" }, { "members" : 1, "_id" : 0 })
```

```csharp
[HttpGet("OneElement")]
public async Task<Person> GetOneElement()
{
    //var sql = _db.FamilyInfo.Find(c => c.Name == "野比家")
    //        .Project(c => c.Members.First()).ToString();
    return await _db.FamilyInfo.Find(c => c.Name == "野比家")
        .Project(c => c.Members.First()).SingleOrDefaultAsync();
    // 下面这行代码可以体现出也可以在Project中使用函数筛选
    //return await _db.FamilyInfo.Find(c => c.Name == "野比家")
    //    .Project(c => c.Members.Find(s => s.Name == "哆啦A梦")).SingleOrDefaultAsync();
}

[HttpGet("AllElement")]
public async Task<IEnumerable<Person>> GetAllElement()
{
    return await _db.FamilyInfo.Find(c => c.Name == "野比家")
        .Project(c => c.Members).SingleOrDefaultAsync();
}
```

- 接下来是比较难的东西了, \$unwind 从输入文档中解构一个数组字段,为每个元素输出一个文档,由于 JavaScript 脚本不会写,先写 C# 版本的代码.
- 由于 $unwind 的一些特殊性,我们添加一些额外的类方便我们的处理,先创建一个 UnwindObj\<T> 的类型,作为所有 \$unwind 操作变化的一个基类或者直接使用.

```csharp
public class UnwindObj<T>
{
    /// <summary>
    /// 1.T as List,use for Projection,
    /// 2.T as single Object,use for MongoDB array field Unwind result
    /// </summary>
    [BsonElement("Obj")]
    public T? Obj { get; set; }
    /// <summary>
    /// when T as List,record Count
    /// </summary>
    [BsonElement("Count")]
    public int Count { get; set; }
    /// <summary>
    /// record array field element's index before Unwinds
    /// </summary>
    [BsonElement("Index")]
    public int Index { get; set; }
}
```

- 然后就是写查询语句了.

```csharp
[HttpGet("Unwind")]
public async Task<dynamic> GetUnwind()
{
    // Project中的UnwindObj我们往往使用子类,这样可以将一些不必要的数据屏蔽或者丢弃
    var query = _db.FamilyInfo.Aggregate().Match(c => c.Name == "野比家")
        .Project(c => new UnwindObj<List<Person>>
        {
            Obj = c.Members,
            Count = c.Members.Count
        })
        .Unwind(c => c.Obj, new AggregateUnwindOptions<UnwindObj<Person>> { IncludeArrayIndex = "Index" });
    //var sql = query.ToString();
    var total = query.Count().FirstOrDefaultAsync().Result.Count;
    var list = await query.Skip(2).Limit(1).ToListAsync();
    return new Tuple<long?, List<UnwindObj<Person>>>(total, list);
}
```

- 通过运行程序在 Swagger 页面中执行接口后查看数据,我们知道我们代码没有错.

```json
{
  "item1": 5,
  "item2": [
    {
      "obj": {
        "id": "63c6a1922521df30add4c49b",
        "index": 2,
        "name": "野比大雄",
        "age": 10,
        "gender": "男",
        "birthday": "1964-08-07"
      },
      "count": 5,
      "index": 2
    }
  ]
}
```

- 那么 JavaScript 脚本应该怎么写呢,通过代码中的 sql 临时变量,在调试的时候我就观察到了,生成的查询语句,所以这就可以很方便的写出来,这也是为什么本文先写 C# 代码的原因之一.

```JavaScript
db.getCollection("family.info").aggregate([{
    "$match": {
        "name": "野比家"
    }
}, {
    "$project": {
        "Obj": "$members",
        "Count": {
            "$size": "$members"
        },
        "_id": 0
    }
}, {
    "$unwind": {
        "path": "$Obj",
        "includeArrayIndex": "Index"
    }
}]);
```

- 到这里本文的内容就全部结束了,讲的可能并不是特别详细,但是针对我目前项目中的情况都算是讲到了.
- 同时 C#源码也会[同步上传到 GitHub,有兴趣的可以关注一下](https://github.com/joesdu/MongoCRUD)
- https://github.com/joesdu/MongoCRUD
- 其中 Unwind 的基础类,在 [Hoyo.Mongo](https://www.nuget.org/packages/Hoyo.Mongo) 库中也已经内置,若是使用该库,可以更方便操作 MongoDB
- https://www.nuget.org/packages/Hoyo.Mongo
- [Hoyo.Mongo 源码地址](https://github.com/joesdu/Hoyo)
- https://github.com/joesdu/Hoyo
