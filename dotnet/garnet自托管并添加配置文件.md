### 使用.NET 控制台应用程序,自托管 Garnet 服务

> Garnet 是 Microsoft Research 推出的一个新的远程缓存存储,旨在实现极快、可扩展和低延迟.Garnet 可在单个节点内进行线程扩展.它还支持分片集群执行,包括复制、检查点、故障转移和事务.它可以在主内存以及分层存储（例如 SSD 和 Azure 存储）上运行.Garnet 支持丰富的 API 图面和强大的扩展性模型.

更多的细节以及和 Redis 的性能对比,可以参考 InCerry 大佬的公众号文章.也可以通过我分享的转载进行查看.本文不再赘述之前的那么多东西了.这里主要介绍我们如何自定义自己的 Garnet 服务.

要使用 Garnet 服务,我们可以利用微软提供的镜像或者别的方式.但是有时候我们希望自己对其进行更多的配置和扩展.这个时候我们就可以利用 Nuget 包的加入到我们的程序来自托管我们的服务.

- 首先创建一个空白的控制台应用程序,为了减少截图,这里我是用命令行的方式.

```bash
# 创建解决方案文件
dotnet new sln -n GarnetServer
# 创建一个控制台应用程序
dotnet new console -n GarnetServer
# 将项目“GarnetServer\GarnetServer.csproj”添加到解决方案中
dotnet sln add ./GarnetServer
# 为控制台项目添加Microsoft.Garnet依赖包.
dotnet add ./GarnetServer/GarnetServer.csproj package Microsoft.Garnet
```

- 通过上面的命令,我们即可成功创建好项目.接下来使用 VS 或者 VS Code 打开项目.将 Program.cs 文件修改为如下内容.

```csharp
using Garnet;

// 重命名终端显示的名字
Console.Title = "GarnetServer";
try
{
    using var server = new GarnetServer(args);
    server.Start();
    Thread.Sleep(Timeout.Infinite);
}
catch (Exception ex)
{
    Console.WriteLine($"Unable to initialize server due to exception: {ex.Message}");
}
```

- 可以看到这里和我们在官方的文档中看到的内容一样,我们并不需要调整其中的代码.接下来我们在项目跟目录添加 garnet.conf 文件.
- 该文件将作为我们的配置文件,格式为 JSON 格式.当然也可以是其他的名字,但是为了规范我们采用和官方一样的名称.为了简单,这里我们仅使用一个配置项来作为例子.
- 这里有个小坑,由于配置文件是 JSON 格式,所以其中不能有注释,JSON 本身就不支持注释.添加注释会导致启动失败.

```json
{
  "Port": 8848
}
```

- 同时我们需要调整一下 GarnetServer.csproj 将其中 garnet.conf 相关的部分调整为更新则复制到输出目录.这是为了让该文件在发布后也能被复制到发布目录.

```xml
  <ItemGroup>
	  <None Update="garnet.conf">
		  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	  </None>
  </ItemGroup>
```

- 完成这些后我们便可以发布项目并通过命令行进行启动.

```bash
GarnetServer.exe --config-import-path ./garnet.conf
```

但是作为跨平台应用程序,我们肯定不希望仅运行在 Windows 端,所以我们还需要添加 Linux 相关的内容,同时每次都通过命令启动实在是有点不优雅.作为一个喜欢优雅的人来说,这不太符合我的风格,所以我打算为他添加一些启动脚本,这样我们不需要每次都用命令行启动.

**Windows 端**

在 Windows 上使用任何程序都非常简单,微软在用户操作的体验上是真的没什么可说的,并且作为微软力推的.NET 平台的应用更是没有话说.在 Windows 上我们可以利用批处理命令的方式进行程序的启动.所以我在项目根目录创建一个 start.bat 文件.使用 Notepad++打开后添加如下内容.

```bash
GarnetServer.exe --config-import-path ./garnet.conf
```

同样我们需要将这个文件的属性调整为更新的时候复制,方便发布后,该脚本也在发布的目录中.

```xml
  <ItemGroup>
	  <None Update="start.bat">
	    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	  </None>
  </ItemGroup>
```

**Linux 端**

在 Linux 端我们也可以通过 Linux 的脚本命令进行程序的启动,所以我们再在根目录创建一个名为 start.sh 的文件,同样使用 Notepad++打开编辑,写入如下内容:

```bash
#!/bin/bash

# 为GarnetServer添加可执行权限
sudo chmod +x ./GarnetServer
./GarnetServer --config-import-path ./garnet.conf
```

Linux 的脚本需要注意一下.需要利用 Notepad++将文件格式改为 Unix 格式,并且将字符编码调整为 UTF-8.
同样我们需要将这个文件的属性调整为更新的时候复制,方便发布后,该脚本也在发布的目录中.最终我们的 csproj 文件应该是如下内容.若是我们后期还需要添加其他东西,如 SSL 证书等,也可以通过这种方式.将文件复制进去.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <DockerDefaultTargetOS>Linux</DockerDefaultTargetOS>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Garnet" Version="1.0.0" />
    <!--这里的容器工具库,后面我们添加Docker支持的时候会自动加入-->
    <PackageReference Include="Microsoft.VisualStudio.Azure.Containers.Tools.Targets" Version="1.20.0-Preview.2" />
  </ItemGroup>

  <!--我们新增的三个文件,均需要在较新的时候复制到输出目录-->
  <ItemGroup>
	  <None Update="garnet.conf">
		  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	  </None>
	  <None Update="start.bat">
	    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	  </None>
	  <None Update="start.sh">
	    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	  </None>
  </ItemGroup>

</Project>
```

**Docker 支持**

在 VS 中我们可以很方便的对应用程序添加 Docker 支持.我们仅需在项目上点击鼠标右键,即可选择添加 Docker 支持.选择 Linux 和 Dockerfile 的选项即可.VS 会自动创建好我们的 Dockerfile,在 WebApi 的项目中,我们不需要做任何更改即可利用这个文件生成 docker 镜像,但是这里不行,因为我们的服务并没有导出端口.所以我们需要调整下 Dockerfile 的内容.

最终的 Dockerfile 的内容应该如下.

```dockerfile
#See https://aka.ms/customizecontainer to learn how to customize your debug container and how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM mcr.microsoft.com/dotnet/runtime:9.0-preview AS base
USER app
WORKDIR /app
# 导出端口,根据我们的配置项进行调整.
EXPOSE 8848

FROM mcr.microsoft.com/dotnet/sdk:9.0-preview AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["GarnetServer/GarnetServer.csproj", "GarnetServer/"]
RUN dotnet restore "./GarnetServer/GarnetServer.csproj"
COPY . .
WORKDIR "/src/GarnetServer"
RUN dotnet build "./GarnetServer.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./GarnetServer.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
# 调整启动命令,添加配置文件路径启动参数
ENTRYPOINT ["dotnet", "GarnetServer.dll", "--config-import-path", "./garnet.conf"]
```

然后我们即可使用 PowerShell 脚本利用 docker 来打包镜像.后面代码我会开源,这里暂时不对文件路径方面做出详细描述.

```powershell
# 这里是你的镜像仓库的配置,别和我一样.
$base="dygood/" ;
$name="garnet" ;

$tag=":latest" ;
$imagename =$base + $name + $tag ;
$imagename ;
docker build -f ./GarnetServer/Dockerfile -t $imagename . ;
```

最后可以使用推送脚本将镜像推送到我们自己的镜像仓库.

```powershell
$base="dygood/" ;
$name="garnet" ;

$tag=":latest" ;
$imagename =$base + $name + $tag ;
$imagename ;
docker push $imagename ;
```

到这里我们的 Linux,Windows 和 Docker 的打包发布就完成.接下来讲解下如何运行项目.

---

通过 VS 的 GUI 窗口我们可以很轻松的对不同平台进行打包发布.在项目上点击鼠标右键,一路点击发布到文件夹即可.可以独立框架发布也可以选依赖框架发布,这里不对这个东西做过多讲解,我用的独立发布.这样文件夹会小一点,稍后传到 Linux 电脑上方便点.

<details> 
<summary style="font-size: 14px">Windows</summary>

- 至于怎么发布.NET 程序我就不过多讲解了,不知道的建议 Bing 学习下,这是基础的内容.
- 进入发布目录中,双击 start.bat 文件即可启动服务.会产生如下输出内容.

```text
    _________
   /_||___||_\      Garnet 1.0.0 64 bit; standalone mode
   '. \   / .'      Port: 8848
     '.\ /.'        https://aka.ms/GetGarnet
       '.'

* Ready to accept connections
```

</details>

<details> 
<summary style="font-size: 14px">Linux</summary>

在 Linux 中由于权限的问题,所以相对来说还需要对可执行文件进行授权.

首先将发布后的文件上传到我们的 Linux 操作系统中.

```bash
# 进入目录
cd ./linux-x64
# 授予start.sh文件的可执行权限.
sudo chmod +x ./start.sh
# 接下来即可运行脚本.
./start.sh
```

- 会得到和 Windows 一样的输出内容.
- 在 Linux 中我们还可以将该命令的路径添加到环境变量中,即可在任意位置运行程序.也可以用 systemd 的方式进行服务的托管.

</details>

<details>
<summary style="font-size: 14px">Docker</summary>

Docker 的话,这里也是很简单通过一行命令启动即可.

```bash
docker run --name garnet -p 8848:8848 -d --rm -it dygood/garnet
```

</details>

至此本文所有内容完结,至于其他的集群配置,以及更多的配置内容可以查看微软官方的配置文件.来进行调整.最后贴上本文中代码地址.

Github 仓库: https://github.com/joesdu/GarnetServer
