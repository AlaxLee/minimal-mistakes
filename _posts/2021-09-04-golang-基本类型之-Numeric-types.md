---
title: "golang 基本类型之 Numeric types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*numeric type* 是整型和浮点型的集合

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Numeric_types](https://golang.google.cn/ref/spec#Numeric_types)

A *numeric type* represents sets of integer or floating-point values. The predeclared architecture-independent numeric types are:

```
#如下与平台无关
uint8       the set of all unsigned  8-bit integers (0 to 255)
uint16      the set of all unsigned 16-bit integers (0 to 65535)
uint32      the set of all unsigned 32-bit integers (0 to 4294967295)
uint64      the set of all unsigned 64-bit integers (0 to 18446744073709551615)

int8        the set of all signed  8-bit integers (-128 to 127)
int16       the set of all signed 16-bit integers (-32768 to 32767)
int32       the set of all signed 32-bit integers (-2147483648 to 2147483647)
int64       the set of all signed 64-bit integers (-9223372036854775808 to 9223372036854775807)

float32     the set of all IEEE-754 32-bit floating-point numbers
float64     the set of all IEEE-754 64-bit floating-point numbers

complex64   the set of all complex numbers with float32 real and imaginary parts
complex128  the set of all complex numbers with float64 real and imaginary parts

byte        alias for uint8
rune        alias for int32
```

There is also a set of predeclared numeric types with implementation-specific sizes:

```
#如下与平台相关
uint        either 32 or 64 bits
int         same size as uint
uintptr     an unsigned integer large enough to store the uninterpreted bits of a pointer value
```



## 类型信息

没有额外附加信息，具体见：src/builtin/builtin.go

```go
package builtin

// uint8 is the set of all unsigned 8-bit integers.
// Range: 0 through 255.
type uint8 uint8

// uint16 is the set of all unsigned 16-bit integers.
// Range: 0 through 65535.
type uint16 uint16

// uint32 is the set of all unsigned 32-bit integers.
// Range: 0 through 4294967295.
type uint32 uint32

// uint64 is the set of all unsigned 64-bit integers.
// Range: 0 through 18446744073709551615.
type uint64 uint64

// int8 is the set of all signed 8-bit integers.
// Range: -128 through 127.
type int8 int8

// int16 is the set of all signed 16-bit integers.
// Range: -32768 through 32767.
type int16 int16

// int32 is the set of all signed 32-bit integers.
// Range: -2147483648 through 2147483647.
type int32 int32

// int64 is the set of all signed 64-bit integers.
// Range: -9223372036854775808 through 9223372036854775807.
type int64 int64

// float32 is the set of all IEEE-754 32-bit floating-point numbers.
type float32 float32

// float64 is the set of all IEEE-754 64-bit floating-point numbers.
type float64 float64

// complex64 is the set of all complex numbers with float32 real and
// imaginary parts.
type complex64 complex64

// complex128 is the set of all complex numbers with float64 real and
// imaginary parts.
type complex128 complex128

// int is a signed integer type that is at least 32 bits in size. It is a
// distinct type, however, and not an alias for, say, int32.
type int int

// uint is an unsigned integer type that is at least 32 bits in size. It is a
// distinct type, however, and not an alias for, say, uint32.
type uint uint

// uintptr is an integer type that is large enough to hold the bit pattern of
// any pointer.
type uintptr uintptr

// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```

## 值结构

直接存放在内存里，占用空间不一定相同。

## 具体案例

```go
package main

import "fmt"

func main() {
	var a uint8 = 1
	var b uint16 = 2
	var c uint32 = 3
	var d uint64 = 4
	var e int8 = 5
	var f int16 = 6
	var g int32 = 7
	var h int64 = 8
	var i float32 = 9.1
	var j float64 = 10.2
	var k complex64 = 11.3 + 12.4i
	var l complex128 = 13.5 + 14.6i
	var m int = 15
	var n uint = 16
	var o uintptr = 17
	var p byte = 18
	var q rune = 19
	fmt.Println(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q)
}
```

```assembly
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=1940 args=0x0 locals=0x248 funcid=0x0
...
	0x0032 00050 (main.go:6)	MOVB	$1, "".a+56(SP)
	0x0037 00055 (main.go:7)	MOVW	$2, "".b+62(SP)
	0x003e 00062 (main.go:8)	MOVL	$3, "".c+76(SP)
	0x0046 00070 (main.go:9)	MOVQ	$4, "".d+128(SP)
	0x0052 00082 (main.go:10)	MOVB	$5, "".e+55(SP)
	0x0057 00087 (main.go:11)	MOVW	$6, "".f+60(SP)
	0x005e 00094 (main.go:12)	MOVL	$7, "".g+72(SP)
	0x0066 00102 (main.go:13)	MOVQ	$8, "".h+120(SP)
	0x006f 00111 (main.go:14)	MOVSS	$f32.4111999a(SB), X1
	0x0077 00119 (main.go:14)	MOVSS	X1, "".i+68(SP)
	0x007d 00125 (main.go:15)	MOVSD	$f64.4024666666666666(SB), X1
	0x0085 00133 (main.go:15)	MOVSD	X1, "".j+112(SP)
	0x008b 00139 (main.go:16)	MOVSS	$f32.4134cccd(SB), X1
	0x0093 00147 (main.go:16)	MOVSS	X1, "".k+104(SP)
	0x0099 00153 (main.go:16)	MOVSS	$f32.41466666(SB), X1
	0x00a1 00161 (main.go:16)	MOVSS	X1, "".k+108(SP)
	0x00a7 00167 (main.go:17)	MOVSD	$f64.402b000000000000(SB), X1
	0x00af 00175 (main.go:17)	MOVSD	X1, "".l+144(SP)
	0x00b8 00184 (main.go:17)	MOVSD	$f64.402d333333333333(SB), X1
	0x00c0 00192 (main.go:17)	MOVSD	X1, "".l+152(SP)
	0x00c9 00201 (main.go:18)	MOVQ	$15, "".m+96(SP)
	0x00d2 00210 (main.go:19)	MOVQ	$16, "".n+88(SP)
	0x00db 00219 (main.go:20)	MOVQ	$17, "".o+80(SP)
	0x00e4 00228 (main.go:21)	MOVB	$18, "".p+54(SP)
	0x00e9 00233 (main.go:22)	MOVL	$19, "".q+64(SP)
...
$f32.4111999a SRODATA size=4
	0x0000 9a 99 11 41                                      ...A
$f32.4134cccd SRODATA size=4
	0x0000 cd cc 34 41                                      ..4A
$f32.41466666 SRODATA size=4
	0x0000 66 66 46 41                                      ffFA
$f64.4024666666666666 SRODATA size=8
	0x0000 66 66 66 66 66 66 24 40                          ffffff$@
$f64.402b000000000000 SRODATA size=8
	0x0000 00 00 00 00 00 00 2b 40                          ......+@
$f64.402d333333333333 SRODATA size=8
	0x0000 33 33 33 33 33 33 2d 40                          333333-@
```

由上述汇编可知，他们都是直接以数值的形式存放在内存里

