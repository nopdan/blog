---
title: 清除"哪些图标显示在任务栏上"缓存
categories:
  - 笔记
tags:
  - windows
date: 2020-03-18
sidebar: false
---

在设置"哪些图标显示在任务栏上"后，当我们卸装软件、更改软件目录就可能导致缓存不刷新，只需如下几步就可以清除你的任务栏图标缓存。

<!-- more -->

![1][]

## 删注册表项

快捷键 win+r, 输入 regedit 打开注册表，找到  
`HKEY_CURRENT_USER\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\TrayNotify`  
删掉下面两项  
![2][]

## 重启资源管理器

快捷键 ctrl+shift+esc 打开任务管理器，找到 windows 资源管理器，右击重启。  
![3][]

[1]: https://p.pstatp.com/origin/pgc-image/d357306da5654cc68b8b490e2fff32eb
[2]: https://p.pstatp.com/origin/pgc-image/b481afa7d13c4148a05dadda27de076a
[3]: https://p.pstatp.com/origin/pgc-image/282a10fa02164ba0b604a0f53369ca7e
