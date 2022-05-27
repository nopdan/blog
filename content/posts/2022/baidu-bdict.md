---
title: "输入法词库解析（四）百度分类词库.bdict(.bcd)"
date: 2022-05-24T18:35:32+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
# draft: true
---

## 前言

`.bdict` 是百度的分类词库格式，可以在 <https://shurufa.baidu.com/dict> 下载。

手机百度的分类词库格式 `.bcd` 是一样的，可以在 <https://mime.baidu.com/web/iw/index/> 下载。

## 解析

| 范围          | 描述     |
| :------------ | :------- |
| 0x70 - 0x73   | 词条数   |
| 0x90 - 0xCF   | 词库名   |
| 0xD0 - 0x10F  | 词库作者 |
| 0x110 - 0x14F | 示例词   |
| 0x150 - 0x34F | 词库描述 |

> 有的词库在 0x250 开始的后 4 个字节是大端序的词条数。

码表偏移 0x350

词库不带拼音表，需要根据词库规纳出来，参考深蓝词库转换。

内部根据是否含有英文分为几种格式

### 格式一

纯中文

![](https://tucang.cc/api/image/show/9682895a284837224335c5f8447cca9f)

| #   | 占用字节数 | 描述                              |
| :-- | :--------- | :-------------------------------- |
| a   | 2          | 拼音长，词长                      |
|     | 2          | 词频                              |
|     | a\*2       | 拼音，（声母索引<24+韵母索引<33） |
|     | a\*2       | 词，utf-16le 编码                 |

带英文的，结构差不多，声母索引为 0xFF 表示英文字母

![](https://tucang.cc/api/image/show/7fe0e61c95ce93052a6d18747c28195d)

### 格式二：纯英文

编码使用 ascii

![](https://tucang.cc/api/image/show/1c5a7c52942eea72aee3bc1a97bafb9f)

| #   | 占用字节数 | 描述           |
| :-- | :--------- | :------------- |
| a   | 2          | 词长           |
|     | 2          | 词频           |
|     | a          | 词，ascii 编码 |

### 格式三：编码和词不等长

拼音不再使用索引，而是直接使用 utf-16le 编码

![](https://tucang.cc/api/image/show/6e0cad6df09a2a39e1179925155f47c5)

| #   | 占用字节数 | 描述           |
| :-- | :--------- | :------------- |
| a   | 2          | 编码数         |
|     | 2          | 词频           |
|     | 2          | 空             |
| b   | 2          | 词长           |
|     | a\*2       | 编码，utf-16le |
|     | b\*2       | 词，utf-16le   |

**_代码实现：_**

```go
var bdictSm = []string{
    "c", "d", "b", "f", "g", "h", "ch", "j", "k", "l", "m", "n",
    "", "p", "q", "r", "s", "t", "sh", "zh", "w", "x", "y", "z",
}

var bdictYm = []string{
    "uang", "iang", "iong", "ang", "eng", "ian", "iao", "ing", "ong",
    "uai", "uan", "ai", "an", "ao", "ei", "en", "er", "ua", "ie", "in", "iu",
    "ou", "ia", "ue", "ui", "un", "uo", "a", "e", "i", "o", "u", "v",
}

func ParseBaiduBdict(rd io.Reader) []PyEntry {
    ret := make([]PyEntry, 0, 1e5)
    data, _ := ioutil.ReadAll(rd)
    r := bytes.NewReader(data)
    var tmp []byte

    // 词条数
    r.Seek(0x250, 0)
    ReadInt(r, 4) // 暂时用不上

    r.Seek(0x350, 0)
    for r.Len() > 4 {
        // 拼音长
        codeLen := ReadInt(r, 2)
        // 词频
        freq := ReadInt(r, 2)

        // 判断下两个字节
        tmp = make([]byte, 2)
        r.Read(tmp)

        // 编码和词不等长，全按 utf-16le
        if tmp[0] == 0 && tmp[1] == 0 {
            wordLen := ReadInt(r, 2)
            // 读编码
            tmp = make([]byte, codeLen*2)
            r.Read(tmp)
            code := string(DecUtf16le(tmp))
            // 读词
            tmp = make([]byte, wordLen*2)
            r.Read(tmp)
            word := string(DecUtf16le(tmp))

            ret = append(ret, PyEntry{word, []string{code}, freq})
            continue
        }

        // 全英文的词，编码和词是一样的
        if int(tmp[0]) >= len(bdictSm) && tmp[0] != 0xff {
            r.Seek(-2, 1)
            eng := make([]byte, codeLen)
            r.Read(eng)
            ret = append(ret, PyEntry{string(eng), []string{string(eng)}, freq})
            continue
        }

        // 一般格式
        r.Seek(-2, 1)
        codes := make([]string, 0, codeLen)
        for i := 0; i < codeLen; i++ {
            smIdx, _ := r.ReadByte()
            ymIdx, _ := r.ReadByte()
            // 带英文的词组
            if smIdx == 0xff {
                codes = append(codes, string(ymIdx))
                continue
            }
            codes = append(codes, bdictSm[smIdx]+bdictYm[ymIdx])
        }
        // 读词
        tmp = make([]byte, 2*codeLen)
        r.Read(tmp)
        word := string(DecUtf16le(tmp))
        ret = append(ret, PyEntry{word, codes, freq})
    }
    return ret
}
```

---

**参考资料：**

[深蓝词库转换](https://github.com/studyzy/imewlconverter)
