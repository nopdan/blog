---
title: "输入法词库解析（四）百度分类词库.bdict"
date: 2022-05-24T18:35:32+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
# draft: true
---

> 参考深蓝词库转换

码表偏移 0x350

词库不带拼音表，需要根据词库规纳出来，参考深蓝

内部根据是否含有英文分为几种格式

### 格式一

纯中文

![](https://tucang.cc/api/image/show/9682895a284837224335c5f8447cca9f)

| 占用字节数   | 描述                              |
| ------------ | --------------------------------- |
| 4            | 拼音长，词长                      |
| 由上一项决定 | 拼音，（声母索引<24+韵母索引<33） |
| 由上一项决定 | 词，utf-16le 编码                 |

带英文的，结构差不多，声母索引为 0xFF 表示英文字母

![](https://tucang.cc/api/image/show/7fe0e61c95ce93052a6d18747c28195d)

### 格式二：纯英文

编码使用 ascii

![](https://tucang.cc/api/image/show/1c5a7c52942eea72aee3bc1a97bafb9f)

| 占用字节数   | 描述           |
| ------------ | -------------- |
| 4            | 词长           |
| 由上一项决定 | 词，ascii 编码 |

### 格式三：编码和词不等长

拼音不再使用索引，而是直接使用 utf-16le 编码

![](https://tucang.cc/api/image/show/6e0cad6df09a2a39e1179925155f47c5)

| 占用字节数       | 描述              |
| ---------------- | ----------------- |
| 4                | 编码字节长        |
| 2                | 2 个空字节        |
| 2                | 词长              |
| 由编码字节长决定 | 编码，utf-16le    |
| 由词长决定       | 词，utf-16le 编码 |

## 代码

`golang` 实现：

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

func ParseBaiduBdict(rd io.Reader) []Pinyin {
    ret := make([]Pinyin, 0, 1e5)
    data, _ := ioutil.ReadAll(rd)
    r := bytes.NewReader(data)

    // utf-16le 转换器
    decoder := unicode.UTF16(unicode.LittleEndian, unicode.IgnoreBOM).NewDecoder()

    r.Seek(0x350, 0)
    for r.Len() > 4 {
        // 拼音长，也是词长
        codeLen, _ := r.ReadByte()
        r.Seek(3, 1) // 丢掉跟着的3个0

        // 判断下两个字节
        tmp := make([]byte, 2)
        r.Read(tmp)
        if tmp[0] == 0 && tmp[1] == 0 {
            r.Read(tmp)
            wordLen := bytesToInt(tmp)
            codeSli := make([]byte, codeLen*2)
            r.Read(codeSli)
            wordSli := make([]byte, wordLen*2)
            r.Read(wordSli)
            codeSli, _ = decoder.Bytes(codeSli)
            wordSli, _ = decoder.Bytes(wordSli)
            ret = append(ret, Pinyin{string(wordSli), []string{string(codeSli)}, 1})
            continue
        }

        // 全英文的词，编码和词是一样的
        if int(tmp[0]) >= len(bdictSm) && tmp[0] != 0xff {
            r.Seek(-2, 1)
            eng := make([]byte, codeLen)
            r.Read(eng)
            ret = append(ret, Pinyin{string(eng), []string{string(eng)}, 1})
            continue
        }

        r.Seek(-2, 1)
        // 一般格式
        code := make([]string, 0, codeLen)
        for i := 0; i < int(codeLen); i++ {
            smIdx, _ := r.ReadByte()
            ymIdx, _ := r.ReadByte()
            // 带英文的词组
            if smIdx == 0xff {
                code = append(code, string(ymIdx))
                continue
            }
            code = append(code, bdictSm[smIdx]+bdictYm[ymIdx])
        }
        // 读词
        wordSli := make([]byte, 2*codeLen)
        r.Read(wordSli)
        wordSli, _ = decoder.Bytes(wordSli)
        ret = append(ret, Pinyin{string(wordSli), code, 1})
    }
    return ret
}

```
