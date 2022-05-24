---
title: "输入法词库解析（一）百度自定义方案.def"
date: 2022-05-24T15:04:00+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
# draft: true
---

> 参考了 asd1fque1 的词库处理工具 js 实现

## 解析

码表偏移量 `0x6D`

![](https://tucang.cc/api/image/show/3ab4d59dda60d731eb6ddb55c7694bd5)

| 占用字节数     | 描述                                               |
| -------------- | -------------------------------------------------- |
| 1              | 编码长度（红色框）                                 |
| 1              | 词长 \* 2 + 2                                      |
| 由编码长度决定 | 编码（黄色框），可以是纯编码，也可以是 `编码=位置` |
| 由词长决定     | 词（绿色框），utf16-le 编码                        |
| 6              | 6 个空字节代表词条结束                             |

`golang` 实现：

```go
func ParseBaiduDef(rd io.Reader) Dict {
    ret := make(Dict, 1e5)       // 初始化
    tmp, _ := ioutil.ReadAll(rd) // 全部读到内存
    r := bytes.NewReader(tmp)
    r.Seek(0x6D, 0) // 从 0x6D 开始读
    // utf-16le 转换
    decoder := unicode.UTF16(unicode.LittleEndian, unicode.IgnoreBOM).NewDecoder()
    for {
        codeLen, err := r.ReadByte() // 编码长度
        wordLen, err := r.ReadByte() // 词长*2 + 2
        if err != nil {
            break
        }
        sliCode := make([]byte, int(codeLen))
        sliWord := make([]byte, int(wordLen)-2) // -2 后就是字节长度，没有考虑4字节的情况

        r.Read(sliCode) // 编码切片
        r.Read(sliWord)

        code := string(sliCode)
        word, _ := decoder.Bytes(sliWord)
        ret.insert(strings.Split(code, "=")[0], string(word))

        r.Seek(6, 1) // 6个00，1是相对当前位置
    }
    return ret
}

```

## 生成

码表部分和解析一样的，没什么好说的。

主要考虑前 0x6C(109) 个字节。

第一个字节意义不明，可能是最大码长（一般是 0，有的码表里是 4）

![](https://tucang.cc/api/image/show/0af196fdd73d7d06ae6d74f3dcab8394)

后面每 4 字节一组，共 27 组。

**表示以 26 个首字母开头词条的字节长度累加**（不包括前 2 个表示长度的字节，包括后 6 个 0）

计算时，统计每个首字母的长度累计，写入时再次累加。

`golang` 实现：

```go

func GenBaiduDef(dl []codeAndWords) []byte {
    var buf bytes.Buffer
    // 首字母词条字节数统计
    lengthMap := make(map[byte]int)
    buf.Write(make([]byte, 0x6D, 0x6D))
    // utf-16le 转换
    encoder := unicode.UTF16(unicode.LittleEndian, unicode.IgnoreBOM).NewEncoder()
    for _, v := range dl {
        code := v.code
        for i, word := range v.words {
            if i != 0 { // 不在首选的写入位置信息，好像没什么用？
                code = v.code + "=" + strconv.Itoa(i+1)
            }
            sliWord, _ := encoder.Bytes([]byte(word)) // 转为utf-16le
            buf.WriteByte(byte(len(code)))            // 写编码长度
            buf.WriteByte(byte(len(sliWord) + 2))     // 写词字节长+2
            buf.WriteString(code)                     // 写编码
            buf.Write(sliWord)                        // 写词
            buf.Write([]byte{0, 0, 0, 0, 0, 0})       // 写6个0

            // 编码长度 + 词字节长 + 6，不包括长度本身占的2个字节
            lengthMap[code[0]] += len(code) + len(sliWord) + 2 + 6
        }
    }

    // 文件头
    byteList := make([]byte, 0, 0x6D)
    byteList = append(byteList, 0) // 第一个字节可能是最大码长？
    // 长度累加
    var currNum int
    for i := 0; i <= 26; i++ {
        currNum += lengthMap[byte(i+0x60)]
        // 不知道怎么来的，反正就这样算
        currBytes := []byte{byte(currNum % 0x100), byte((currNum / 0x100) % 0x100),
            byte((currNum / 0x10000) % 0x100), byte((currNum / 0x1000000) % 0x100)}
        byteList = append(byteList, currBytes...)
    }
    // 替换文件头
    ret := buf.Bytes()
    for i := 0; i < len(byteList); i++ {
        ret[i] = byteList[i]
    }
    return ret
}

```
