今天不讲技术,讲一讲工具.对于会前端开发程序员来说前端工程化工作流中有个常用的工具 Husky,它方便我们在项目中添加 git hooks 在代码提交前自动检查编码规范,或对填写的 message 进行检查.对于大型团队来说这个工具可以确保每个开发人员都使用相同开发规范和工作流工作.但是在.NET 中却没有这样的工具,这是因为 VS 的智能提示解决了大部分问题,但也有一些问题 VS 无法解决的,并且 VS 只是给出建议并非强制规范,而且功能也有限,很难自定义.
基于这些原因[Husky.Net](https://github.com/alirezanet/Husky.Net)横空出世.

- 改善了你的提交，更多的 🐶 哇！
  安装了 Husky.Net 后,当我们提交.NET 项目代码时,就可以用它来做提交前检查,例如格式化代码,运行单元测试等等.下面我们首先来看看 Husky.Net 的特点:

Git 的 core.hooksPath 功能提供支持;

- 🔥 内部任务运行器！
- 🔥 多个文件状态(暂存,最后提交,全局)
- 🔥 兼容 [dotnet format](https://github.com/dotnet/format)
- 🔥 可自定义的任务
- 🔥 支持特定分支的任务
- 🔥C#脚本(csx)!🆕
- 🔥 支持 gitflow 钩子 🆕
- 支持所有 Git 钩子
- 由现代新 Git 功能(core.hooksPath)提供支持
- 用户友好的消息
- 支持 MacOS、Linux 和 Windows
- Git GUI
- 自定义目录
- Monorepo

Husky.Net 它支持两种安装方式，分别是全局安装和本地安装。方式如下：

- 全局安装:

```bash
dotnet tool install --global Husky
```

- 本地安装:

```bash
cd 项目根目录
dotnet new tool-manifest
dotnet tool install Husky
```

执行完上面的命令后就可以把 Husky 安装到项目中了,命令如下:

```bash
cd 项目根目录
dotnet husky install
```

接着我们添加 commit hook,例如我们添加一句话:

```bash
dotnet husky add .husky/pre-commit "dotnet husky run"
git add .husky/pre-commit
```

执行完后，每次我们提交代码就都会执行 dotnet format 命令(该命令仅支持.Net 6+)

接下来是重头戏,若是是团队开发,则可以将该工具添加到项目中任意项目,在打开项目的时候自动还原.例如:我这里工作文件夹为当前项目的 **'../../../'** 按需调整

```xml
<Target Name="husky" BeforeTargets="Restore;CollectPackageReferences">
	<Exec Command="dotnet tool restore"  StandardOutputImportance="Low" StandardErrorImportance="High"/>
	<Exec Command="dotnet husky install" StandardOutputImportance="Low" StandardErrorImportance="High"
             WorkingDirectory="../../../" />
	<!--Update this to the releative path to your project root dir -->
</Target>
```

当使用如上功能配置好后,即可在提交的时候看到 Husky 的输出内容.
然而,我们使用这个工具,肯定不是为了在提交代码的时候在控制台输出 Hello.而是为了他自动使用 dotnet format 命令格式化我们未格式化的代码.

- 先使用 Visual Studio 添加<.editorconfig>文件,给整个项目添加格式配置
- 再打开项目目录中的<.husky>目录,找到 task-runner.json 文件.
  原文如下:

```json
{
  "tasks": [
    {
      "command": "dotnet-format",
      "args": ["--include", "${staged}"],
      "include": ["**/*.cs", "**/*.vb"]
    },
    {
      "name": "Welcome",
      "output": "always",
      "command": "bash",
      "args": ["-c", "echo Husky.Net is awesome!"],
      "windows": {
        "command": "cmd",
        "args": ["/c", "echo Husky.Net is awesome!"]
      }
    }
  ]
}
```

然后我们会发现无论如何都无法正常格式化代码.那我们只好对该文件进行一些调整.给我们的指令添加一个名称:Formate Code,添加 Unix 系统中的命令为":bash,若是 Windows 安装了 PowerShell Core 的话,则可以在 windows.comman 中写入 pwsh,否则可以使用 powershell 或者 cmd

```json
{
  "tasks": [
    {
      "name": "Formate Code",
      "output": "always",
      "command": "bash",
      "args": ["dotnet format './*.sln'", "--include", "${staged}"],
      "include": ["**/*.cs", "**/*.vb"],
      "windows": {
        "command": "pwsh",
        "args": ["/c", "dotnet format './{你的项目资源管理名称}.sln'"]
      }
    },
    {
      "name": "Welcome",
      "output": "always",
      "command": "bash",
      "args": ["-c", "echo Husky.Net is awesome!"],
      "windows": {
        "command": "cmd",
        "args": ["/c", "echo Husky.Net is awesome!"]
      }
    }
  ]
}
```
