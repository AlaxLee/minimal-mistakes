---
title: "golang 基本类型之 Array types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*array type* 数组

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Array_types](https://golang.google.cn/ref/spec#Array_types)

An array is a numbered sequence of elements of a single type, called the element type. The number of elements is called the length of the array and is never negative.

## 类型信息

除了_type里面记录的信息以外，还包括元素类型、对应的slice类型和数组长度。具体见：src/runtime/type.go

```go
type arraytype struct {
	typ   _type
	elem  *_type
	slice *_type
	len   uintptr
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

它在内存里就是一段连续的内存

## 具体案例

```go
package main

import "fmt"

func main() {
	a := [3]int{1, 2, 3}
	fmt.Println(a)
}
```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=305 args=0x0 locals=0x98 funcid=0x0
...
	0x002f 00047 (main.go:6)	MOVQ	$0, "".a+48(SP)
	0x0038 00056 (main.go:6)	XORPS	X0, X0
	0x003b 00059 (main.go:6)	MOVUPS	X0, "".a+56(SP)
	0x0040 00064 (main.go:6)	MOVQ	$1, "".a+48(SP)
	0x0049 00073 (main.go:6)	MOVQ	$2, "".a+56(SP)
	0x0052 00082 (main.go:6)	MOVQ	$3, "".a+64(SP)
...
	0x0088 00136 (main.go:7)	LEAQ	type.[3]int(SB), AX
...
type.[3]int SRODATA dupok size=72
	0x0000 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 63 c4 1a 46 0a 08 08 11 00 00 00 00 00 00 00 00  c..F............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 03 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 type..eqfunc24+0
	rel 32+8 t=1 runtime.gcbits.+0
	rel 40+4 t=5 type..namedata.*[3]int-+0
	rel 44+4 t=6 type.*[3]int+0
	rel 48+8 t=1 type.int+0
	rel 56+8 t=1 type.[]int+0
...
```

其 main.go:6 的语句可以看出来，数组是一段连续的内存。

由于 [3]int 并非内置类型，所以需要额外构建，即接下来的 type.[3]int

size=72 对应的是 src/runtime/type.go 里的 arraytype，大小正好是72。

0+8  对应的是 arraytype.typ.size，其二进制的值 0x18 即 24，表示值结构的大小（3个int，每个8字节，正好24）。

32+8  对应的是 arraytype.typ.gcdata，它对应的 runtime.gcbits.，表示值结构里面不存在指针。

48+8  对应的是 arraytype.elem，它指的是元素类型，这里是 type.int。

56+8 对应的是 arraytype.slice，但不知道这个有啥用。

64+8 对应的是 arraytype.len，二进制里面的值是 3，与 [3]int 的len一致。

### 初始化

```go
package main

import "fmt"

const n int = 3

func main() {
	a := [3]string{}
	fmt.Println(a)   // [  ]   三个 ""
	var b [2*n]int = [2*n]int{}
	fmt.Println(b)   // [0 0 0 0 0 0]
	c := [0]int{}
	fmt.Println(c)   // []
	d := [...]int{1,2,3}
	fmt.Println(d)   // [1, 2, 3]
  e := [5]int{2:1,4:3}
	fmt.Println(e)   // [0 0 1 0 3]
}
```



### 把一个数组的一部分作为另一个数组

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {

	a := [5]int{}

	fmt.Println(a)

	b := (*[3]int)(unsafe.Pointer(&a[1]))
	fmt.Println(*b)

	(*b)[0] = 1
	b[1] = 2
	b[2] = 3

	fmt.Println(a, *b)
}
```



### 一维数组的底层是连续的

```go
package main

import "fmt"

func main() {
	a := [5]byte{}
	for i := 0; i < 5; i++ {
		fmt.Printf("a[%d]: 0x%x\n", i, &a[i])
	}
}
```

执行结果如下，明显可以看到是连续的

```
a[0]: 0xc000092000
a[1]: 0xc000092001
a[2]: 0xc000092002
a[3]: 0xc000092003
a[4]: 0xc000092004
```



### 多维数组底层是连续的，就像一维数组一样

官方文档里的说法是 [spec: Array types:](https://golang.google.cn/ref/spec#Array_types)

> **Array types are always one-dimensional** but may be composed to form multi-dimensional types.

举个例子：

```go
package main

import "fmt"

func main() {
	a := [2][3][3]byte{}
	for i := 0; i < 2; i++ {
		for j := 0; j < 3; j++ {
			for k := 0; k < 3; k++ {
				fmt.Printf("a[%d][%d][%d]: 0x%x\n", i, j, k, &a[i][j][k])
			}
		}
	}
}
```

执行后的结果如下，明显可以看到地址是连续的：

```
a[0][0][0]: 0xc000090000
a[0][0][1]: 0xc000090001
a[0][0][2]: 0xc000090002
a[0][1][0]: 0xc000090003
a[0][1][1]: 0xc000090004
a[0][1][2]: 0xc000090005
a[0][2][0]: 0xc000090006
a[0][2][1]: 0xc000090007
a[0][2][2]: 0xc000090008
a[1][0][0]: 0xc000090009
a[1][0][1]: 0xc00009000a
a[1][0][2]: 0xc00009000b
a[1][1][0]: 0xc00009000c
a[1][1][1]: 0xc00009000d
a[1][1][2]: 0xc00009000e
a[1][2][0]: 0xc00009000f
a[1][2][1]: 0xc000090010
a[1][2][2]: 0xc000090011
```



### 作为函数参数时，array 会发生复制。所以，不要直接拿 array 做函数参数

```go
package main

import "fmt"

func main() {
	a := [5]int{1,2,3,4,5}
	do(a)
	fmt.Println(a[3])  // 4
}

func do(b [5]int) {
	b[3] = 10
}
```



### 作为 for  :=  range 的 对象时，array 会发生复制。所以，遍历 array 不要用 for := range 方式

```go
package main

import "fmt"

func main() {
	a := [5]int{1,2,3,4,5}
	for i, v := range a {
		if i == 0 {
			a[3] = 10
		} else if i == 3 {
			fmt.Println(v, a[3])   // 4 10
		}
	}
}
```

