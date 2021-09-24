---
title: "golang 基本类型之 Pointer types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*pointer type* 指针类型，golang中的指针类型会声明指向的具体类型

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Pointer_types](https://golang.google.cn/ref/spec#Pointer_types)

A pointer type denotes the set of all pointers to variables of a given type, called the base type of the pointer. The value of an uninitialized pointer is nil.

## 类型信息

具体见：src/runtime/type.go。另外，也可以参考 src/reflect/type.go，这里描述的结构差不多，且注释更多。

```go
type ptrtype struct {
	typ  _type
	elem *_type // pointer element (pointed at) type
}
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```

## 值结构

一个指针

## 具体案例

```go
package main

import "fmt"

func main() {
	b := [3]int{1, 2, 3}
	var a *[3]int
	a = &b
	fmt.Println(a)
}

```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=222 args=0x0 locals=0x80 funcid=0x0
...
	0x0021 00033 (main.go:6)	LEAQ	type.[3]int(SB), AX   # main.go:6  b := [3]int{1, 2, 3}
	0x0028 00040 (main.go:6)	MOVQ	AX, (SP)
	0x002c 00044 (main.go:6)	PCDATA	$1, $0
	0x002c 00044 (main.go:6)	CALL	runtime.newobject(SB)
	0x0031 00049 (main.go:6)	MOVQ	8(SP), AX
	0x0036 00054 (main.go:6)	MOVQ	AX, "".&b+72(SP)
	0x003b 00059 (main.go:6)	MOVQ	$1, (AX)
	0x0042 00066 (main.go:6)	MOVQ	$2, 8(AX)
	0x004a 00074 (main.go:6)	MOVQ	$3, 16(AX)
	0x0052 00082 (main.go:7)	MOVQ	$0, "".a+48(SP)   # main.go:7  var a *[3]int
	0x005b 00091 (main.go:8)	MOVQ	"".&b+72(SP), AX  # main.go:8  a = &b
	0x0060 00096 (main.go:8)	MOVQ	AX, "".a+48(SP)
...
	0x0083 00131 (main.go:9)	LEAQ	type.*[3]int(SB), DX
...
type.*[3]int SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 25 30 9a e3 08 08 08 36 00 00 00 00 00 00 00 00  %0.....6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*[3]int-+0
	rel 48+8 t=1 type.[3]int+0
...
```

`type.*[3]int` ：

- size=56，与 ptrtype 大小正好对应上
- 0+8，ptrtype.typ.size  表示函数值结构大小是8
- 8+8，ptrtype.typ.ptrdata 值是8，表示值结构前8个字节存放有指针，与值结构存放地址的说法一致
- 32+8，ptrtype.typ.gcdata 表示值结构的哪些word存放着指针，这里值是 runtime.gcbits.01，01 的 二进制是 01，表示值结构只有第一个word存放的指针，与值结构实际情况一致
- 48+8，ptrtype.elem 标志指针指向的类型

