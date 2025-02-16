---
title: '我用 go 写了一个赛码器'
date: 2021-08-16T19:48:53+08:00
categories:
  - 编程
tags:
  - go
  - 输入法
---

> github 项目地址：<https://github.com/nopdan/gosmq/>

<!--more-->

![](https://i.loli.net/2021/11/26/Fo4VJKqvyPjRWND.png)

![](https://i.loli.net/2021/11/26/3kyp7ZtRhwKONoc.png)

## 为什么要写赛码器

- 替代极速赛码器
  > 目前最流行的赛码器「极速赛码器」对系统资源利用极低，内存只用到 10 多 MB，cpu 占用百分之几，速度极慢，10w 字文章要几十秒，上百万文章基本不可用。
- 练习 go 语言
  > 这是我用 go 语言写的第一个项目

<!--more-->

## 技术

- 文件读取和写入
- 字典树(TrieTree)
- 一个好看的包 `go-pretty`
- go1.16 新特性 `go embed`

### 文件读取

- 一次性读取  
  `ioutil.ReadFile()`，性能最好
- 分片读取，适合读取二进制文件
  - `bufio.NewReader()`
  - `buf.Read()`
- 逐行读取
  - `bufio.Scanner()`，Scanner 读取单行太长会报错
  - `buf.Readline()`

### 文件写入

`ioutil.WriteFile()`

### 字符串拼接

> [字符串拼接性能及原理](https://geektutu.com/post/hpg-string-concat.html)

- `+` 号
- `[]byte`
- `bytes.Builder`
- `strings.Builder` 推荐

### 字典树

> [小白详解 Trie 树](https://segmentfault.com/a/1190000008877595)

- 数组实现：空间换时间

- 链表实现：时间换空间

- 哈希实现：时空平衡

- 双数组字典树：速度快，空间少，构建时间长

其它：tail 节点，ac 自动机，fail 指针

### 命令行输出美化

[go-pretty](https://github.com/jedib0t/go-pretty)
