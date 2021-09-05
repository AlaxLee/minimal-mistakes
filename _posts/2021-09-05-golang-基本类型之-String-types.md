---
title: "golang 基本类型之 String types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*string type* 就是字符串

# [String types](https://golang.google.cn/ref/spec#String_types)

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#String_types](https://golang.google.cn/ref/spec#String_types)

A string type represents the set of string values. A string value is a (possibly empty) sequence of bytes. Strings are immutable: once created, it is impossible to change the contents of a string. The predeclared string type is string; it is a defined type.

The length of a string s (its size in bytes) can be discovered using the built-in function len. The length is a compile-time constant if the string is a constant. A string's bytes can be accessed by integer indices 0 through len(s)-1. It is illegal to take the address of such an element; if s[i] is the i'th byte of a string, **&s[i] is invalid.**

## 类型信息

没有额外附加信息，具体见：src/builtin/builtin.go

```go
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
```

## 值结构

连续两个word，一个指向byte数组的指针和一个字符串长度。具体见：src/runtime/string.go

```go
//string 的真实结构
type stringStruct struct {
	str unsafe.Pointer
	len int
}

//string 的实例化函数，可见 stringStruct.str 是个 byte的数组
//go:nosplit
func gostringnocopy(str *byte) string {
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
	s := *(*string)(unsafe.Pointer(&ss))
	return s
}
```

## 具体案例

```go
package main

import "fmt"

func main() {
	a := "hello, world!"
	fmt.Println(a)
}
```

```assembly
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=236 args=0x0 locals=0x80 funcid=0x0
...
	0x0021 00033 (main.go:6)	LEAQ	go.string."hello, world!"(SB), AX
	0x0028 00040 (main.go:6)	MOVQ	AX, "".a+64(SP)
	0x002d 00045 (main.go:6)	MOVQ	$13, "".a+72(SP)
...
	0x0076 00118 (main.go:7)	LEAQ	type.string(SB), DX
...
go.string."hello, world!" SRODATA dupok size=13
	0x0000 68 65 6c 6c 6f 2c 20 77 6f 72 6c 64 21           hello, world!
...
```

从汇编对应 main.go:6 的三行可以看到，变量a确实包含一个指向字符串符号名的地址和一个字符串长度。

### 验证

```go
package main

import (
	"fmt"
	"unsafe"
)

type stringStruct struct {
	str unsafe.Pointer
	len int
}

func main() {
	var a string = "12345"
	b := (*stringStruct)(unsafe.Pointer(&a))
	fmt.Printf("%p, %p, %d\n", &a, b.str, b.len)
	haha(a)
}

func haha(s string) {
	b := (*stringStruct)(unsafe.Pointer(&s))
	fmt.Printf("%p, %p, %d\n", &s, b.str, b.len)
}
/*
执行结果：
0xc000082030, 0x10c7b59, 5
0xc00008e000, 0x10c7b59, 5
说明 string 底层是个结构体，传参的时候直接传值即可，底层字符串是不会被复制的
*/
```

### 相同值的字符串的底层指针指向同一个地址

```go
package main

import (
	"fmt"
	"unsafe"
)


func main() {
	la := "12345"
	le := "12345"
	fmt.Println(getAddress(la), getAddress(le))  //0x10c7f5b 0x10c7f5b
}

type stringStruct struct {
	str unsafe.Pointer
	len int
}

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

func getAddress(i interface{}) string {
	switch a := i.(type) {
	case string:
		ss := (*stringStruct)(unsafe.Pointer(&a))
		return fmt.Sprintf("%p", ss.str)
	case []byte:
		bs := (*slice)(unsafe.Pointer(&a))
		return fmt.Sprintf("%p", bs.array)
	default:
		return "nil"
	}
} 
```

另外，从汇编的角度也可以得出这个结论，过程略。
