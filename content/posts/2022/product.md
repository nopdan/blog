---
title: "golang 实现笛卡尔积（泛型）"
date: 2022-06-05T21:24:25+08:00
draft: true
---

## 背景

input:  
`[[a,b],[c],[d,e]]`

output:  
`[[a,c,d],[a,c,e],[b,c,d],[b,c,e]]`

## 思路：分治

预处理第一项：`[a,b] -> [[a],[b]]`

`[[a],[b]]` 和 `[c]` -> `[[a,c],[b,c]]`

`[[a,c],[b,c]]` 和 `[d,e]` -> `[[a,c,d],[a,c,e],[b,c,d],[b,c,e]]`

## 代码

```go

// 笛卡尔积
func Product[T any](sli [][]T) [][]T {
	if len(sli) == 0 {
		return sli
	}
	ret := make([][]T, 0, len(sli[0]))
	for i := 0; i < len(sli[0]); i++ {
		ret = append(ret, []T{sli[0][i]})
	}
	for i := 1; i < len(sli); i++ {
		ret = product(ret, sli[i])
	}
	return ret
}

// 分治
func product[T any](sli [][]T, curr []T) [][]T {
	ret := make([][]T, 0, len(sli)*len(curr))
	for i := 0; i < len(sli); i++ {
		for j := 0; j < len(curr); j++ {
			tmp := make([]T, len(sli[i]))
			copy(tmp, sli[i])
			ret = append(ret, append(tmp, curr[j]))
		}
	}
	return ret
}

```
