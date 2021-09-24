---
title: "golang 基本类型之 Function types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*function type* 函数类型，通俗的来说，就是可以将一个变量声明为函数。

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Function_types](https://golang.google.cn/ref/spec#Function_types)

A function type denotes the set of all functions with the same parameter and result types. The value of an uninitialized variable of function type is `nil`.

## 类型信息

包括 functype、uncommontype（这个可能有也可能没有）、输入参数类型列表和输出参数类型列表。

具体见：src/runtime/type.go。另外，也可以参考 src/reflect/type.go，这里描述的结构差不多，且注释更多。

```go
type functype struct {
	typ      _type
	inCount  uint16
	outCount uint16 // top bit is set if last input parameter is ...
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

// 从如下的 in 和 out 方法可知，functype 后面会跟着记录输入参数和输出参数类型

func (t *functype) in() []*_type {
	// See funcType in reflect/type.go for details on data layout.
	uadd := uintptr(unsafe.Sizeof(functype{}))
	if t.typ.tflag&tflagUncommon != 0 {
		uadd += unsafe.Sizeof(uncommontype{})
	}
	return (*[1 << 20]*_type)(add(unsafe.Pointer(t), uadd))[:t.inCount]
}

func (t *functype) out() []*_type {
	// See funcType in reflect/type.go for details on data layout.
	uadd := uintptr(unsafe.Sizeof(functype{}))
	if t.typ.tflag&tflagUncommon != 0 {
		uadd += unsafe.Sizeof(uncommontype{})
	}
	outCount := t.outCount & (1<<15 - 1)
	return (*[1 << 20]*_type)(add(unsafe.Pointer(t), uadd))[t.inCount : t.inCount+outCount]
}

func (t *functype) dotdotdot() bool { // outCount's top bit is set if last input parameter is ...
	return t.outCount&(1<<15) != 0
}

type uncommontype struct {
	pkgpath nameOff
	mcount  uint16 // number of methods
	xcount  uint16 // number of exported methods
	moff    uint32 // offset from this uncommontype to [mcount]method
	_       uint32 // unused
}
```

## 值结构

是一个指针，指向函数体的地址

## 具体案例

### 输入参数不含 ...

```go
package main

import (
	"fmt"
)

func lala(i int, j uint, k string) (string, []interface{}) {
	return fmt.Sprintf("%d%d%s", i, j, k), []interface{}{i, j, k}
}

func main() {
	var a func(int, uint, string) (string, []interface{})
	a = lala
	fmt.Println(a)
}
```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".lala STEXT size=916 args=0x48 locals=0xf0 funcid=0x0
...
"".main STEXT size=175 args=0x0 locals=0x78 funcid=0x0
...
	0x0021 00033 (main.go:12)	MOVQ	$0, "".a+48(SP)   # main.go:12  var a func(int, uint, string) (string, []interface{})
	0x002a 00042 (main.go:13)	LEAQ	"".lala·f(SB), AX # main.go:13  a = lala
	0x0031 00049 (main.go:13)	MOVQ	AX, "".a+48(SP)
...
	0x0054 00084 (main.go:14)	LEAQ	type.func(int, uint, string) (string, []interface {})(SB), DX
...
"".lala·f SRODATA dupok size=8
	0x0000 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 "".lala+0
...
type.func(int, uint, string) (string, []interface {}) SRODATA dupok size=96
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 d0 de a0 d9 02 08 08 33 00 00 00 00 00 00 00 00  .......3........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 03 00 02 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func(int, uint, string) (string, []interface {})-+0
	rel 44+4 t=6 type.*func(int, uint, string) (string, []interface {})+0
	rel 56+8 t=1 type.int+0
	rel 64+8 t=1 type.uint+0
	rel 72+8 t=1 type.string+0
	rel 80+8 t=1 type.string+0
	rel 88+8 t=1 type.[]interface {}+0
...
```

从 `"".main` 里面的 0x002a、0x0031 行可以看出来，函数的值结构存储的是一个指向`函数名·f`符号的地址，而`函数名·f`符号指向的地址存储的是原函数的符号的地址。也可粗暴的认为，函数的值结构存放的是指向函数地址的指针。

`type.func(int, uint, string) (string, []interface {})` ：

- size=96，对应 functype 和 输入参数类型、输出参数类型总大小，56 + 3 * 8 + 2 * 8 = 96 正好对的上
- 0+8，functype.typ.size  表示函数值结构大小是8
- 8+8，functype.typ.ptrdata 值是8，表示值结构前8个字节存放有指针，与值结构存放地址的说法一致
- 32+8，functype.typ.gcdata 表示值结构的那些word存放着指针，这里值是 runtime.gcbits.01，01 的 二进制是 01，表示值结构只有第一个word存放的指针，与值结构实际情况一致
- 48+2，functype.inCount 表示输入参数个数，这里值为3，与实际情况一致
- 50+2，functype.outCount，2字节16位，最高位表示输入参数是否包含`...`，低15位表示输出参数个数。这里值为2，其二进制位最高位是0，低15位值为2，分别表示输入参数里面没有`...`，输出参数个数为2，与实际情况一致
- 56+24，依次存放的是输入参数类型，`type.int`、`type.uint`、`type.string`
- 80+16，依次存放的是输出参数类型，`type.string`、`type.[]interface {}`

`"".lala·f` 这个符号的生成，是由 src/cmd/compile/internal/gc/dcl.go 的 funcsym 函数实现的。

```go
func funcsymname(s *types.Sym) string {
	return s.Name + "·f"
}

// funcsym returns s·f.
func funcsym(s *types.Sym) *types.Sym {
	// funcsymsmu here serves to protect not just mutations of funcsyms (below),
	// but also the package lookup of the func sym name,
	// since this function gets called concurrently from the backend.
	// There are no other concurrent package lookups in the backend,
	// except for the types package, which is protected separately.
	// Reusing funcsymsmu to also cover this package lookup
	// avoids a general, broader, expensive package lookup mutex.
	// Note makefuncsym also does package look-up of func sym names,
	// but that it is only called serially, from the front end.
	funcsymsmu.Lock()
	sf, existed := s.Pkg.LookupOK(funcsymname(s))
	// Don't export s·f when compiling for dynamic linking.
	// When dynamically linking, the necessary function
	// symbols will be created explicitly with makefuncsym.
	// See the makefuncsym comment for details.
	if !Ctxt.Flag_dynlink && !existed {
		funcsyms = append(funcsyms, s)
	}
	funcsymsmu.Unlock()
	return sf
}
```

另外，部分系统函数（主要是hash和equal类）的 `·f` 格式是由 src/cmd/compile/internal/gc/alg.go 的 `func sysClosure(name string) *obj.LSym` 方法实现的，具体 sysClosure 实现了哪些系统函数，可以在源码里看到。

### 输入参数含 ...

```go
package main

import (
	"fmt"
)

func lala(i int, j uint, k ...string) (string, []interface{}) {
	return fmt.Sprintf("%d%d%s", i, j, k), []interface{}{i, j, k}
}

func main() {
	var a func(int, uint, ...string) (string, []interface{})
	a = lala
	fmt.Println(a)
}
```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".lala STEXT size=943 args=0x50 locals=0xf0 funcid=0x0
...
"".main STEXT size=175 args=0x0 locals=0x78 funcid=0x0
...
	0x0021 00033 (main.go:12)	MOVQ	$0, "".a+48(SP)   # main.go:12  var a func(int, uint, ...string) (string, []interface{})
	0x002a 00042 (main.go:13)	LEAQ	"".lala·f(SB), AX # main.go:13  a = lala
	0x0031 00049 (main.go:13)	MOVQ	AX, "".a+48(SP)
...
	0x0054 00084 (main.go:14)	LEAQ	type.func(int, uint, ...string) (string, []interface {})(SB), DX
...
"".lala·f SRODATA dupok size=8
	0x0000 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 "".lala+0
...
type.func(int, uint, ...string) (string, []interface {}) SRODATA dupok size=96
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 f8 3f 00 c0 02 08 08 33 00 00 00 00 00 00 00 00  .?.....3........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 03 00 02 80 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func(int, uint, ...string) (string, []interface {})-+0
	rel 44+4 t=6 type.*func(int, uint, ...string) (string, []interface {})+0
	rel 56+8 t=1 type.int+0
	rel 64+8 t=1 type.uint+0
	rel 72+8 t=1 type.[]string+0
	rel 80+8 t=1 type.string+0
	rel 88+8 t=1 type.[]interface {}+0
...
```

其它的都和不含 `...` 的一样，除了如下：

50+2，functype.outCount，2字节16位，最高位表示输入参数是否包含`...`，低15位表示输出参数个数。这里值为0x8002，其二进制位最高位是1，低15位值为2，分别表示输入参数里面有`...`，输出参数个数为2，与实际情况一致

