---
title: "golang 基本类型之 Boolean types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*boolean type* 布尔值，即 true or false

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Boolean_types](https://golang.google.cn/ref/spec#Boolean_types)

A boolean type represents the set of Boolean truth values denoted by the predeclared constants true and false. The predeclared boolean type is bool; it is a [defined type](https://golang.google.cn/ref/spec#Type_definitions).

## 类型信息

没有额外附加信息，具体见：src/builtin/builtin.go

```go
package builtin

// bool is the set of boolean values, true and false.
type bool bool

// true and false are the two untyped boolean values.
const (
	true  = 0 == 0 // Untyped bool.
	false = 0 != 0 // Untyped bool.
)
```

## 值结构

在内存里，1 表示 true，0 表示 false

## 具体案例

```go
package main

import "fmt"

func main() {
	var a bool
	a = false
	a = true
	fmt.Println(a)
}
```

```assembly
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=185 args=0x0 locals=0x70 funcid=0x0
...
	0x0021 00033 (main.go:6)	MOVB	$0, "".a+54(SP)
	0x0026 00038 (main.go:7)	MOVB	$0, "".a+54(SP)
	0x002b 00043 (main.go:8)	MOVB	$1, "".a+54(SP)
...
	0x004e 00078 (main.go:9)	LEAQ	type.bool(SB), DX
...
```

在汇编结果里，从第4、5、6行，可见 bool 的默认值是false，false 在内存里是 0，true 在内存里是 1。

从第8行，可知bool的符号就是 type.bool。
