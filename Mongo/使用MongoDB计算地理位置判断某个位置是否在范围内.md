在日常开发中可能会遇到定位考勤等问题.
但是使用圆形的范围计算不太精确.这个时候我们就可以使用 Mongodb 的地理位置计算功能了.
这里我们使用自己调整过的[Miracle.MongoDB](https://github.com/GardeningX/Miracle.MongoDB),也可以使用 Nuget 安装.

- 我们先定义一个 List 来存放范围信息

```csharp
using Miracle.Common;
namespace Range.Test;
public class AttendanceRange
{
    public string Id { get; set; }
    public string Title { get; set; }
    public OperatorItem Creator { get; set; }
    public List<double[]> Locations { get; set; }
}
```

- 然后向数据库中存入地理范围信息.

```csharp
public async Task<AttendanceRange> Post(AttendanceRangePost dto)
{
    var user = HttpContext.GetLoginUserFromToken();
    var obj = dto.GetMapClass();
    if (obj.Locations.Count < 3) throw new("无法构成范围信息");
    var first = obj.Locations.First();
    var last = obj.Locations.Last();
    if (Math.Abs(first[0] - last[0]) > 0 | Math.Abs(first[1] - last[1]) > 0) throw new("无法构成闭合位置的范围");
    obj.Creator = user.ToOperator();
    await Coll.InsertOneAsync(obj);
    return obj;
}
```

- 接下来写几个函数来计算传入的位置是否在范围内

```csharp
private readonly FilterDefinitionBuilder<Location> _bf = Builders<Location>.Filter;
private async Task<bool> InRange(PointDro l)
{
    var o = new Location() { Point = l };
    var s = await db._client?.StartSessionAsync()!;
    s.StartTransaction();
    try
    {
        await db.Location.InsertOneAsync(o);
        var f = await InRange(s, o.Id);
        _ = db.Location.DeleteOneAsync(s, c => c.Id == o.Id);
        _ = s.CommitTransactionAsync();
        return f;
    }
    catch
    {
        _ = s.AbortTransactionAsync();
        throw;
    }
}
private async Task<bool> InRange(IClientSessionHandle session, string id)
{
    var rs = await db.AttendanceRange.Find(session, x => true).Project(x => x.Locations).ToListAsync();
    if (rs is not null && rs.Count <= 0) throw new("无范围信息");
    var f = false;
    if (rs is null) return false;
    {
        foreach (var r in rs)
        {
            var a = new double[r.Count, 2];
            for (var i = 0; i < r.Count; i++)
            {
                a[i, 0] = r[i][0];
                a[i, 1] = r[i][1];
            }
            var c = await db.Location.Find(session, _bf.Eq(x => x.Id, id) & _bf.GeoWithinPolygon(x => x.Point, a)).CountDocumentsAsync();
            if (c <= 0) continue;
            f = true;
            break;
        }
    }
    return f;
}
```

- 调用 InRange(PointDro l)传入需要验证的位置点就可以计算是否在范围内.
