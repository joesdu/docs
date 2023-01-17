## MongoDB 数组及在.NET7 中进行操作

- MongoDB 作为非常优秀的一款 NoSQL 文档型数据库,在日常的使用中我发现他不但可以替代传统的关系型数据库,而且性能非常优异,若是使用分片集群,更是离谱.
- 并且在我们的业务系统中实践多年也是没有任何问题.各种业务场景均可以解决.
- 而我们的数据往往是复杂多样的,常常一个实体中会包含 List, Array, 或者 IEnumerable\<T> 类型的数据
- 这类数据在序列化为 JSON 后,往往都是以数组类型的格式.所以 MongoDB 作为 Bson 文档存放数据的数据库中就会有很多数组类型的数据.
- 然而数组类型的数据在操作的时候又和普通的文档不一样,在传统数据库中也不存在这类数据.
- 所以就产生了今天的话题,如何给 MongoDB 的数组元素中新增数据,查询数组中的数据,以及更新和删除.
- 有了上边的问题后,又产生一个其他问题,如何把数组中的元素变成一个表格的形式展示出来.比如这样一个场景:
- 前端需要显示某个人的家庭成员信息表,在数据库中我们可能将所有家庭成员均放在一个数组中,我们返回给前端的时候,想让他看起来和普通的 List 类型的数据没什么差别.这时候我们就发现 MongoDB 也支持这种功能.
- 所以本文一共 5 个知识点,数组类型的 CRUD 以及数组转成表的方法.

---

- 首先我们先想一个数据结构出来.由于前边的文章和例子中,我们使用到 Person.cs 这个类,所以这里还是使用他做一个新的类型出来.
- 我看到群里有人发 哆啦 A 梦 的表情包,我就想起了出来一个简要的数据结构.
- 我们用 name 表示野比家,members 表示家庭成员,家庭成员就用之前的 person 对象.Nice!👍

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
