---
title: '输入法词库解析（一）百度自定义方案.def'
date: 2022-05-24T15:04:00+08:00
categories:
  - 输入法
tags:
  - 输入法
  - 词库
  - 二进制
series: [lexicon]
# draft: true
---

> 详细代码：<https://github.com/nopdan/rose>

## 前言

`.def` 是百度手机输入法-更多设置-自定义输入方案所使用的格式。

## 解析

码表偏移量 `0x6D`

![](https://tucang.cc/api/image/show/3ab4d59dda60d731eb6ddb55c7694bd5)

| #   | 占用字节数 | 描述                                               |
| --- | ---------- | -------------------------------------------------- |
| a   | 1          | 编码长度（红色框）                                 |
| b   | 1          | 词长 \* 2 + 2                                      |
|     | a          | 编码（黄色框），可以是纯编码，也可以是 `编码=位置` |
|     | b-2        | 词（绿色框），utf16-le 编码                        |
|     | 6          | 6 个空字节代表词条结束                             |

**_代码实现：_**

```go
    r.Seek(0x6D, 0) // 从 0x6D 开始读
    for r.Len() > 4 {
        codeLen, _ := r.ReadByte()  // 编码长度
        wordSize, _ := r.ReadByte() // 词长*2 + 2

        // 读编码
        tmp = make([]byte, int(codeLen))
        r.Read(tmp) // 编码切片
        code := string(tmp)
        spl := strings.Split(code, "=") // 直接删掉 = 号后的
        code = spl[0]

        // 读词
        tmp = make([]byte, int(wordSize)-2) // -2 后就是字节长度，没有考虑4字节的情况
        r.Read(tmp)
        word, _ := util.Decode(tmp, "UTF-16LE")
        // def = append(def, defEntry{word, code, order})
        ret = append(ret, Entry{word, code, 1})

        r.Seek(6, 1) // 6个00，1是相对当前位置
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

**_代码实现：_**

```go
func (BaiduDef) Gen(table Table) []byte {
    jdt := ToJdTable(table)
    var buf bytes.Buffer
    // 首字母词条字节数统计
    lengthMap := make(map[byte]int)
    buf.Write(make([]byte, 0x6D))

    for _, v := range jdt {
        code := v.Code

        for i, word := range v.Words {
            if i != 0 { // 不在首选的写入位置信息，好像没什么用？
                code = v.Code + "=" + strconv.Itoa(i+1)
            }
            sliWord, _ := util.Encode([]byte(word), "UTF-16LE") // 转为utf-16le
            buf.WriteByte(byte(len(code)))                      // 写编码长度
            buf.WriteByte(byte(len(sliWord) + 2))               // 写词字节长+2
            buf.WriteString(code)                               // 写编码
            buf.Write(sliWord)                                  // 写词
            buf.Write([]byte{0, 0, 0, 0, 0, 0})                 // 写6个0

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
        currBytes := make([]byte, 4)
        binary.LittleEndian.PutUint32(currBytes, uint32(currNum))
        byteList = append(byteList, currBytes...)
    }
    // 替换文件头
    ret := buf.Bytes()
    copy(ret, byteList)
    return ret
}
```

---

**参考资料：**

[DictTool 词库处理工具](https://github.com/asd2fque1/DictTool)
