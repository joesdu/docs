## MongoDB 索引以及在.NET7 中如何创建索引

> 索引通常能够极大的提高查询的效率,如果没有索引,MongoDB 在读取数据时必须扫描集合中的每个文件并选取那些符合查询条件的记录.其他关系型数据库也会出现全表扫描.这种扫描全集合的查询效率是非常低的,特别在处理大量的数据时,查询可以要花费几秒到几十秒甚至超过达到分钟级,这对网站的性能是非常致命的.索引是特殊的数据结构,索引存储在一个易于遍历读取的数据集合中,索引是对数据库表中一列或多列的值进行排序的一种结构.

### 索引简介

- 通常数据库都有索引(目前没听说过没有索引的),由此可见索引在数据库中是非常重要的存在.
- 使用索引提高常见查询的性能.在其上构建索引 经常出现在查询中的字段以及返回的所有操作的字段 排序结果.MongoDB 会自动在字段上创建一个唯一的索引 \_id .

### 注意事项

- 根据官网的描述,使用索引还有一些注意事项,如下:
  - 每个索引至少需要 8 kB 的数据空间.
  - 添加索引对写操作有一些负面的性能影响.对于具有高写-读比率的集合,索引是昂贵的,因为每个插入必须同时更新任何索引.
  - 具有高读写比的集合通常从额外的索引中受益.索引并不影响无索引的读取操作.
  - 当激活时,每个索引会消耗磁盘空间和内存.这种使用量可能是很大的,应该为容量规划进行跟踪,特别是对工作集大小的关注.

### 索引类型

- 在 MongoDB 中的索引类型非常的多.所以这里我们先列举一下索引的各种类型.
- 不过我们并不会全部讲解,也并不需要全部都熟悉,只需要作为一种了解,在遇到这方面的问题的时候,知道如何查阅资料.
- 我看了下[官网文档](https://www.mongodb.com/docs/manual/indexes)
- https://www.mongodb.com/docs/manual/indexes/

|       类型       | 简述                                                                                                                                                                                                                                                         | 英文原文                                                                                                                                                                                                                                                                                                                                                                      |
| :--------------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|    单字段索引    | MongoDB 提供对文档集合中任何字段的索引的完整支持。默认情况下，所有集合都有一个关于\_id 字段的索引，应用程序和用户可以添加额外的索引来支持重要的查询和操作。                                                                                                  | MongoDB provides complete support for indexes on any field in a collection of documents. By default, all collections have an index on the \_id field, and applications and users may add additional indexes to support important queries and operations.                                                                                                                      |
|     复合索引     | MongoDB 支持复合索引，其中一个索引结构持有对一个集合的文档中的多个字段的引用.                                                                                                                                                                                | MongoDB supports compound indexes, where a single index structure holds references to multiple fields within a collection's documents.                                                                                                                                                                                                                                        |
|     多键索引     | 为了给持有数组值的字段建立索引，MongoDB 为数组中的每个元素创建一个索引键。这些多键索引支持对数组字段的有效查询。多键索引可以在容纳标量值（如字符串、数字）和嵌套文档的数组上构建。                                                                           | To index a field that holds an array value, MongoDB creates an index key for each element in the array. These multikey indexes support efficient queries against array fields. Multikey indexes can be constructed over arrays that hold both scalar values (e.g. strings, numbers) and nested documents.                                                                     |
|     文本索引     | 要在自我管理的部署上运行文本搜索查询，你必须有一个 文本索引 在你的集合上。MongoDB 提供文本索引以支持对字符串内容的文本搜索查询。文本索引可以包括任何字段，其值是一个字符串或一个字符串元素的数组。一个集合只能有一个文本搜索索引，但该索引可以覆盖多个字段。 | To run text search queries on self-managed deployments, you must have a text index on your collection. MongoDB provides text indexes to support text search queries on string content. Text indexes can include any field whose value is a string or an array of string elements. A collection can only have one text search index, but that index can cover multiple fields. |
|    通配符索引    | MongoDB 支持在一个字段或一组字段上创建索引以支持查询。由于 MongoDB 支持动态模式，应用程序可以对那些名称不能事先知道或任意的字段进行查询。                                                                                                                    | MongoDB supports creating indexes on a field or set of fields to support queries. Since MongoDB supports dynamic schemas, applications can query against fields whose names cannot be known in advance or are arbitrary.                                                                                                                                                      |
|  2dsphere 索引   | 2dsphere 索引支持在类地球体上计算几何图形的查询。2dsphere 索引支持所有 MongoDB 地理空间查询：包含、相交和接近的查询。                                                                                                                                        | A 2dsphere index supports queries that calculate geometries on an earth-like sphere. 2dsphere index supports all MongoDB geospatial queries: queries for inclusion, intersection and proximity.                                                                                                                                                                               |
|     2d 索引      | 对于存储为二维平面上的点的数据，使用 2d 索引。2d 索引旨在用于 MongoDB 2.2 和更早版本中使用的传统坐标对。                                                                                                                                                     | Use a 2d index for data stored as points on a two-dimensional plane. The 2d index is intended for legacy coordinate pairs used in MongoDB 2.2 and earlier.                                                                                                                                                                                                                    |
| geoHaystack 索引 | geoHaystack 索引是一个特殊的索引，它被优化以返回小区域的结果。geoHaystack 索引提高了使用平面几何的查询的性能。                                                                                                                                               | A geoHaystack index is a special index that is optimized to return results over small areas. geoHaystack indexes improve performance on queries that use flat geometry.                                                                                                                                                                                                       |
|     哈希索引     | 散列索引用索引字段的值的散列来维护条目.                                                                                                                                                                                                                      | Hashed indexes maintain entries with hashes of the values of the indexed field.                                                                                                                                                                                                                                                                                               |

- 从表中的而数据我们观察到 MongoDB 的索引还挺多的.但是我们这里一般只使用单字段和符合索引.别的索引依据情况使用.
- 并且从官网文档种我们可以发现 geoHaystack 索引和 2d 索引已经发生变化,所以少了两个索引需要学习.
- geoHaystack 索引 从 MongoDB5.x 版本开使就弃用了.
- 这里还是推荐大家看文章的时候也多看看官网的描述,比如这里的索引问题,官网就还写了 TTL 的属性等其他属性.
- 不过简单的使用的话,也不用去啃英文文档了.😁 看这里就应该够了.

### 语法介绍 createIndex() 方法

- 在 MongoDB 种使用 createIndex() 方法来创建索引。
  > 注意在 3.0.0 版本前创建索引方法为 db.collection.ensureIndex()，之后的版本使用了 db.collection.createIndex() 方法，ensureIndex() 还能用，但只是 createIndex() 的别名。
- 在 MongoDB 中排序和索引的值,一般使用 1 和 -1 表示,1 表示升序,-1 表示降序
- 语法:

```JavaScript
db.collection.createIndex(keys, options);
```

- 以我们之前的 Cat 猫咪集合,举个栗子 🌰
- 我们使用 no 字段来创建一个新索引.

```JavaScript
db.getCollection("cute.cat").createIndex({
    'no': 1
});
```

- 可以看到输出信息, OK:1 说明创建索引成功.

```json
{
    "numIndexesBefore": NumberInt("1"),
    "numIndexesAfter": NumberInt("2"),
    "createdCollectionAutomatically": false,
    "commitQuorum": "votingMembers",
    "ok": 1,
    "$clusterTime": {
        "clusterTime": Timestamp(1674107948, 7),
        "signature": {
            "hash": BinData(0, "ReKPSvXxUES2q/rI8Ppx0Mrqwxc="),
            "keyId": NumberLong("7155695624313110535")
        }
    },
    "operationTime": Timestamp(1674107948, 7)
}
```

- 从上面的语法里我们可以看到,该方法是支持可选参数的,那接下来就简单的枚举下各个可选参数的含义.

| 参数名             |    数据类型(Type)     | 描述                                                                                                                                         |
| ------------------ | :-------------------: | :------------------------------------------------------------------------------------------------------------------------------------------- |
| background         |        Boolean        | 建索引过程会阻塞其它数据库操作，background 可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为 false。            |
| unique             |        Boolean        | 建立的索引是否唯一。指定为 true 创建唯一索引。默认值为 false.                                                                                |
| name               |        string         | 索引的名称。如果未指定，MongoDB 的通过连接索引的字段名和排序顺序生成一个索引名称。                                                           |
| dropDups           |        Boolean        | 3.0+版本已废弃。在建立唯一索引时是否删除重复记录,指定 true 创建唯一索引。默认值为 false.                                                     |
| sparse             |        Boolean        | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为 true 的话，在索引字段中不会查询出不包含对应字段的文档.。默认值为 false. |
| expireAfterSeconds |        integer        | 指定一个以秒为单位的数值，完成 TTL 设定，设定集合的生存时间。                                                                                |
| v                  | index<br>version</br> | 索引的版本号。默认的索引版本取决于 mongod 创建索引时运行的版本。                                                                             |
| weights            |       document        | 索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。                                                                |
| default_language   |        string         | 对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语                                                                        |
| language_override  |        string         | 对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的 language，默认值为 language.                                                  |

- 讲完了简单的单字段索引,就顺带讲一下复合索引.
- 先看看语法.

```javascript
db.collection.createIndex( { <field1>: <type>, <field2>: <type2>, ... } )
```

- 在来举个例子,还是使用 Cat 集合,我们将\_id 字段升序,no 字段降序.

```JavaScript
db.getCollection("cute.cat").createIndex({
    '_id': 1,
    'no': - 1
}, {
    background: true
});
// 这里我们就添加一个可选参数作为例子
```

- 执行脚本后,可以看到输出信息,说明创建成功.

```json
// 1
{
    "numIndexesBefore": NumberInt("3"),
    "numIndexesAfter": NumberInt("4"),
    "createdCollectionAutomatically": false,
    "commitQuorum": "votingMembers",
    "ok": 1,
    "$clusterTime": {
        "clusterTime": Timestamp(1674108743, 7),
        "signature": {
            "hash": BinData(0, "Av/IKMpnnVoKuYO6SboRSQaDBOA="),
            "keyId": NumberLong("7155695624313110535")
        }
    },
    "operationTime": Timestamp(1674108743, 7)
}
```

- 同样在 MongoDB 的索引中也可以支持子文档的字段索引,如前边的 family 集合,我们可以使用如下代码来创建子文档的索引.以及数组字段的索引.

```JavaScript
db.getCollection("family.info").createIndex({
    '_id': 1,
    'members': 1,
}, {
    background: true
});
```

- 子文档的话,由于前边的数据结构中还没有涉及,所以这里写个伪代码.

```json
// 假设数据结构如下.
{
  "_id": 1,
  "name": "李四",
  "linkInfo": {
    "phone": 18588888888,
    "address": "三里屯",
    "zipcode": "569212"
  }
}
```

- 根据上边的数据结构,我们可以这么写子文档的索引创建代码

```JavaScript
db.getCollection("family.info").createIndex({
    '_id': 1,
    'linkInfo.phone': -1,
}, {
    background: true
});
```

- 光是创建了索引,没有删除肯定是不合理的,不然如何管理就成了问题.
- 话也不多说,了解我的都知道我喜欢直接上代码.

```JavaScript
// 删除所有索引
db.getCollection("cute.cat").dropIndexes();
```

- 查看输出信息,可以发现删除所有成功.为了接下来测试删除一个索引的操作,这里先重新创建一下索引.

```json
// 1
{
    "nIndexesWas": NumberInt("4"),
    "msg": "non-_id indexes dropped for collection",
    "ok": 1,
    "$clusterTime": {
        "clusterTime": Timestamp(1674112374, 3),
        "signature": {
            "hash": BinData(0, "5ICTdaUVR4jJtmt08RQ0KNDz2go="),
            "keyId": NumberLong("7155695624313110535")
        }
    },
    "operationTime": Timestamp(1674112374, 3)
}
```

- 删除单个索引规则.由于我们前面没有指定名称,所以生成了一个默认名称.但是我们并不知道名字是什么,所以就查一下.

```JavaScript
db.getCollection("cute.cat").getIndexes();
```

- 输出结果

```json
// 1
[
    {
        "v": NumberInt("2"),
        "key": {
            "_id": NumberInt("1")
        },
        "name": "_id_"
    },
    {
        "v": NumberInt("2"),
        "key": {
            "_id": 1,
            "no": -1
        },
        "name": "_id_1_no_-1",
        "background": true
    }
]
```

- 第一个索引为默认的 \_id 索引,不用管,我们看第二个就是我们生成的,将他的 name 拿过来.

```JavaScript
db.getCollection("cute.cat").dropIndex('_id_1_no_-1');
```

- 执行完成后,可以看到输出结果.说明删除成功.

```json
// 1
{
    "nIndexesWas": NumberInt("2"),
    "ok": 1,
    "$clusterTime": {
        "clusterTime": Timestamp(1674112666, 1),
        "signature": {
            "hash": BinData(0, "yubwfj6jabfxczZtC8MXu2acQfI="),
            "keyId": NumberLong("7155695624313110535")
        }
    },
    "operationTime": Timestamp(1674112666, 1)
}
```

- 使用脚本创建索引就暂时讲到这里了,其实也可以通过可视化工具更方便的创建索引,如使用 Navicat 鼠标邮件,点击设计集合就可切换到索引页签管理索引.这里就不过多的描述了.

### 在.NET 7 中如何使用 C# 代码创建索引呢?

- 作为一个.Net 开发者,肯定是非常喜欢 C# 的,这一定得安排一下.
- 接着请出我们的老朋友 MongoCRUD 项目并用 VS 打开它.
- [Github 链接](https://github.com/joesdu/MongoCRUD)我也再安排一下.
- https://github.com/joesdu/MongoCRUD
- 本次不用新增类型了,继续使用我们的 Cat 对象.
- 但是为了和之前的区分开,还是需要创建一个新的控制器. IndexesController.cs 并将 DbContext 注入.

```csharp
[Route("api/[controller]"), ApiController]
public class IndexesController : ControllerBase
{
    private readonly DbContext _db;
    public IndexesController(DbContext db)
    {
        _db = db;
    }
}
```

- 通过前面 CRUD 以及数组等操作,我们就能猜到,索引在 C# 驱动中的接口也差不多.
- 但是其实猜错了 🤣,在 C# 驱动中,索引是集合的一个属性 Indexes ,然后使用 Indexes 上的函数 CreateOne 或者 CreateMany 来创建一个或者多个索引.同样他们有异步版本.
- 在.Net 的 MongoDB 驱动中,我们创建索引还需要一个 CreateIndexModel\<T> 的对象来作为上述函数(CreateOne,CreateMany...)的参数.
- 所以我们根据前边的需求,写出了下面的代码.

```csharp
[HttpPut("CreateIndexes")]
public async Task CreateIndexes()
{
    var indexModel = new CreateIndexModel<Cat>(
        Builders<Cat>.IndexKeys.Ascending(c => c.Id).Descending(c => c.No),
        new CreateIndexOptions
        {
            Name = "Id 1 No -1",
            Background = true
        });
    _ = await _db.Cat.Indexes.CreateOneAsync(indexModel);
}
```

- 从上面的代码中,我们可以看到涉及到复合索引,以及可选参数的设置.由于单字段索引很简单,所以就不再介绍了,去掉上面代码 `.Descending(c => c.No)` 就变成但字段索引了.
- 到这里创建索引就讲完了,不过索引创建了就能删除,所以再写一下删除索引的例子.
- 在.NET 驱动中,删除索引需要使用到 DropOneAsync 和 DropAllAsync 以及他们的同步版本.
- 先通过 F12 看看他们的参数列表.

```csharp
/// <summary>Drops an index by its name.</summary>
/// <param name="name">The name.</param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>A task.</returns>
Task DropOneAsync(string name, CancellationToken cancellationToken = default (CancellationToken));
/// <summary>Drops an index by its name.</summary>
/// <param name="name">The name.</param>
/// <param name="options">The options. </param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>A task.</returns>
Task DropOneAsync(string name, DropIndexOptions options, CancellationToken cancellationToken = default (CancellationToken));
/// <summary>Drops an index by its name.</summary>
/// <param name="session">The session.</param>
/// <param name="name">The name.</param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>A task.</returns>
Task DropOneAsync(
  IClientSessionHandle session,
  string name,
  CancellationToken cancellationToken = default (CancellationToken));
```

- All 和同步版本就没贴出来.我们可以看到它支持事务,还支持可选参数.事务前边已经讲过如何使用,这里就不再赘述.第一个参数是 name,表示索引的名称.
- 根据上边创建的索引名称,就可以写出如下代码.

```csharp
[HttpPut("DeleteIndexes")]
public async Task DeleteIndexes()
{
    await _db.Cat.Indexes.DropOneAsync("Id 1 No -1");
}
```

- 至此.NET 7 中使用 C# 创建和删除索引的讲解就结束了.
