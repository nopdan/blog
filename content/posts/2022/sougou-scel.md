---
title: "输入法词库解析（二）搜狗拼音细胞词库.scel"
date: 2022-05-24T15:39:17+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
# draft: true
---

前面很多空字节的地方不用管，是一些描述信息，词库名、示例词等。

**0x124 跟的 4 个字节是词条数，新的 scel 在文件最后面可能有违禁词（黑名单词）。**

## 拼音表

直接从 0x1540 开始。

![](https://tucang.cc/api/image/show/7c7ba509a61717b904ec30fea41fb488)

前两个字节是拼音表的长度，这里面都是按小端算的。

这里 `9D 01` 就表示有 `0x100 * 0x01 + 0x9D = 413` 组。

后两个字节意义不明，一般是 0。

从 0x1544 开始就是拼音表正文部分。

| 占用字节数   | 描述                                     |
| ------------ | ---------------------------------------- |
| 2            | 索引，从 00 00 到 9C 01                  |
| 2            | 拼音字节的长度                           |
| 由上一项决定 | 拼音，utf-16le 编码，一个字母占 2 字节。 |

## 词条

偏移量 0x2628

![](https://tucang.cc/api/image/show/1ccbf630635055b1a5c74949c4ef90ac)

| 占用字节数   | 描述               |
| ------------ | ------------------ |
| 2            | 同一个音有多少词   |
| 2            | 拼音索引的字节长度 |
| 由上一项决定 | 拼音索引数组       |
| 2            | 词占用字节数       |
| 由上一项决定 | 词，utf-16le 编码  |
| 2            | 描述信息字节长度   |
| 由上一项决定 | 描述               |

--- 
**带英文词库的索引**

从拼音表的长度往后，依次是 abcd。比如表长 413，最大索引`9D 01`，则下一个索引`9E 01`表示字母 a，依次类推。

## `golang` 代码实现

```go

func ParseSougouScel(rd io.Reader) []Pinyin {
    ret := make([]Pinyin, 0, 1e5)
    data, _ := ioutil.ReadAll(rd)
    r := bytes.NewReader(data)

    // utf-16le 转换器
    decoder := unicode.UTF16(unicode.LittleEndian, unicode.IgnoreBOM).NewDecoder()

    // 读词条数
    tmp := make([]byte, 4)
    r.Seek(0x124, 0)
    r.Read(tmp)
    dictLen := bytesToInt(tmp)

    // 拼音表偏移量
    r.Seek(0x1540, 0)

    // 前两个字节是拼音表长度，413
    tmp = make([]byte, 2)
    r.Read(tmp)
    pyTableLen := bytesToInt(tmp)
    pyTable := make([]string, pyTableLen)
    // fmt.Println("拼音表长度", pyTableLen)

    // 丢掉两个字节
    r.Read(tmp)

    // 读拼音表
    for i := 0; i < pyTableLen; i++ {
        // 索引
        tmp := make([]byte, 2)
        r.Read(tmp)
        idx := bytesToInt(tmp)

        // 拼音长度
        r.Read(tmp)
        pyLen := bytesToInt(tmp)

        // 拼音 utf-16le
        pySli := make([]byte, pyLen)
        r.Read(pySli)
        py, _ := decoder.Bytes(pySli)

        pyTable[idx] = string(py)
    }

    // 读码表
    for count := 0; count < dictLen; {
        // 重码数（同一串音对应多个词）
        tmp := make([]byte, 2)
        r.Read(tmp)
        repeat := bytesToInt(tmp)

        // 索引数组长
        r.Read(tmp)
        codeLen := bytesToInt(tmp)

        // 读取编码
        var code []string
        for i := 0; 2*i < codeLen; i++ {
            r.Read(tmp)
            theIdx := bytesToInt(tmp)
            if theIdx >= pyTableLen {
                code = append(code, string(byte(theIdx-pyTableLen+97)))
                continue
            }
            code = append(code, pyTable[theIdx])
        }

        // 读取一个或多个词
        count += repeat
        for i := 1; i <= repeat; i++ {
            // 词长
            r.Read(tmp)
            wordLen := bytesToInt(tmp)

            // 读取词
            wordSli := make([]byte, wordLen)
            r.Read(wordSli)
            wordSli, _ = decoder.Bytes(wordSli)
            word := string(wordSli)
            ret = append(ret, Pinyin{word, code, 1})

            // 末尾的补充信息
            r.Read(tmp)
            infoLen := bytesToInt(tmp)
            info := make([]byte, infoLen)
            r.Read(info)
        }
    }
    return ret
}

```
