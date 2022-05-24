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

紫光的词库有点复杂。

前面是拼音表，但是没有拼音只有索引，应该是写到了程序内部。好在深蓝词库处理工具已经解析好了，这部分就跳过了。

词库偏移 0xC10。

![](https://tucang.cc/api/image/show/60b7ae73b574e23cef39fe01007299e5)

| 占用字节数 | 描述                                                             |
| ---------- | ---------------------------------------------------------------- |
| 1          | 词占用字节 + 1（所以总是奇数），大于 0x80 要减掉，并且下一项要+1 |
| 1          | 拼音长度的一半，前 4 位（总是偶数）意义不明                      |
| 2          | 词频                                                             |
| 由前面决定 | 拼音索引                                                         |
| 由前面决定 | 词，utf-16le 编码                                                |

> 拼音长度由前 2 字节决定，具体看代码实现

词库中有些无效字节，主要是两种。

一种是空字节，2 个一组。

另一种 16 字节一组，`6C 1D 00 00 FF FF FF FF F8 00 00 00 F4 01 00 00`

再按 4 个一组，后两位都是 `00 00` 或 `FF FF`（还有 `01 00`）

去除的判断我直接写的后两字节差 < 2

`golang` 实现：

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

func ParseZiguangUwl(rd io.Reader) []Pinyin {
    ret := make([]Pinyin, 0, 1e5)
    data, _ := ioutil.ReadAll(rd)
    r := bytes.NewReader(data)

    // utf-16le 转换器
    decoder := unicode.UTF16(unicode.LittleEndian, unicode.IgnoreBOM).NewDecoder()

    r.Seek(0xC10, 0)
    for r.Len() > 7 {
        // 读2个字节
        space := make([]byte, 2)
        r.Read(space)
        // 2字节都为空
        if space[0] == 0 && space[1] == 0 {
            continue
        }
        if space[0]%2 == 0 || // 1字节是偶数
            space[1]>>4%2 != 0 || // 2字节前4位是奇数
            space[1]%0x10 == 0 { // 2字节后4位是0
            r.Seek(14, 1)
            continue
        }
        r.Seek(-2, 1) // 回退2字节

        // 读 16 字节
        tmp := make([]byte, 16)
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
        wordLen := head[0] - 1
        // 拼音长
        codeLen := head[1] % 0x10 * 2
        if wordLen > 0x80 {
            wordLen -= 0x80
            codeLen++
        }

        // 频率
        freq := bytesToInt(head[2:])
        // fmt.Println(freqSli, freq)

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
        wordSli := make([]byte, wordLen)
        r.Read(wordSli)
        word, _ := decoder.Bytes(wordSli)
        // fmt.Println(string(word))

        ret = append(ret, Pinyin{string(word), code, freq})

    }
    return ret
}

```
