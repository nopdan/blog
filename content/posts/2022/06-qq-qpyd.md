---
title: '输入法词库解析（六）QQ 拼音分类词库.qpyd'
date: 2022-05-25T15:29:14+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
series: [lexicon]
# draft: true
---

> 详细代码：<https://github.com/imetool/dtool>

## 前言

`.qpyd` 是 QQ 拼音输入法 6.0 以下版本所用的词库格式，可以在 <http://cdict.qq.pinyin.cn/v1/> 下载。

该格式解析的主要难点是其使用了 zlib 压缩，解压后的数据很好解析。

## 解析

### 原始文件

![](https://tucang.cc/api/image/show/b86501a86fa0ace3fa09f817a4c855cf)

0x38 后跟的 4 字节表示压缩数据开始的字节。

0x44 后跟的 4 字节表示词条数。

0x60 - 0x16F 是词库的一些描述信息。

其余未知。

### 解压数据

使用了 `zlib` 格式。

我们看看解压后的数据是什么形式。

可以发现它分为两部分，前部分每 10 个一组，总长 10\*词条数。

放到文本编辑器里分析一下，这里取了前后两部分前三条。

![](https://tucang.cc/api/image/show/5f653dd803e89fcca72f59eea9966b52)

可以看到前部分是编码长和词长信息，后半部分 ascii 的编码 + utf-16le 的词条。

前半部分保存了所有词条的编码长，词长，索引位置。

![](https://tucang.cc/api/image/show/ef9e706967a70db50a901d2f8ed69e6c)

| 占用字节数 | 描述                    |
| ---------- | ----------------------- |
| 1          | 拼音的长度              |
| 1          | 词字节长                |
| 4          | 未知，全是`00 00 80 3F` |
| 4          | 词条的索引位置          |

后半部分就是词条本身了，拼音和词，词条之间都是紧挨着的。

![](https://tucang.cc/api/image/show/78f1112a8bc8ef7ec681162e71ad2e2f)

前面是编码，框里的是词。

**_代码实现：_**

```go
func (QqQpyd) Parse(filename string) Dict {
    data, _ := os.ReadFile(filename)
    r := bytes.NewReader(data)
    ret := make(Dict, 0, r.Len()>>8)
    var tmp []byte

    // 0x38 后跟的是压缩数据开始的偏移量
    r.Seek(0x38, 0)
    startZip := ReadUint32(r)
    // 0x44 后4字节是词条数
    r.Seek(0x44, 0)
    dictLen := ReadUint32(r)
    // 0x60 到zip数据前的一段是一些描述信息
    r.Seek(0x60, 0)
    head := make([]byte, startZip-0x60)
    r.Read(head)
    // headStr, _ := Decode(head, "UTF-16LE")
    // fmt.Println(headStr) // 打印描述信息

    // 解压数据
    zrd, err := zlib.NewReader(r)
    if err != nil {
        log.Panic(err)
    }
    defer zrd.Close()
    buf := new(bytes.Buffer)
    buf.Grow(r.Len())
    _, err = io.Copy(buf, zrd)
    if err != nil {
        log.Panic(err)
    }
    // 解压完了
    r.Reset(buf.Bytes())

    for i := 0; i < dictLen; i++ {
        // 指向当前
        r.Seek(int64(10*i), 0)

        // 读码长、词长、索引
        addr := make([]byte, 10)
        r.Read(addr)
        idx := BytesToInt(addr[6:]) // 后4字节是索引
        r.Seek(int64(idx), 0)       // 指向索引
        // 读编码，自带 ' 分隔符
        tmp = make([]byte, addr[0])
        r.Read(tmp)
        code := string(tmp)
        // 读词
        tmp = make([]byte, addr[1])
        r.Read(tmp)
        word, _ := util.Decode(tmp, "UTF-16LE")

        ret = append(ret, Entry{word, strings.Split(code, "'"), 1})
    }
    return ret
}
```
