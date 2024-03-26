#### LINQ 3 数组操作(2)

书接上回,上次还有一个函数的用法当时没有摸清楚,经过两天的摸索和测试.目前用法已经比较明朗了.所以补一下内容.

AllMatchingElements,该方法需要配合 UpdateOptions 的 ArrayFilters 参数来进行筛选匹配的元素.

ArrayFilters 的参数类型为 `IEnumerable<ArrayFilterDefinition>` 所以我们可以传递一个集合来实现多个匹配条件.在 MongoDB 的驱动中,实现了 `BsonDocumentArrayFilterDefinition` 和 `JsonArrayFilterDefinition` 所以这里我们通过这两种类型来创建两种实现.

<details> 
<summary style="font-size: 14px">BsonDocumentArrayFilterDefinition</summary>

使用 `BsonDocumentArrayFilterDefinition` 可以较为直观的创建查询条件,而不需要进行复杂的字符串的拼接,可以减少一些错误.这里直接给出我的代码,然后再逐一讲解.

```csharp
[HttpPut("UpdateOne")]
public async Task UpdateOneElement()
{
    // 这里我们举得例子是将哆啦美的名字变更为日文名字.
    // 这里我们假设查询参数同样是通过参数传入的,所以我们写出了如下代码.
    //await db.FamilyInfo.UpdateOneAsync(c => (c.Name == "野比家") & c.Members.Any(s => s.Index == 4),
    //    _bu.Set(c => c.Members.FirstMatchingElement().Name, "ドラミ"));
    await db.FamilyInfo.UpdateOneAsync(_bf.Eq(c => c.Name, "蜡笔小新"),
       _bu.Inc(c => c.Members.AllMatchingElements("ele").Age, 100), new()
       {
           ArrayFilters =
           [
               // 第一个条件为数组中的所有女性
               //new BsonDocumentArrayFilterDefinition<BsonDocument>(new($"ele.{nameof(Person.Gender).ToLowerCamelCase()}", EGender.女.ToString()))
               // 第二个条件为所有年纪小于1000的元素
               new BsonDocumentArrayFilterDefinition<BsonDocument>(new($"ele.{nameof(Person.Age).ToLowerCamelCase()}", new BsonDocument("$lt", 1000)))
               // 这里面还可以加入其他的过滤条件.(这里数据较为简单,不太好举例)
           ]
       });
}
```

这个例子中,我新增了几个数据,方便测试,数据结构如下(其中年纪不用在意,测试的时候加上去的.):

```json
{
    "_id": ObjectId("6600f371d11d2c7ed09067ac"),
    "name": "蜡笔小新",
    "members": [
        {
            "_id": ObjectId("6600f371d11d2c7ed09067a2"),
            "index": NumberInt("0"),
            "name": "野原广志",
            "age": NumberInt("535"),
            "gender": "男",
            "birthday": "1963-09-27"
        },
        {
            "_id": ObjectId("6600f371d11d2c7ed09067a3"),
            "index": NumberInt("1"),
            "name": "野原美冴",
            "age": NumberInt("1929"),
            "gender": "女",
            "birthday": "1969-10-10"
        },
        {
            "_id": ObjectId("6600f371d11d2c7ed09067a4"),
            "index": NumberInt("2"),
            "name": "野原新之助",
            "age": NumberInt("505"),
            "gender": "男",
            "birthday": "1994-07-22"
        },
        {
            "_id": ObjectId("6600f371d11d2c7ed09067a5"),
            "index": NumberInt("3"),
            "name": "野原向日葵",
            "age": NumberInt("1901"),
            "gender": "女",
            "birthday": "1998-09-27"
        }
    ]
}
```

通过上面 UpdateOneElement 中的查询条件,我们的数据库会先定位到 name 为 `蜡笔小新` 的数据,然后更新 members 数组中的所有年龄小于 1000 的数据.给他们每个加 100,第一个条件的作用可以查看注释的内容.

</details>
 
当然为了减少魔法字符串的使用.比如 `$"ele.{nameof(Person.Age).ToLowerCamelCase()}"` 中的age字段对应数据库中的age.所以我实现了几个函数,用于字段名称和数据库字段名称的匹配.下面也贴出代码.
```csharp
public static class StringExtensions
{
    /// <summary>
    /// 将小驼峰命名转为大驼峰命名,如: FirstName
    /// </summary>
    public static string ToUpperCamelCase(this string value) =>
        // 将第一个字母大写并与其余字符串连接
        string.IsNullOrEmpty(value) ? value : $"{char.ToUpperInvariant(value[0])}{value[1..]}";
    
    /// <summary>
    /// 将大驼峰命名转为小驼峰命名,如: firstName
    /// </summary>
    public static string ToLowerCamelCase(this string value)
    {
        var firstChar = value[0];
        if (firstChar == char.ToLowerInvariant(firstChar)) return value;
        var name = value.ToCharArray();
        name[0] = char.ToLowerInvariant(firstChar);
        return new(name);
    }
    
    /// <summary>
    /// 将大(小)驼峰命名转为蛇形命名,如: first_name
    /// </summary>
    public static string ToSnakeCase(this string value)
    {
        var builder = new StringBuilder();
        var previousUpper = false;
        for (var i = 0; i < value.Length; i++)
        {
            var c = value[i];
            if (char.IsUpper(c))
            {
                if (i > 0 && !previousUpper)
                {
                    builder.Append('_');
                }
                builder.Append(char.ToLowerInvariant(c));
                previousUpper = true;
            }
            else
            {
                builder.Append(c);
                previousUpper = false;
            }
        }
        return builder.ToString();
    }
    
    /// <summary>
    /// 将蛇形命名转化成驼峰命名
    /// </summary>
    /// <param name="value"></param>
    /// <param name="toType">目标方式,默认:小驼峰</param>
    /// <returns></returns>
    public static string SnakeCaseToCamelCase(this string value, ECamelCase toType = ECamelCase.LowerCamelCase)
    {
        if (string.IsNullOrEmpty(value)) return value;
        // 分割字符串，然后将每个单词(除了第一个)的首字母大写
        var words = value.Split('_');
        int index;
        if (toType is ECamelCase.LowerCamelCase)
        {
            // 第一个单词首字母小写
            words[0] = $"{char.ToLowerInvariant(words[0][0])}{words[0][1..]}";
            index = 1;
        }
        else
        {
            index = 0;
        }
        for (; index < words.Length; index++)
        {
            if (words[index].Length > 0)
            {
                words[index] = $"{char.ToUpperInvariant(words[index][0])}{words[index][1..]}";
            }
        }
        return string.Join("", words);
    }
}
```

<details> 
<summary style="font-size: 14px">JsonArrayFilterDefinition</summary>

使用 `JsonArrayFilterDefinition` 就更直接了,我们只需要直接传入一个 JSON 格式的字符串就行,这里也直接贴出代码.

```csharp
[HttpPut("UpdateOne")]
public async Task UpdateOneElement()
{
    // 这里我们举得例子是将哆啦美的名字变更为日文名字.
    // 这里我们假设查询参数同样是通过参数传入的,所以我们写出了如下代码.
    //await db.FamilyInfo.UpdateOneAsync(c => (c.Name == "野比家") & c.Members.Any(s => s.Index == 4),
    //    _bu.Set(c => c.Members.FirstMatchingElement().Name, "ドラミ"));
    await db.FamilyInfo.UpdateOneAsync(_bf.Eq(c => c.Name, "蜡笔小新"),
        _bu.Inc(c => c.Members.AllMatchingElements("ele").Age, 100), new()
        {
            ArrayFilters =
            [
                new JsonArrayFilterDefinition<BsonDocument>(new($$"""
                                                                  {
                                                                    "ele.{{nameof(Person.Age).ToLowerCamelCase()}}" : {
                                                                      "$lt" : {{1000}}
                                                                    }
                                                                  }
                                                                  """))
            ]
        });
}
```

最终生成的查询条件就是直接将 JSON 字符串作为筛选条件传递给数据库.

</details>

这便是最后一种 `AllMatchingElements` 的用法,虽然也有大佬给我别的示例,但是我发现他仅仅能应对简单的条件,若是类似我这种在 ArrayFilters 里添加查询指令 `$lt` 这种就无法支持了,所以为了通用性,还是使用官方提供的 API 来操作.

并且可以发现我即使是用了字符串,还是尽量的避免直接将代码写死,比如 `ele.{{nameof(Person.Age).ToLowerCamelCase()}}` 而不是直接写成 `ele.age` ,是为了避免魔法字符串后期在维护和发生变化后产生的问题.为后期的更新带来未知问题,因为使用这种方式,当属性变化的时候 IDE 可以感知到.进而全局同步更改.这个东西在日常编程中也应该尽量避免.

所有代码已经同步到 GitHub,可以通过链接访问: https://github.com/joesdu/MongoCRUD
