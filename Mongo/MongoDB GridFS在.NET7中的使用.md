## MongoDB GridFS 在.NET 7 中的使用

- 突然忆起 MongoDB 中还有个很大的功能.那就是 MongoDB GridFS.

  > GridFS 是 MongoDB 的一个子模块,使用 GridFS 可以基于 MongoDB 来持久存储文件,并且支持分布式应用(文件分布存储和读取).GridFS 是 MongoDB 提供的二进制数据存储在数据库中的解决方案,对于 MongoDB 的 BSON 格式的数据(文档)存储有尺寸限制,最大为 16M.但是在实际系统开发中,上传的图片或者文件可能尺寸会很大,此时我们可以借用 GridFS 来管理这些文件.

- 常用的使用场景及原理概念

  1. GridFS 用于存储和恢复那些超过 16M(BSON 文件限制)的文件(如:图片、音频、视频等).
  1. GridFS 也是文件存储的一种方式,但是它是存储在 MonoDB 的集合中.
  1. GridFS 可以更好的存储大于 16M 的文件.
  1. GridFS 会将大文件对象分割成多个小的 chunk(文件片段或块),一般为 256k/个,每个 chunk 将作为 MongoDB 的一个文档(document)被存储在 chunks 集合中.
  1. GridFS 用两个集合来存储一个文件:fs.files 与 fs.chunks. 1.每个文件的实际内容被存在 chunks(二进制数据)中,和文件有关的 meta 数据(filename,content_type,还有用户自定义的属性)将会被存在 files 集合中.

- 其中 fs.chunks 的数据结构如下:

```json
{
  "_id" : <ObjectId>,
  "files_id" : <ObjectId>,
  "n" : <num>,
  "data" : <binary>
}
```

- 各个字段的含义如下

|   字段   | 描述                                            | 英文原文                                                                      |
| :------: | ----------------------------------------------- | ----------------------------------------------------------------------------- |
|   \_id   | 该块的唯一 ObjectId                             | The unique ObjectId of the chunk.                                             |
| files_id | 文档集合中指定的 "父 "文档的\_ID.               | The \_id of the "parent" document, as specified in the files collection.      |
|    n     | 该块的序列号.GridFS 对所有块进行编号,从 0 开始. | The sequence number of the chunk. GridFS numbers all chunks, starting with 0. |
|   data   | 该块的有效载荷为 BSON 二进制类型                | The chunk's payload as a BSON Binary type.                                    |

- 其中 fs.files 的数据结构如下:

```json
{
  "_id" : <ObjectId>,
  "length" : <num>,
  "chunkSize" : <num>,
  "uploadDate" : <timestamp>,
  "md5" : <hash>,
  "filename" : <string>,
  "contentType" : <string>,
  "aliases" : <string array>,
  "metadata" : <any>,
}
```

- 各个字段的含义如下

|    字段    | 描述                                                                                                                                                  | 英文原文                                                                                                                                                                                                                                          |
| :--------: | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|    \_id    | 这个文档的唯一标识符.\_id 是你为原始文档选择的数据类型.MongoDB 文档的默认类型是 BSON ObjectId.                                                        | The unique identifier for this document. The \_id is of the data type you chose for the original document. The default type for MongoDB documents is BSON ObjectId.                                                                               |
|   length   | 文件的大小,以字节为单位.                                                                                                                              | The size of the document in bytes.                                                                                                                                                                                                                |
| chunkSize  | 每个块的大小,以字节为单位.GridFS 将文件分成大小为 chunkSize 的块,除了最后一个,它只按需要大小.默认大小为 255 千字节（kB）.                             | The size of each chunk in bytes. GridFS divides the document into chunks of size chunkSize, except for the last, which is only as large as needed. The default size is 255 kilobytes (kB).                                                        |
| uploadDate | 文件首次被 GridFS 存储的日期.该值具有日期类型.                                                                                                        | The date the document was first stored by GridFS. This value has the Date type.                                                                                                                                                                   |
|  metadata  | 可选的.元数据字段可以是任何数据类型,可以容纳任何你想存储的额外信息.如果你想给文件集合中的文件添加额外的任意字段,请将它们添加到元数据字段的一个对象中. | Optional. The metadata field may be of any data type and can hold any additional information you want to store. If you wish to add additional arbitrary fields to documents in the files collection, add them to an object in the metadata field. |

- 其中的 md5,filename,contentType 和 aliases 已经弃用,不知道什么时候会移除.所以这部分我就不介绍了,看名知其意.

### GridFS 索引

- GridFS 在每个块和文件集合上使用索引以提高效率.驱动程序 符合 规范的驱动程序 的驱动程序会自动创建这些索引以方便使用.你也可以根据需要创建任何额外的索引,以满足你的应用程序的需要.
- 由于 chunks 和 files 的索引驱动会自己创建,所以我这里就不啰嗦的写那么多了.
- 在 Mongo Shell 中使用 mongofiles 工具来管理 GridFS.(这个命令我倒是不怎熟悉,从官网的文档来写这部分代码)
- mongofiles 命令存在很多参数,我们可以使用如下命令来获取帮助文档.

```shell
mongofiles --help
```

- 他会输出如下信息

```text
Usage:
  mongofiles <options> <connection-string> <command> <filename or _id>

Manipulate gridfs files using the command line.

Connection strings must begin with mongodb:// or mongodb+srv://.

Possible commands include:
        list      - list all files; 'filename' is an optional prefix which listed filenames must begin with
        search    - search all files; 'filename' is a regex which listed filenames must match
        put       - add files with filenames specified in the supporting arguments
        put_id    - add a file with filename 'filename' and a given '_id'
        get       - get files with filenames specified in the supporting arguments
        get_id    - get a file with the given '_id'
        get_regex - get files matching the supplied 'regex'
        delete    - delete all files with filename 'filename'
        delete_id - delete a file with the given '_id'

See http://docs.mongodb.com/database-tools/mongofiles/ for more information.

general options:
      --help                                      print usage
      --version                                   print the tool version and exit
      --config=                                   path to a configuration file

verbosity options:
  -v, --verbose=<level>                           more detailed log output (include multiple times for more verbosity, e.g. -vvvvv, or specify a numeric value, e.g. --verbose=N)
      --quiet                                     hide all log output

connection options:
  -h, --host=<hostname>                           mongodb host to connect to (setname/host1,host2 for replica sets)
      --port=<port>                               server port (can also use --host hostname:port)

ssl options:
      --ssl                                       connect to a mongod or mongos that has ssl enabled
      --sslCAFile=<filename>                      the .pem file containing the root certificate chain from the certificate authority
      --sslPEMKeyFile=<filename>                  the .pem file containing the certificate and key
      --sslPEMKeyPassword=<password>              the password to decrypt the sslPEMKeyFile, if necessary
      --sslCRLFile=<filename>                     the .pem file containing the certificate revocation list
      --sslFIPSMode                               use FIPS mode of the installed openssl library
      --tlsInsecure                               bypass the validation for server's certificate chain and host name

authentication options:
  -u, --username=<username>                       username for authentication
  -p, --password=<password>                       password for authentication
      --authenticationDatabase=<database-name>    database that holds the user's credentials
      --authenticationMechanism=<mechanism>       authentication mechanism to use
      --awsSessionToken=<aws-session-token>       session token to authenticate via AWS IAM

kerberos options:
      --gssapiServiceName=<service-name>          service name to use when authenticating using GSSAPI/Kerberos (default: mongodb)
      --gssapiHostName=<host-name>                hostname to use when authenticating using GSSAPI/Kerberos (default: <remote server's address>)

uri options:
      --uri=mongodb-uri                           mongodb uri connection string

storage options:
  -d, --db=<database-name>                        database to use
  -l, --local=<filename>                          local filename for put|get
  -t, --type=                                     content/MIME type for put (optional)
  -r, --replace                                   remove other files with same name after put
      --prefix=<prefix>                           GridFS prefix to use
      --writeConcern=<write-concern>              write concern options e.g. --writeConcern majority, --writeConcern '{w: 3, wtimeout: 500, fsync: true, j: true}'
      --regexOptions=<regex-options>              regex options used for get_regex

query options:
      --readPreference=<string>|<json>            specify either a preference mode (e.g. 'nearest') or a preference json object (e.g. '{mode: "nearest", tagSets: [{a: "b"}], maxStalenessSeconds: 123}')
```

- 从帮助文档中我们可以看出来,它支持非常多种类的参数.这里就简单的介绍下他的几个命令,毕竟我们今天的主题不是命令行如何操作.
- 因为实际使用过程中我们除非特殊情况,肯定也不可能全程命令行去管理 GridFS.
- 推送文件到 GridFS 简要语法.
  - ```bash
      # mongofiles -h IP --port 端口号 -u 用户名 -p 密码 --db 数据库名 put 文件路径
      # eg:
      mongofiles -h 127.0.0.1 --port 27017 -u joe -p a123456 --db 'hoyo.files' put /home/download/Tom and Jerry.mp4
    ```
- 下载文件到本地指定目录.
  - ```bash
      # mongofiles -h IP --port 端口号 -u 用户名 -p 密码 --db 数据库名 get 保存路径
      mongofiles -h 127.0.0.1 --port 27017 -u joe -p a123456 --db 'hoyo.files' get /home/download/Tom and Jerry2.mp4
    ```
- 查看所有文件列表.
  - ```bash
      # mongofiles -h -u -p --db list
      # 参数就不详细描述了,和上边差不多的.
      mongofiles -h 127.0.0.1 --port 27017 -u joe -p a123456 --db 'hoyo.files' list
    ```
- 删除文件.
  - ```bash
      # mongofiles -h -u -p --db delete 文件名
      mongofiles -h 127.0.0.1 --port 27017 -u joe -p a123456 --db 'hoyo.files' delete 'Tom and Jerry.mp4'
    ```
- 上述的命令都还没有测试过,一般直接传文件的话,我会使用 Navicat 图形化工具,简单直接.这里我也就不再验证了,应该是 OK 的.

### 那么如何在.NET 7 中使用呢?

- 由于我们正常开发者使用 GridFS 的话肯定是在程序中使用,而不会使用管理工具或者命令行去操作.所以接下来介绍如何在.NET 7 中使用.
- 首先我们需要明确一些东西,由于在 GridFS 中管理文件的一些信息不是很方便.比如分页查询文件列表,以及一些业务属性的数据.如上传者.所属子系统等内容.虽然我们可以利用 metadata 来实现这些信息的记录,但是我觉得还是不够方便,不利于管理.所以我这里对这个功能做一些变化和改动.通过一个普通的集合来实现.
- 这个功能在 Hoyo.Mongo.GridFS 或 Hoyo.Mongo.GridFS.Extension(包含 Hoyo.Mongo.GridFS,添加了虚拟目录功能,用来做文件的下载缓存) 包中有提供,这里就不写这方面的东西了.
- 我们还是使用之前的 [MongoCRUD 项目](https://github.com/joesdu/MongoCRUD))来从 0 实现 GridFS 的使用.
- GitHub 地址: https://github.com/joesdu/MongoCRUD
- 首先在 Nuget 包管理工具中添加 MongoDB.Driver.GridFS 驱动包
- 为了便于注册服务,这里我们写一个扩展类. GridFSExtensions.cs

```csharp
/// <summary>
/// 服务注册于配置扩展
/// </summary>
public static class GridFSExtensions
{
    /// <summary>
    /// 注册GridFS服务
    /// </summary>
    /// <param name="services"></param>
    /// <param name="db">IMongoDatabase,为空情况下使用默认数据库hoyofs</param>
    /// <returns></returns>
    public static void AddHoyoGridFS(this IServiceCollection services, IMongoDatabase? db = null)
    {
        var client = services.BuildServiceProvider().GetService<IMongoClient>();
        if (db is null & client is null)
        {
           throw new("无法从容器中获取IMongoClient的服务依赖,请考虑显示传入db参数.");
        }
        var hoyo = client!.GetDatabase("hoyofs");
        _ = services.Configure<FormOptions>(c =>
        {
            c.MultipartBodyLengthLimit = long.MaxValue;
            c.ValueLengthLimit = int.MaxValue;
        }).Configure<KestrelServerOptions>(c => c.Limits.MaxRequestBodySize = int.MaxValue).AddSingleton(new GridFSBucket(hoyo));
    }
}
```

- 并且在 Program.cs 中调用该方法.这样,我们后期就可以在别的地方直接注入 GridFSBucket 类了.

```csharp
#region MongoDB服务注入

var dbStr = builder.Configuration.GetConnectionString("Mongo");
builder.Services.AddMongoDbContext<DbContext>(dbStr!);

#endregion

#region GridFS服务注入

// 由于我们BaseDbContext中已经注入了IMongoClient所以这里不需要显示传入db参数.否则需要显示传入.
builder.Services.AddHoyoGridFS();

#endregion
```

- 接下来创建一个控制器,来实现我们的文件上传,下载和删除,以及修改名称.
- 首先创建一个控制器 GridFSController.cs 并写入构造函数和注入我们需要的服务.

```csharp
[Route("api/[controller]"), ApiController]
public class GridFSController : ControllerBase
{
    /// <summary>
    /// GridFSBucket
    /// </summary>
    private readonly GridFSBucket Bucket;

    public GridFSController(GridFSBucket bucket)
    {
        Bucket = bucket;
    }
    private readonly FilterDefinitionBuilder<GridFSFileInfo> gbf = Builders<GridFSFileInfo>.Filter;
}
```

- 接下来先写两个新增功能,一个上传单文件,一个上传多文件.

```csharp
/// <summary>
/// 添加一个文件
/// </summary>
[HttpPost("UploadSingle")]
public async Task<string> PostSingle([FromForm] IFormFile fs)
{
    if (fs is null) throw new("no files find");
    if (fs.ContentType is null) throw new("ContentType in File is null");
    var metadata = new Dictionary<string, object> { { "contentType", fs.ContentType } };
    var upo = new GridFSUploadOptions
    {
        BatchSize = 1,
        Metadata = new(metadata)
    };
    var oid = await Bucket.UploadFromStreamAsync(fs.FileName, fs.OpenReadStream(), upo);
    return oid.ToString();
}

/// <summary>
/// 添加一个或多个文件
/// </summary>
[HttpPost("UploadMulti")]
public async Task<IEnumerable<string>> PostMulti([FromForm] IFormFileCollection fs)
{
    if (fs is null || fs.Count == 0) throw new("no files find");
    var result = new List<string>();
    foreach (var item in fs)
    {
        if (item.ContentType is null) throw new("ContentType in File is null");
        var metadata = new Dictionary<string, object> { { "contentType", item.ContentType } };
        var upo = new GridFSUploadOptions
        {
            BatchSize = fs.Count,
            Metadata = new(metadata)
        };
        var oid = await Bucket.UploadFromStreamAsync(item.FileName, item.OpenReadStream(), upo);
        result.Add(oid.ToString());
    }
    return result;
}
```

- 接下来是获取文件信息.(这里介绍通过 ID 或者文件名的方式.)

```csharp
/// <summary>
/// 下载
/// </summary>
/// <param name="id">文件ID</param>
/// <returns></returns>
[HttpGet("Download/{id}")]
public async Task<FileStreamResult> Download(string id)
{
    var stream = await Bucket.OpenDownloadStreamAsync(ObjectId.Parse(id), new() { Seekable = true });
    return File(stream, stream.FileInfo.Metadata["contentType"].AsString, stream.FileInfo.Filename);
}
/// <summary>
/// 通过文件名下载
/// </summary>
/// <param name="name"></param>
/// <returns></returns>
[HttpGet("DownloadByName/{name}")]
public async Task<FileStreamResult> DownloadByName(string name)
{
    var stream = await Bucket.OpenDownloadStreamByNameAsync(name, new() { Seekable = true });
    return File(stream, stream.FileInfo.Metadata["contentType"].AsString, stream.FileInfo.Filename);
}
/// <summary>
/// 打开文件内容
/// </summary>
/// <param name="id">文件ID</param>
/// <returns></returns>
[HttpGet("FileContent/{id}")]
public async Task<FileContentResult> FileContent(string id)
{
    var fi = await (await Bucket.FindAsync(gbf.Eq(c => c.Id, ObjectId.Parse(id)))).SingleOrDefaultAsync() ?? throw new("no data find");
    var bytes = await Bucket.DownloadAsBytesAsync(ObjectId.Parse(id), new GridFSDownloadOptions() { Seekable = true });
    return File(bytes, fi.Metadata["contentType"].AsString, fi.Filename);
}
/// <summary>
/// 通过文件名打开文件
/// </summary>
/// <param name="name"></param>
/// <returns></returns>
[HttpGet("FileContentByName/{name}")]
public async Task<FileContentResult> FileContentByName(string name)
{
    var fi = await (await Bucket.FindAsync(gbf.Eq(c => c.Filename, name))).FirstOrDefaultAsync() ?? throw new("can't find this file");
    var bytes = await Bucket.DownloadAsBytesByNameAsync(name, new() { Seekable = true });
    return File(bytes, fi.Metadata["contentType"].AsString, fi.Filename);
}
```

- 接下来是重命名文件.

```csharp
/// <summary>
/// 重命名文件
/// </summary>
/// <param name="id">文件ID</param>
/// <param name="newName">新名称</param>
/// <returns></returns>
[HttpPut("{id}/Rename/{newName}")]
public async Task Rename(string id, string newName)
{
    await Bucket.RenameAsync(ObjectId.Parse(id), newName);
}
```

- 再下来就是删除一个或者多个文件.

```csharp
/// <summary>
/// 删除文件
/// </summary>
/// <param name="ids">文件ID集合</param>
/// <returns></returns>
[HttpDelete]
public async Task<IEnumerable<string>> Delete(params string[] ids)
{
    var oids=ids.Select(ObjectId.Parse).ToList();
    var fi = await (await Bucket.FindAsync(gbf.In(c=>c.Id, oids))).ToListAsync();
    var fids = fi.Select(c => new
    {
        Id = c.Id.ToString(),
        FileName = c.Filename
    }).ToArray();
    Task DeleteSingleFile()
    {
        foreach (var item in fids)
        {
            _ = Bucket.DeleteAsync(ObjectId.Parse(item.Id));
        }
        return Task.CompletedTask;
    }
    _ = fids.Length > 6 ? Task.Run(DeleteSingleFile) : DeleteSingleFile();
    return fids.Select(c => c.FileName);
}
```

- 以上内容就是 MongoDB 中 GridFS 的简单使用了.
- 可以发现 GridFS 若是想实现文件列表的查询就不行了,或者说很麻烦,我没找到 C# 的方案.
- 所以可以查看我的开源库 Hoyo.Mongo.GridFS,其中就通过新增一个集合的方式管理了,文件列表信息.这样我们在提供文件列表的时候就可以直接查询这个集合中的数据来查看.
- 同时还提供了 Hoyo.Mongo.GridFS.Extensions,他在 Hoyo.Mongo.GridFS 的基础上,添加了虚拟文件目录,可以使用一个文件夹做缓存目录,将文件以 Url 的方式返回给用户.方便用户在线查看和下载.
- 上述的两个开源包的[GitHub 仓库地址](https://github.com/joesdu/Hoyo)在这里,感兴趣的可以关注下,或者去看看其中的文件列表实现方案.
- https://github.com/joesdu/Hoyo
- 顺路祝大家 23 年新年快乐.
