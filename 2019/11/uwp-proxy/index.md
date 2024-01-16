# 为uwp应用设置系统代理


Win 10 的 UWP 应用 (应用商店下载的 APP)，默认是不走代理的 (沙盒的网络隔离特性：禁止 APP 访问 localhost)，也就无法在中国使用像 Facebook 一样无法访问或者直接链接特别慢的 APP。

<!-- more -->

## 一、CheckNetIsolation

### 为单个 UWP 应用设置代理

#### 指定 app 名称

1. 在资源管理器地址栏输入：

```
C:\Users\%username%\AppData\Local\Packages
```

![](https://i.loli.net/2021/11/26/pDxNFfQYL8sIVnA.png)

2. 命令行输入：

```powershell
CheckNetIsolation.exe LoopbackExempt -a -n="APP名称"
# 例
CheckNetIsolation.exe LoopbackExempt -a -n="903DB504.QQWP_a99ra4d2cbcxa"
```

1. 如需取消只需将 _-a_ 换成 _-d_

#### 指定 SID

1. 找 SID，在地址栏输入：

```
HKEY_CURRENT_USER\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\AppContainer\Mappings
```

![](https://i.loli.net/2021/11/26/EVzc3CHZja4Dq6A.png)

2. 命令行输入：

```powershell
CheckNetIsolation.exe LoopbackExempt -a -p=APP的SID
# 例
CheckNetIsolation.exe loopbackexempt -a -p=S-1-15-2-3603386487-2480558841-1671909046-581189492-76553544-2527268699-3538496040
```

1. 如需取消只需将 _-a_ 换成 _-d_

### 为全部 UWP 应用设置代理

powershell 命令(读取注册表中的所有 SID)：

```powershell
foreach ( $Obj in Get-ChildItem "HKCU:\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\AppContainer\Mappings\" -name ) { CheckNetIsolation.exe LoopbackExempt -a -n=$Oj }
```

CMD 命令：

```powershell
FOR /F "tokens=11 delims=\" %p IN ('REG QUERY "HKCU\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\AppContainer\Mappings"') DO CheckNetIsolation.exe LoopbackExempt -a -p=%p
```

## 二、使用其他软件

Fiddler: <https://www.telerik.com/fiddler>

## 三、python 脚本

<https://yuan.ga/enable-win10-uwp-use-system-proxy/>

