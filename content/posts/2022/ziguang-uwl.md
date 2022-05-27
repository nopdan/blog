---
title: "输入法词库解析（三）紫光拼音词库.uwl"
date: 2022-05-24T16:18:55+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
#draft: true
---

> 详细代码：<https://github.com/cxcn/dtool>

## 前言

`.uwl` 是紫光拼音输入法（现在叫华宇拼音输入法）使用的词库。

## 解析

紫光的词库有点复杂，拼音用的索引，但是拼音表没有写在词库里。

好在深蓝词库处理工具已经解析好了，这部分就跳过了。

词长和拼音长关系密切，要注意。

中间有不少无效字节，读取时要处理掉。

| 范围        | 描述     |
| :---------- | :------- |
| 0x00 - 0x03 | 未知     |
| 0x04 - 0x23 | 词库名   |
| 0x24 - 0x43 | 词库作者 |
| 0x44 - 0x47 | 词条数   |

后面就不知道了，怀疑就是无效字节。

词库偏移 0xC10。

![](https://tucang.cc/api/image/show/60b7ae73b574e23cef39fe01007299e5)

|     | 占用字节数    | 描述                                          |
| :-- | :------------ | :-------------------------------------------- |
| a   | 1             | 词占用字节 + 1（所以总是奇数），可能大于 0x80 |
| b   | 1             | 拼音长度的一半，前 4 位（总是偶数）意义不明   |
|     | 2             | 词频                                          |
|     | 2\*b + a/0x80 | 拼音索引                                      |
|     | a%0x80 + 1    | 词，utf-16le 编码                             |

词库中有些无效字节，主要是两种。

一种是空字节，2 个一组。

另一种 16 字节一组，`6C 1D 00 00 FF FF FF FF F8 00 00 00 F4 01 00 00`

再按 4 个一组，后两位都是 `00 00` 或 `FF FF`（还有的是 `01 00`）

去除的判断我直接写的后两字节差 < 2

**_代码实现：_**

```go
var uwlSm = []string{
    "", "b", "c", "ch", "d", "f", "g", "h", "j", "k", "l", "m", "n",
    "p", "q", "r", "s", "sh", "t", "w", "x", "y", "z", "zh",
}

var uwlYm = []string{
    "ang", "a", "ai", "an", "ang", "ao", "e", "ei", "en", "eng", "er",
    "i", "ia", "ian", "iang", "iao", "ie", "in", "ing", "iong", "iu",
    "o", "ong", "ou", "u",
    "ua", "uai", "uan", "uang", "ue", "ui", "un", "uo", "v",
}

func ParseZiguangUwl(rd io.Reader) []PyEntry {
    ret := make([]PyEntry, 0, 1e5)
    data, _ := ioutil.ReadAll(rd)
    r := bytes.NewReader(data)
    var tmp []byte

    r.Seek(0xC10, 0)
    for r.Len() > 7 {
        // 读2个字节
        space := make([]byte, 2)
        r.Read(space)
        // 2字节都为空
        if space[0] == 0 && space[1] == 0 {
            continue
        }
        r.Seek(-2, 1) // 回退2字节

        // 读 16 字节
        tmp = make([]byte, 16)
        r.Read(tmp)
        flag := true // 是否丢弃
        for i := 0; i < 4; i++ {
            // 后两个字节相差不超过1
            if tmp[4*i+2]-tmp[4*i+3] < 2 {
                continue
            } else {
                flag = false
            }
        }
        if flag {
            continue
        }
        r.Seek(-16, 1)

        // 正式读
        head := make([]byte, 4)
        r.Read(head)
        // 词长 * 2
        wordLen := head[0]%0x80 - 1
        // 拼音长
        codeLen := head[1]<<4>>4*2 + head[0]/0x80

        // 频率
        freq := BytesToInt(head[2:])

        // 拼音
        code := make([]string, 0, codeLen)
        for i := 0; i < int(codeLen); i++ {
            bsm, _ := r.ReadByte()
            bym, _ := r.ReadByte()
            smIdx := bsm & 0x1F
            ymIdx := (bsm >> 5) + (bym << 3)
            if bym >= 0x10 || smIdx >= 24 || ymIdx >= 34 {
                break
            }
            code = append(code, uwlSm[smIdx]+uwlYm[ymIdx])
        }

        // 词
        tmp = make([]byte, wordLen)
        r.Read(tmp)
        word := string(DecUtf16le(tmp))

        ret = append(ret, PyEntry{word, code, freq})
    }
    return ret
}
```

---

**参考资料：**

[深蓝词库转换](https://github.com/studyzy/imewlconverter)
