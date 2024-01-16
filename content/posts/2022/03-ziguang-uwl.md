---
title: '输入法词库解析（三）紫光拼音词库.uwl'
date: 2022-05-24T16:18:55+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
series: [lexicon]
#draft: true
---

> 详细代码：<https://github.com/nopdan/rose>

## 前言

`.uwl` 是紫光拼音输入法（现在叫华宇拼音输入法）使用的词库。

## 解析

紫光的词库有点复杂，拼音用的索引，但是拼音表没有写在词库里。

好在深蓝词库转换工具已经解析好了，这部分就跳过了。

词长和拼音长关系密切，要注意。

主要词库部分每 1024 字节为一段（分段意义何在？）

前两个字节未知，第 3 个字节表示字符编码格式 0x08 是 GBK，0x09 是 UTF-16LE。

| 范围        | 描述     |
| :---------- | :------- |
| 0x04 - 0x23 | 词库名   |
| 0x24 - 0x43 | 词库作者 |
| 0x44 - 0x47 | 词条数   |
| 0x48 - 0x4B | 分为几段 |

0x4C - 0xBFF 未知。

从 0xC00 开始，每 1024 为一段，一段有个 16 字节的头信息。

4 个字节为一组，分别表示：第几段，未知，未知，词条占用字节数(小于 1024-16)。

![](https://tucang.cc/api/image/show/60b7ae73b574e23cef39fe01007299e5)

|     | 占用字节数         | 描述                                          |
| :-- | :----------------- | :-------------------------------------------- |
| a   | 1                  | 词占用字节 + 1（所以总是奇数），可能大于 0x80 |
| b   | 1                  | 拼音长度的一半，前 4 位（总是偶数）意义不明   |
|     | 2                  | 词频                                          |
|     | b%0x10\*2 + a/0x80 | 拼音索引                                      |
|     | a%0x80 - 1         | 词                                            |

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

func (ZiguangUwl) Parse(filename string) Dict {
    data, _ := os.ReadFile(filename)
    r := bytes.NewReader(data)
    ret := make(Dict, 0, r.Len()>>8)
    r.Seek(2, 0)
    // 编码格式，08 为 GBK，09 为 UTF-16LE
    encoding, _ := r.ReadByte()

    // 分段
    r.Seek(0x48, 0)
    partLen := ReadUint32(r)
    for i := 0; i < partLen; i++ {
        r.Seek(0xC00+int64(i)<<10, 0)
        ret = parseZgUwlPart(r, ret, encoding)
    }

    return ret
}

func parseZgUwlPart(r *bytes.Reader, ret Dict, e byte) Dict {
    r.Seek(12, 1)
    // 词条占用字节数
    max := ReadUint32(r)
    // 当前字节
    curr := 0
    for curr < max {
        head := make([]byte, 4)
        r.Read(head)
        // 词长 * 2
        wordLen := head[0]%0x80 - 1
        // 拼音长
        codeLen := head[1]<<4>>4*2 + head[0]/0x80
        // 频率
        freq := BytesToInt(head[2:])
        // fmt.Println(freqSli, freq)
        curr += int(4 + wordLen + codeLen*2)

        // 拼音
        code := make([]string, 0, codeLen)
        for i := 0; i < int(codeLen); i++ {
            bsm, _ := r.ReadByte()
            bym, _ := r.ReadByte()
            smIdx := bsm & 0x1F
            ymIdx := (bsm >> 5) + (bym << 3)
            // fmt.Println(bsm, bym, smIdx, ymIdx)
            if bym >= 0x10 || smIdx >= 24 || ymIdx >= 34 {
                break
            }
            code = append(code, uwlSm[smIdx]+uwlYm[ymIdx])
            // fmt.Println(smIdx, ymIdx, uwlSm[smIdx]+uwlYm[ymIdx])
        }

        // 词
        tmp := make([]byte, wordLen)
        r.Read(tmp)
        var word string
        switch e {
        case 0x08:
            word, _ = util.Decode(tmp, "GBK")
        case 0x09:
            word, _ = util.Decode(tmp, "UTF-16LE")
        }
        // fmt.Println(string(word))
        ret = append(ret, Entry{word, code, freq})
    }
    return ret
}
```

---

**参考资料：**

[深蓝词库转换](https://github.com/studyzy/imewlconverter)
