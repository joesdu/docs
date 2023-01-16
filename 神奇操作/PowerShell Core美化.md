#### 前置条件

安装[PowerShell Core](https://docs.microsoft.com/zh-cn/powershell/scripting/install/installing-powershell)和[VS Code](https://code.visualstudio.com/insiders/#)

## 安装[oh-my-posh](https://ohmyposh.dev/docs/) [posh-git](https://github.com/dahlbyk/posh-git)

以管理员身份打开 PowerShell, 执行

```powershell
Install-Module oh-my-posh
Install-Module posh-git
Install-Module -Name PSReadLine -AllowPrerelease -Force
```

这里之所以要安装预览版 PSReadLine 是因为后续的 PredictionViewStyle 选项是 2.2 之后才支持的，而目前稳定版是 2.1。 powershell core 默认内置稳定版 PSReadLine, 等下个版本应该就不用手动安装 PSReadLine 了

配置 profile
执行 `$profile` 然后会显示 profile 文件所在路径
一般是`%UserProfile%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`
执行`code $profile`或者`code-insiders $profile`用 vsc 打开, 如果没有就会自动创建
内容如下:
设置为 material 主题

```powershell
Import-Module posh-git
Import-Module oh-my-posh

Set-PoshPrompt -Theme material

Register-ArgumentCompleter -Native -CommandName dotnet -ScriptBlock {
    param($commandName, $wordToComplete, $cursorPosition)
        dotnet complete --position $cursorPosition "$wordToComplete" | ForEach-Object {
            [CompletionResult]::new($_, $_, 'ParameterValue', $_)
        }
}
Set-PSReadLineKeyHandler -Key "Ctrl+f" -Function ForwardWord
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
Set-PSReadLineKeyHandler -Key "Ctrl+z" -Function Undo
Set-PSReadLineOption -PredictionSource History -PredictionViewStyle ListView
```

可以通过命令`Get-PoshThemes`获取主题列表, 然后替换里边的 material 主题即可

效果如下，按上下键切换 自动提示列表来源于历史记录
![](images/b27a36c0.webp)
