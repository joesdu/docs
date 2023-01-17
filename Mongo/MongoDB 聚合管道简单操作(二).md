## MongoDB 聚合管道简单操作(二)

- 之前简单的介绍了,MongoDB 聚合管道中的一些阶段的操作符和其含义(用途).
- 这里就继续之前的内容介绍一些常用的表达式运算符.
- 并且结合 MongoDB 语句和 C# WebApi 程序来具体举一些例子.便于学习和理解.
- 首先介绍一下表达式运算符的语法.
- 运算符表达式类似于接受参数的函数. 通常,这些表达式采用一系列参数 并具有以下形式:

```json
/*
{
    操作符: [ 参数1, 参数2, ...]
}
*/
{ <operator>: [ <argument1>, <argument2> ... ] }
```

- 若是运算符接受单个参数.那么可以省略外部数组写成如下形式:

```json
/*
{
    操作符: 参数1
}
*/
{ <operator>: <argument> }
```

- 但是有个注意点: 若要避免在参数是文本数组时分析歧义,必须将文本数组包装在 $literal 表达或保留指定参数列表的外部数组.
- 接下来介绍一些各方面的常用运算符.具体的详细内容可以[参考官网文档](https://www.mongodb.com/docs/manual/reference/operator/aggregation)
- https://www.mongodb.com/docs/manual/reference/operator/aggregation
- 由于篇幅原因,这里仅介绍一些我觉得较为常用的.

| 名称  | 描述                                                                                                                                                       | 英文原文                                                                                                                                                                                                                                          |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $abs  | 绝对值                                                                                                                                                     | Returns the absolute value of a number.                                                                                                                                                                                                           |
| $add  | 将数字相加以返回总和,或将数字和日期添加到返回新日期.如果将数字和日期相加,则处理以毫秒为单位的数字.接受任意数量的参数表达式,但最多一个表达式可以解析为日期. | Adds numbers to return the sum, or adds numbers and a date to return a new date. If adding numbers and a date, treats the numbers as milliseconds. Accepts any number of argument expressions, but at most, one expression can resolve to a date. |
| $ceil | 返回大于或等于指定数字的最小整数                                                                                                                           | Returns the smallest integer greater than or equal to the specified number.                                                                                                                                                                       |
| $pow  | 将数字提高到指定的指数                                                                                                                                     | Raises a number to the specified exponent.                                                                                                                                                                                                        |
| $avg  | 返回数值的平均值.忽略非数值.在 5.0 版更改:在$setWindowFields 阶段.                                                                                         | Returns an average of numerical values. Ignores non-numeric values. Changed in version 5.0: Available in $setWindowFields stage.                                                                                                                  |
| $max  | 返回每个组的最高表达式值.在 5.0 版更改:在$setWindowFields 阶段.                                                                                            | Returns the highest expression value for each group. Changed in version 5.0: Available in $setWindowFields stage.                                                                                                                                 |
| $sum  | 返回数值的总和。忽略非数值.在 5.0 版更改:在$setWindowFields 阶段.                                                                                          | Returns a sum of numerical values. Ignores non-numeric values. Changed in version 5.0: Available in $setWindowFields stage.                                                                                                                       |

---

- 哎 😔,偷个懒,暂时就介绍这么多,主要是一些算数运算,三角函数,逻辑运算以及对数组的一些操作.具体的内容建议参考官网.官网上写的很详细.
- 接下来我们就搞点数据来试试这个聚合管道的使用.
- 不知道大家还记不记得前几篇文章中弄的好几百个张三数据.这里再把数据结构展示一下.

```json
{
    "_id": ObjectId("63c27d0a2c369b078ee1d082"),
    "index": NumberInt("0"),
    "name": "张三",
    "age": NumberInt("20"),
    "gender": "女",
    "birthday": "2023-01-14"
}
```

- 我将之前的数据都清除了,重新使用 InsertMany 函数插入了 100 个数据.

```csharp
[HttpPost("many")]
public async Task<IEnumerable<Person>> InsertMany()
{
    var list = new List<Person>();
    for (var i = 0; i < 100; i++)
    {
        list.Add(new()
        {
            Name = "张三",
            Age = 20 + i,
            Birthday = DateOnly.FromDateTime(DateTime.Now),
            Gender = i % 2 == 0 ? Gender.女 : Gender.男,
            Index = i
        });
    }
    await _db.Person.InsertManyAsync(list);
    return list;
}
```

- 从代码我们可以看出虽然插入了 100 个数据,但是性别有男有女.我们今天就用这个数据来继续展示聚合管道的使用.
- 首先来个简单的,使用 $match 查询所有男张三.

```JavaScript
db.getCollection("person").aggregate([{
    $match: {
        'name': '张三',
        'gender': '男'
    }
}]);
```

- 执行后,通过结果我们可以看到预期结果.

```json
// 1
{
    "_id": ObjectId("63c62f875d61e4ad53914cab"),
    "index": NumberInt("1"),
    "name": "张三",
    "age": NumberInt("21"),
    "gender": "男",
    "birthday": "2023-01-17"
}
// 2
{
    "_id": ObjectId("63c62f875d61e4ad53914cad"),
    "index": NumberInt("3"),
    "name": "张三",
    "age": NumberInt("23"),
    "gender": "男",
    "birthday": "2023-01-17"
}
// 3
{
    "_id": ObjectId("63c62f875d61e4ad53914caf"),
    "index": NumberInt("5"),
    "name": "张三",
    "age": NumberInt("25"),
    "gender": "男",
    "birthday": "2023-01-17"
}
// 4
{
    "_id": ObjectId("63c62f875d61e4ad53914cb1"),
    "index": NumberInt("7"),
    "name": "张三",
    "age": NumberInt("27"),
    "gender": "男",
    "birthday": "2023-01-17"
}
...
```

- 单一个操作符的使用到这里应该就全会了.接下来增加一点点难度.
- 我们来个常用的 Group By ,且年龄大于 40 岁的男张三和女张三的人数,并将结果转成其他格式.

```JavaScript
db.getCollection("person").aggregate([{
    $match: {
        'age': {
            $gt: 40
        }
    }
}, {
    $group: {
        '_id': '$gender',
        'count': {
            $sum: 1
        }
    }
}, {
    $project: {
        '_id': new ObjectId(),
        '性别': '$_id',
        '人数': '$count'
    }
}]);
```

- 执行完成后,输出如下内容.

```json
// 1
// 1
{
    "_id": ObjectId("63c636649be1fb6bb10cda32"),
    "性别": "男",
    "人数": 40
}

// 2
{
    "_id": ObjectId("63c636649be1fb6bb10cda32"),
    "性别": "女",
    "人数": 39
}

```

- 可以很容易的分析出,我们先执行了 $match 匹配了所有大于 40 岁的张三,然后使用 $group 将男女的人数计算出来了(其中使用了 $sum 运算符),然后再使用 $project 将结果转成了我们想要的格式.
- 再来一个非常常用的分页,比如每页 3 个数据,我们查第 4 页的数据.并对年龄排倒叙,对性别排正序.同时年龄大于 30 岁.

```JavaScript
db.getCollection("person").aggregate([
    {
        // 查询条件
        $match: {
            'age': {
                $gt: 30
            }
        }
    },
    {
        $sort: {
            // 排到序: -1 ,正序: 1
            // 同样可以多个条件组合排序,只需要像Json一样,直接写就行.
            'age': - 1,
            'gender': 1
        }
    },
    {
        // 计算需要跳过的数据
        $skip: (4 - 1) * 3
    },
    {
        // 提取的数据量
        $limit: 3
    }
]);
```

- 点击执行后,会出现如下数据,这里的聚合管道顺序很重要,我刚开始顺序不对,查出来的数据就不一样了.
- 分页查询应该是先筛选再排序,最后再

```json
// 1
{
    "_id": ObjectId("63c62f875d61e4ad53914d04"),
    "index": NumberInt("90"),
    "name": "张三",
    "age": NumberInt("110"),
    "gender": "女",
    "birthday": "2023-01-17"
}
// 2
{
    "_id": ObjectId("63c62f875d61e4ad53914d03"),
    "index": NumberInt("89"),
    "name": "张三",
    "age": NumberInt("109"),
    "gender": "男",
    "birthday": "2023-01-17"
}
// 3
{
    "_id": ObjectId("63c62f875d61e4ad53914d02"),
    "index": NumberInt("88"),
    "name": "张三",
    "age": NumberInt("108"),
    "gender": "女",
    "birthday": "2023-01-17"
}
```

- 从数据中我们看不出来对性别排序了,是由于我们没有年龄一样的数据,所以看不太出来.实际上也参与了排序

- 到此聚合管道的初步使用就讲解完成了,聚合管道主要用于一些复杂的统计,数据处理.有了管道的支持,很多事情就可以一步一步的,逻辑很清晰.
- 接下来就是在 C# 中如何使用呢?

---

- 首先我们新建一个控制器 AggregateController.cs 为了和之前的 CRUD 区分.
- 然后常规的使用依赖注入注入我们需要的 DBContext
- 先实现一个简单的 $match ,也可以使用 find 函数进行查找,但是这里我们不这么干.这里使用 Aggregate

```csharp
[HttpGet("Match")]
public async Task<dynamic> GetMatch()
{
    var result = await _db.Person.Aggregate()
        // 为了和前边的例子保持一致,这里的查询条件我们就不通过参数传入了.
        .Match(_bf.Eq(c => c.Name, "张三") & _bf.Gt(c => c.Age, 40))
        .ToListAsync();
    return result;
}
```

- 可以发现 C# 真的是非常优雅,非常简单.接下来就是分组了.

```csharp
[HttpGet("Group")]
public async Task<dynamic> GetGroup()
{
    var result = await _db.Person.Aggregate()
        .Match(_bf.Gt(c => c.Age, 40))
        .Group(c => new { c.Gender }, g => new
        {
            g.Key,
            Count = g.Sum(c => 1)
        })
        .Project(c => new
        {
            性别 = c.Key.Gender,
            人数 = c.Count
        }).ToListAsync();
    return result;
}
```

- 其实第一次我用 C#的版本我也不知道如何写代码,是[参考了这个帖子](https://www.mongodb.com/community/forums/t/c-aggregation-group-by-string-and-select-last-by-date/9574)的内容
- https://www.mongodb.com/community/forums/t/c-aggregation-group-by-string-and-select-last-by-date/9574

- 接下来就写一下常见的分页查询.
- 由于分页信息往往是前端传入的,这里我为了简单,就将数据直接在程序中构造了.可以作为一个参考.

```csharp
[HttpPost("Page")]
public async Task<dynamic> Page()
{
    // 该方法往往需要传入一个对象,因此使用Post
    // 首先我们使用动态类型创建一个分页数据的对象.
    var post = new { Index = 4, Size = 3 };
    // 这里有多种写法,我先写个两三种.
    // 使用管道分页的话,顺序很重要,正确的分页执行顺序应该是
    // 过滤 -> 排序 -> 跳过数据 -> 拿数据 -> 返回
    var result1 = await _db.Person.Aggregate()
        .Match(_bf.Gt(c => c.Age, 30))
        .SortByDescending(c => c.Age).ThenBy(c => c.Gender)
        .Skip((post.Index - 1) * post.Size)
        .Limit(post.Size)
        .ToListAsync();

    var result2 = await _db.Person.Find(_bf.Gt(c => c.Age, 30))
        .Skip((post.Index - 1) * post.Size)
        .Limit(post.Size)
        .SortByDescending(c => c.Age).ThenBy(c => c.Gender)
        .ToListAsync();

    var result3 = await _db.Person.FindAsync(_bf.Gt(c => c.Age, 30),
        new FindOptions<Person, Person>
        {
            Skip = (post.Index - 1) * post.Size,
            Limit = post.Size,
            Sort = _bs.Descending(c => c.Age).Ascending(c => c.Gender)
        }).Result.ToListAsync();

    return new Tuple<dynamic, dynamic, dynamic>(result1, result2, result3);
}
```

- 以上就是 MongoDB 中常用的几种聚合管道的使用.学会这些基本上日常开发遇到的问题都可以解决了.
- 当遇到没有用过的聚合函数或者管道的时候,可以看看官网的详细介绍.基本上都是触类旁通,举一反三.
- 稍后我会将代码全部上传到 GitHub 中,有兴趣的小伙伴可以取下来做一下参考.
- [GitHub 源码地址](https://github.com/joesdu/MongoCRUD)
- https://github.com/joesdu/MongoCRUD
