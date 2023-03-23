---
title: '输入法词库解析（五）极点码表.mb'
date: 2022-05-24T22:09:45+08:00
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

`mb` 是极点五笔的码表格式。

## 解析

| 偏移量       | 描述                   |
| :----------- | :--------------------- |
| 0x00         | 版本信息               |
| 0x1B         | 码表介绍               |
| 0x11F        | 所用到的按键数         |
| 0x120        | 所用到的按键，utf-16le |
| 0x154        | 万能键                 |
| 0x156        | 编码截止键             |
| 0x176        | 组词规则               |
| 0x176        | 组词规则               |
| 0x194        | 径直上屏的标点         |
| 0x1B4        | 特殊符号引导符         |
| 0x1B8        | 未知                   |
| 0x1B620 左右 | 码表                   |

![](https://tucang.cc/api/image/show/f247661553894fe968bdbe1fbae0061a)

上图选中部分解析为

```
五笔点儿词库2022春 QQ群313225526
生成日期:2022-3-17 18:36
```

所有用到的按键：
![](https://tucang.cc/api/image/show/a5de036b87b103b22399e3be42249cda)

组词规则：
![](https://tucang.cc/api/image/show/a22851de0f98136734115c9970d24c80)

特殊符号引导符：
![](https://tucang.cc/api/image/show/a905bf4c20363e20d4abb900eabfcf31)

下面的部分就有规律了

![](https://tucang.cc/api/image/show/753ba4dd9df4658efaeee98719d8a489)

每 4 个字节一组，前两个字节表示一个字符，后两个字节从 00 00 ~ 29 00，一共 41 个值（意义不明，可能是某种索引），中间有一些 `FF FF FF FF`

一直到 `0x1B620`左右，有的词库可能会相差几个字节。

下面才是词库部分。

![](https://tucang.cc/api/image/show/24595503ce2b7866585c4d0ee04cfb47)

|     | 占用字节数 | 描述                                   |
| :-- | :--------- | :------------------------------------- |
| a   | 1          | 编码长度                               |
| b   | 1          | 词字节长度                             |
|     | 1          | 只有 0x64、0x32、0x10 几个值，意义不明 |
|     | a          | 编码，ascii                            |
|     | b          | 词，utf-16le                           |

**_代码实现（只读 0x1B620 之后的码表）：_**

```go
func (JidianMb) Parse(filename string) Table {
    data, _ := os.ReadFile(filename)
    r := bytes.NewReader(data)
    ret := make(Table, 0, r.Len()>>8)
    var tmp []byte

    r.Seek(0x1B620, 0) // 从 0x1B620 开始读
    for r.Len() > 3 {
        codeLen, _ := r.ReadByte()
        if codeLen == 0xff {
            r.Seek(1, 1)
            continue
        }
        wordLen, _ := r.ReadByte()
        r.Seek(1, 1)

        // 读编码
        tmp = make([]byte, codeLen)
        r.Read(tmp)
        code := string(tmp)

        // 读词
        tmp = make([]byte, wordLen)
        r.Read(tmp)
        word, _ := util.Decode(tmp, "UTF-16LE")

        ret = append(ret, Entry{word, code, 1})
    }
    return ret
}
```
