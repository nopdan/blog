# Scoop | windows上的包管理神器


Scoop 和 chocolatey 一样，是 windows 上的包管理软件。  
它不需要管理员权限就能安装软件到用户目录，用户目录和全局目录都可以自定义。  
下面是我的安装目录

<!-- more -->

![1][]

## 安装 Scoop

进入[**官网**](https://scoop.sh/)可以看到 Scoop 的安装非常简单  
需要 powershell 5 和 .NET Framework 4.5 及以上版本  
输入以下命令

```powershell
Invoke-Expression (New-Object System.Net.WebClient).DownloadString('https://get.scoop.sh')
# 或者 短命令
iwr -useb get.scoop.sh | iex

# (如果上述命令报错，先运行下面这行命令)
Set-ExecutionPolicy RemoteSigned -scope CurrentUser
```

运行 `scoop help` 检查是否安装成功

```
PS C:\Users\Elio> scoop help
Usage: scoop <command> [<args>]

Some useful commands are:

alias       Manage scoop aliases
bucket      Manage Scoop buckets
cache       Show or clear the download cache
checkup     Check for potential problems
cleanup     Cleanup apps by removing old versions
config      Get or set configuration values
create      Create a custom app manifest
depends     List dependencies for an app
export      Exports (an importable) list of installed apps
help        Show help for a command
hold        Hold an app to disable updates
home        Opens the app homepage
info        Display information about an app
install     Install apps
list        List installed apps
prefix      Returns the path to the specified app
reset       Reset an app to resolve conflicts
search      Search available apps
status      Show status and check for new app versions
unhold      Unhold an app to enable updates
uninstall   Uninstall an app
update      Update apps, or Scoop itself
virustotal  Look for app's hash on virustotal.com
which       Locate a shim/executable (similar to 'which' on Linux)

Type 'scoop help <command>' to get help for a specific command.
```

## 安装目录

Scoop 软件默认安装到 `C:\Users\<用户名>\scoop\` 下  
全局默认安装到 `C:\ProgramData\scoop\` 下，通过 shim 软链接应用  
你可以添加 `SCOOP` 用户环境变量更改用户安装目录  
添加 `SCOOP_GLOBAL` 系统变量更改全局安装目录

## 安装软件

使用 `scoop install <PackageName>` 安装软件  
在 PackageName 后加 @version 安装指定版本软件  
添加 `-g` 属性安装到全局目录

## 卸装软件

将上面的 install 改为 uninstall

## 更新软件

将上面的 install 改为 update  
可以使用 `scoop update *` 更新全部软件

## 查看软件是否最新

`scoop status`

## 重置软件，可切换软件版本

`scoop reset <PackageName>`

如同时安装 zulu8 和 zulu11,切换 jdk 环境为 zulu8

`scoop reset zulu8`

## 清除软件旧版本

`scoop cleanup <PackageName>`

清除所有软件旧版本 `scoop cleanup *`

## 搜索包

`scoop search <PackageName>`

## 列出已安装包

`scoop list`

## 添加 bukect

Scoop 的软件数量不多，但可以通过 bukect 添加软件源  
输入 `scoop bucket known` 列出官方认证的 bucket

```
PS C:\Users\Elio> scoop bucket known
main
extras
versions
nightlies
nirsoft
php
nerd-fonts
nonportable
java
games
jetbrains
```

输入 `scoop add java` 就可以添加 java 源

也可以输入 `scoop bucket add <name> [<仓库地址>]` 添加其他源

## 使用代理

`scoop config proxy 127.0.0.1:1080`

## 配置文件路径

`C:\Users\<用户名>\.config\scoop\`

## 缺点

- Scoop 大量使用 github 作为下载来源，建议使用代理
- 国内软件，gui 软件收录较少
- 安装失败需要卸装后再次安装

[1]: https://i.loli.net/2021/11/26/da8FIo7H1eUKjm5.png

