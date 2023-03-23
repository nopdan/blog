---
title: '输入法词库解析（二）搜狗拼音细胞词库.scel(.qcel)'
date: 2022-05-24T15:39:17+08:00
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

`.scel` 是搜狗拼音输入法所使用的细胞词库格式，可以在 <https://pinyin.sogou.com/dict/> 下载。

`.qcel` 是 QQ 拼音输入法 6.0 以上版本所用的词库格式，可以在 <http://cdict.qq.pinyin.cn/> 下载。

## 解析

| #   | 范围           | 描述                         |
| :-- | :------------- | :--------------------------- |
|     | 0x00 - 0x11F   | 未知                         |
| a   | 0x120 - 0x123  | 不展开重码的词条数（编码数） |
| b   | 0x124 - 0x127  | 展开重码的词条数（词数）     |
|     | 0x128 - 0x12B  | 未知，和 a 有关              |
|     | 0x12C - 0x12F  | 未知，和 b 有关              |
|     | 0x130 - 0x337  | 词库名                       |
|     | 0x338 - 0x53F  | 地点？                       |
|     | 0x540 - 0xD3F  | 备注                         |
|     | 0xD40 - 0x153F | 示例词                       |

### 拼音表

从 0x1540 开始。

![](https://tucang.cc/api/image/show/7c7ba509a61717b904ec30fea41fb488)

前两个字节是拼音表的长度。这里 `9D 01` 就表示有 `0x100 * 0x01 + 0x9D = 413` 组。

后两个字节意义不明，一般是 0。

从 0x1544 开始就是拼音表正文部分。

| #   | 占用字节数 | 描述                                     |
| :-- | :--------- | :--------------------------------------- |
|     | 2          | 索引，从 `00 00` 到 `9C 01`              |
| a   | 2          | 拼音字节的长度                           |
|     | a          | 拼音，utf-16le 编码，一个字母占 2 字节。 |

**带英文词库的索引：** 从拼音表的长度往后，依次是 abcd。比如表长 413，最大索引`9D 01`，则下一个索引`9E 01`表示字母 a，依次类推。

### 词库

偏移量 0x2628

![](https://tucang.cc/api/image/show/1ccbf630635055b1a5c74949c4ef90ac)

| #   | 占用字节数 | 描述               |
| :-- | :--------- | :----------------- |
|     | 2          | 同一个音有多少词   |
| a   | 2          | 拼音索引的字节长度 |
|     | a          | 拼音索引数组       |
| b   | 2          | 词占用字节数       |
|     | b          | 词，utf-16le 编码  |
| c   | 2          | 描述信息字节长度   |
|     | c          | 描述               |

### 黑名单

一些新的 `.scel` 文件最后有一个黑名单词库。

前 12 个字节表示标识 `DELTBL`。

接下来 2 个字节表示黑名单词库词条数。

![](https://tucang.cc/api/image/show/9b4243279ec3df86ff063596f2d58b00)

| #   | 占用字节数 | 描述 |
| :-- | :--------- | :--- |
| a   | 2          | 词长 |
|     | a\*2       | 词   |

**_代码实现：_**

```go
func (SogouScel) Parse(filename string) Dict {
    data, _ := os.ReadFile(filename)
    r := bytes.NewReader(data)
    ret := make(Dict, 0, r.Len()>>8)
    var tmp []byte

    // 不展开的词条数
    r.Seek(0x120, 0)
    dictLen := ReadUint32(r)

    // 拼音表偏移量
    r.Seek(0x1540, 0)

    // 前两个字节是拼音表长度，413
    pyTableLen := ReadUint16(r)
    pyTable := make([]string, pyTableLen)
    // fmt.Println("拼音表长度", pyTableLen)

    // 丢掉两个字节
    r.Seek(2, 1)

    // 读拼音表
    for i := 0; i < pyTableLen; i++ {
        // 索引，2字节
        idx := ReadUint16(r)
        // 拼音长度，2字节
        pyLen := ReadUint16(r)
        // 拼音 utf-16le
        tmp = make([]byte, pyLen)
        r.Read(tmp)
        py, _ := util.Decode(tmp, "UTF-16LE")
        //
        pyTable[idx] = string(py)
    }

    // 读码表
    for j := 0; j < dictLen; j++ {
        // 重码数（同一串音对应多个词）
        repeat := ReadUint16(r)

        // 索引数组长
        pinyinSize := ReadUint16(r)

        // 读取编码
        var pinyin []string
        for i := 0; i < pinyinSize/2; i++ {
            theIdx := ReadUint16(r)
            if theIdx >= pyTableLen {
                pinyin = append(pinyin, string(byte(theIdx-pyTableLen+97)))
                continue
            }
            pinyin = append(pinyin, pyTable[theIdx])
        }

        // 读取一个或多个词
        for i := 1; i <= repeat; i++ {
            // 词长
            wordSize := ReadUint16(r)

            // 读取词
            tmp = make([]byte, wordSize)
            r.Read(tmp)
            word, _ := util.Decode(tmp, "UTF-16LE")

            // 末尾的补充信息，作用未知
            extSize := ReadUint16(r)
            ext := make([]byte, extSize)
            r.Read(ext)

            ret = append(ret, Entry{word, pinyin, 1})
        }
    }
    if r.Len() < 16 {
        return ret
    }

    // 黑名单
    r.Seek(12, 1)
    blackLen := ReadUint16(r)
    var black_list bytes.Buffer
    for i := 0; i < blackLen; i++ {
        wordLen := ReadUint16(r)
        tmp = make([]byte, wordLen*2)
        r.Read(tmp)
        word, _ := util.Decode(tmp, "UTF-16LE")
        black_list.WriteString(word)
        black_list.WriteByte('\n')
    }
    // os.WriteFile("black_list.txt", black_list.Bytes(), 0666)
    return ret
}
```

---

**参考资料：**

- [深蓝词库转换](https://github.com/studyzy/imewlconverter)
- [将搜狗词库(scel)转为 Python 可读的文本](https://zhuanlan.zhihu.com/p/43446135)
