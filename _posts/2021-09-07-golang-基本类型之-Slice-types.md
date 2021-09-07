---
title: "golang 基本类型之 Slice types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*slice type* 切片，可以粗暴的理解为长度可变的数组

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Slice_types](https://golang.google.cn/ref/spec#Slice_types)

A slice is a descriptor for a contiguous segment of an underlying array and provides access to a numbered sequence of elements from that array. A slice type denotes the set of all slices of arrays of its element type. The value of an uninitialized slice is nil.

## 类型信息

除了_type里面记录的信息以外，还包括元素类型。具体见：src/runtime/type.go

```go
type slicetype struct {
	typ  _type
	elem *_type
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

连续三个word，分别存放指向底层数组的指针、切片长度、切片容量。具体见：src/runtime/slice.go

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

## 具体案例

```go
package main

import "fmt"

func main() {
	a := []int{1,2,3,4}
	fmt.Println(a)
}
```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=362 args=0x0 locals=0x90 funcid=0x0
...
	0x002f 00047 (main.go:6)	LEAQ	type.[4]int(SB), AX
	0x0036 00054 (main.go:6)	MOVQ	AX, (SP)
	0x003a 00058 (main.go:6)	PCDATA	$1, $0
	0x003a 00058 (main.go:6)	CALL	runtime.newobject(SB)
	0x003f 00063 (main.go:6)	MOVQ	8(SP), AX
	0x0044 00068 (main.go:6)	MOVQ	AX, ""..autotmp_2+64(SP)
	0x0049 00073 (main.go:6)	MOVQ	$1, (AX)
	0x0050 00080 (main.go:6)	MOVQ	""..autotmp_2+64(SP), AX
	0x0055 00085 (main.go:6)	TESTB	AL, (AX)
	0x0057 00087 (main.go:6)	MOVQ	$2, 8(AX)
	0x005f 00095 (main.go:6)	MOVQ	""..autotmp_2+64(SP), AX
	0x0064 00100 (main.go:6)	TESTB	AL, (AX)
	0x0066 00102 (main.go:6)	MOVQ	$3, 16(AX)
	0x006e 00110 (main.go:6)	MOVQ	""..autotmp_2+64(SP), AX
	0x0073 00115 (main.go:6)	TESTB	AL, (AX)
	0x0075 00117 (main.go:6)	MOVQ	$4, 24(AX)
	0x007d 00125 (main.go:6)	MOVQ	""..autotmp_2+64(SP), AX
	0x0082 00130 (main.go:6)	TESTB	AL, (AX)
	0x0084 00132 (main.go:6)	JMP	134
	0x0086 00134 (main.go:6)	MOVQ	AX, "".a+88(SP)
	0x008b 00139 (main.go:6)	MOVQ	$4, "".a+96(SP)
	0x0094 00148 (main.go:6)	MOVQ	$4, "".a+104(SP)
...
	0x00e2 00226 (main.go:7)	LEAQ	type.[]int(SB), DX
...
type.[]int SRODATA dupok size=56
	0x0000 18 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 8e 66 f9 1b 02 08 08 17 00 00 00 00 00 00 00 00  .f..............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*[]int-+0
	rel 44+4 t=6 type.*[]int+0
	rel 48+8 t=1 type.int+0
...
```

main.go:6 对应的汇编 00047 ~ 00130 是一个创建位于heap上的 [4]int 的过程，00134 ~ 00148 三行正好对应的是值结构里面的 slice.array、slice.len、slice.cap。

后面的 type.[]int 里面：

size=56 对应的是 src/runtime/type.go 里的 slicetype，大小正好是56。

0+8  对应的是 slicetype.typ.size，其二进制的值 0x18 即 24，表示值结构的大小。

32+8  对应的是 slicetype.typ.gcdata，它对应的 runtime.gcbits.01，表示值结构的第一个word是个指针。

48+8  对应的是 slicetype.elem，它指的是元素类型，这里是 type.int。

### 看 slice 内部指向底层数组的指针

```go
package main

import (
	"fmt"
	"unsafe"
)

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

func main() {
	base := [7]int{1, 2, 3, 4, 5, 6, 7}

	//声明与初始化
	for {
		fmt.Println("声明")
		var a []int
		sp := (*slice)(unsafe.Pointer(&a))
		if a == nil {
			fmt.Println("a is nil")
		}
		fmt.Println(a, sp.len, sp.cap, sp.array) // [] 0 0 <nil>
		fmt.Println()

		fmt.Println("初始化--不带容量")
		b := make([]int, 2)
		sp = (*slice)(unsafe.Pointer(&b))
		fmt.Println(b, sp.len, sp.cap, sp.array) // [0 0] 2 2 0xc00007e010
		fmt.Println()

		fmt.Println("初始化--带容量")
		c := make([]int, 2, 3)
		sp = (*slice)(unsafe.Pointer(&c))
		fmt.Println(c, sp.len, sp.cap, sp.array) // [0 0] 2 3 0xc00001a0c0
		fmt.Println()

		fmt.Println("初始化--基于数组的引用")
		d := new([10]int)[:5]
		sp = (*slice)(unsafe.Pointer(&d))
		fmt.Println(d, sp.len, sp.cap, sp.array) // [0 0 0 0 0] 5 10 0xc00001e0a0
		fmt.Println()

		fmt.Println("初始化--带元素")
		e := []int{1, 2, 3}
		sp = (*slice)(unsafe.Pointer(&e))
		fmt.Println(e, sp.len, sp.cap, sp.array) // [1 2 3] 3 3 0xc000098000
		fmt.Println()

		fmt.Println("初始化--基于slice")
		f := e[0:2]
		sp = (*slice)(unsafe.Pointer(&f))
		fmt.Println(f, sp.len, sp.cap, sp.array) // [1 2] 2 3 0xc000098000
		fmt.Println()

		fmt.Println("初始化--基于数组")
		g := base[0:2]
		sp = (*slice)(unsafe.Pointer(&g))
		fmt.Println(g, sp.len, sp.cap, sp.array) // [1 2] 2 7 0xc00008a000

		fmt.Println("\n")
		break
	}

	//slice只能用在可取地址的值上
	for {
		//_ = [10]int{}[:5]
		break
	}

	//查看slice底层数组
	for {
		fmt.Println("查看slice底层数组")
		a := base[2:6]
		sp := (*slice)(unsafe.Pointer(&a))
		fmt.Printf("len: %d %d\n", len(a), sp.len) // len: 4 4
		fmt.Printf("cap: %d %d\n", cap(a), sp.cap) // cap: 5 5
		fmt.Printf("base address: %p %p\n", &base[2], sp.array)
		fmt.Println()

		fmt.Println("新增元素数量不超过cap，底层数组不变")
		a = append(a, 8)
		fmt.Printf("len: %d %d\n", len(a), sp.len) // len: 5 5
		fmt.Printf("cap: %d %d\n", cap(a), sp.cap) // cap: 5 5
		fmt.Printf("base address: %p %p\n", &base[2], sp.array)
		fmt.Println()

		fmt.Println("新增元素数量超过cap，底层数组已经变了")
		a = append(a, 9)
		fmt.Printf("len: %d %d\n", len(a), sp.len) // len: 6 6
		fmt.Printf("cap: %d %d\n", cap(a), sp.cap) // cap: 10 10
		fmt.Printf("base address: %p %p\n", &base[2], sp.array)
		fmt.Println()

		fmt.Println("新增元素数量继续超过cap，注意观察cap增大的规律：倍增")
		a = append(a, 10, 11, 12, 13, 14)
		fmt.Printf("len: %d %d\n", len(a), sp.len) // len: 11 11
		fmt.Printf("cap: %d %d\n", cap(a), sp.cap) // cap: 20 20
		fmt.Printf("base address: %p %p\n", &base[2], sp.array)

		fmt.Println("\n")
		break
	}
}
```

### string与[]byte的相互转化

1. 标准类型转换方式，此方式会产生内存复制，但很安全，也容易理解。
2. 使用unsafe.Point转，此方式会复用内存，在高频场景下效率更高。不过从 string 转 []byte 以后，不要对 []byte 做append，可能会报错的。

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	//1. 标准类型转换方式，此方式会产生内存复制，但很安全，也容易理解。
	//string to []byte
	var s_1_1 string = "12345"
	var b_1_1 []byte = []byte(s_1_1)
	fmt.Println(s_1_1, b_1_1, getAddress(s_1_1), getAddress(b_1_1)) //12345 [49 50 51 52 53] 0x10c879b 0xc000094000
	//[]byte to string
	var b_1_2 []byte = []byte{'1', '2', '3', '4', '5'}
	var s_1_2 string = string(b_1_2)
	fmt.Println(s_1_2, b_1_2, getAddress(s_1_2), getAddress(b_1_2)) //12345 [49 50 51 52 53] 0xc000094040 0xc000094008

	//2. 使用unsafe.Point转，此方式会复用内存，在高频场景下效率更高。不过从 string 转 []byte 以后，不要对 []byte 做append，可能会报错的。
	//string to []byte
	var s_2_1 string = "12345"
	var b_2_1 []byte = *(*[]byte)(unsafe.Pointer(&s_2_1))
	fmt.Println(s_2_1, b_2_1, getAddress(s_2_1), getAddress(b_2_1)) //12345 [49 50 51 52 53] 0x10c879b 0x10c879b
	//[]byte to string
	var b_2_2 []byte = []byte{'1', '2', '3', '4', '5'}
	var s_2_2 string = *(*string)(unsafe.Pointer(&b_2_2))
	fmt.Println(s_2_2, b_2_2, getAddress(s_2_2), getAddress(b_2_2)) //12345 [49 50 51 52 53] 0xc000094079 0xc000094079
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

#### 从 string 转 []byte 以后，对 []byte 做append可能会报错

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	a := struct{s string; cap int}{"12345", 2}
	var b1 []byte = *(*[]byte)(unsafe.Pointer(&a.s))
	println(a.s, b1)    //12345 [5/2]0x10afc4a
	_ = append(b1, '6') //不报错

	var s2 string = "12345"
	var b2 []byte = *(*[]byte)(unsafe.Pointer(&s2))
	println(s2, b2)     //12345 [5/824634200016]0x10afc4a
	_ = append(b2, '6') //报错
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
```
