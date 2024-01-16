# 清除"哪些图标显示在任务栏上"缓存


在设置"哪些图标显示在任务栏上"后，当我们卸装软件、更改软件目录就可能导致缓存不刷新，只需如下几步就可以清除你的任务栏图标缓存。

<!-- more -->

![1][]

## 删注册表项

快捷键 win+r, 输入 regedit 打开注册表，找到  
`HKEY_CURRENT_USER\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\TrayNotify`  
找到以下两个键值`IconStreams`和`PastIconsStream`，将它们的值删除。  
![2][]

## 重启资源管理器

快捷键 ctrl+shift+esc 打开任务管理器，找到 windows 资源管理器，右击重启。  
![3][]

[1]: https://i.loli.net/2021/11/26/ZRnJsLxaT1dKi8G.png
[2]: https://i.loli.net/2021/11/26/Ae8ubLYljKiyxsn.png
[3]: https://i.loli.net/2021/11/26/BZyhs4me3dcxuiE.png

