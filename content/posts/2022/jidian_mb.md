---
title: "输入法词库解析（五）极点码表.mb"
date: 2022-05-24T22:09:45+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
# draft: true
---

前 0x1A 个是版本信息

![](https://tucang.cc/api/image/show/f247661553894fe968bdbe1fbae0061a)

0x1B ~ 0x11E 是词库的描述信息，utf-16le 编码

上图部分解析为

```
五笔点儿词库2022春 QQ群313225526
生成日期:2022-3-17 18:3
```

第 0x11F，这个字节表示所有用到的按键数，1A 即 26 键

![](https://tucang.cc/api/image/show/a5de036b87b103b22399e3be42249cda)

0x120 开始，就是用到的按键，utf-16le

后面跟了 20 个空字节

![](https://tucang.cc/api/image/show/a22851de0f98136734115c9970d24c80)

组词规则，后面又跟了 20 个空字节

![](https://tucang.cc/api/image/show/a905bf4c20363e20d4abb900eabfcf31)

下面 4 字节，可能是特殊符号引导符

下面的部分就有规律了

![](https://tucang.cc/api/image/show/753ba4dd9df4658efaeee98719d8a489)

每 4 个字节一组，前两个字节表示一个字符，后两个字节从 00 00 ~ 29 00，一共 41 个值（可能是对应的首字母），中间有一些 `FF FF FF FF`

一直到 `0x1B620`左右，有的词库可能会相差几个字节。

下面才是词库部分。

![](https://tucang.cc/api/image/show/24595503ce2b7866585c4d0ee04cfb47)

| 占用字节数       | 描述                                   |
| ---------------- | -------------------------------------- |
| 1                | 编码长度                               |
| 1                | 词字节长度                             |
| 1                | 只有 0x64、0x32、0x10 几个值，意义不明 |
| 取决于编码长度   | 编码，ascii                            |
| 取决于词字节长度 | 词，utf-16le                           |

`golang` 实现（只读 0x1B620 之后的码表）：

```go
func ParseJidianMb(rd io.Reader) Dict {

    ret := make(Dict, 1e5)       // 初始化
    tmp, _ := ioutil.ReadAll(rd) // 全部读到内存
    r := bytes.NewReader(tmp)
    r.Seek(0x1B620, 0) // 从 0x1B620 开始读
    // utf-16le 转换
    decoder := unicode.UTF16(unicode.LittleEndian, unicode.IgnoreBOM).NewDecoder()
    for r.Len() > 3 {
        codeLen, _ := r.ReadByte()
        if codeLen == 0xff {
            r.Seek(1, 1)
            continue
        }
        wordLen, _ := r.ReadByte()

        r.Seek(1, 1) // 丢掉一个字节
        codeSli := make([]byte, codeLen)
        r.Read(codeSli)
        wordSli := make([]byte, wordLen)
        r.Read(wordSli)
        wordSli, _ = decoder.Bytes(wordSli)
        ret.insert(string(codeSli), string(wordSli))
    }

    return ret
}
```
