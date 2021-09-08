---
title: "golang 基本类型之 Struct types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---

*struct type* 结构体

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Struct_types](https://golang.google.cn/ref/spec#Struct_types)

A struct is a sequence of named elements, called fields, each of which has a name and a type. Field names may be specified **explicitly (IdentifierList)** or **implicitly (EmbeddedField)**. Within a struct, non-blank field names must be unique.

### explicitly (IdentifierList)

```go
struct {
	x, y int
	u float32
	_ float32  // padding
	A *[]int
	F func()
}
```

### implicitly (EmbeddedField)

A field declared with a type but no explicit field name is called an embedded field. An embedded field must be specified as a type name T or as a pointer to a non-interface type name *T, and **T itself may not be a pointer type**. The unqualified type name acts as the field name.

```go
// A struct with four embedded fields of types T1, *T2, P.T3 and *P.T4
struct {
	T1        // field name is T1
	*T2       // field name is T2
	P.T3      // field name is T3
	*P.T4     // field name is T4
}
struct {
	T     // conflicts with embedded field *T and *P.T
	*T    // conflicts with embedded field T and *P.T
	*P.T  // conflicts with embedded field T and *T
}
type T *int
type A struct {
    a T  //this is ok
	T    //embedded type cannot be a pointer
}
```



## 类型信息

除了_type里面记录的信息以外，还包括包路径和结构体的属性，如果类型带有方法的话，还会有 uncommontype 用来记录方法。具体见：src/runtime/type.go。另外，也可以参考 src/reflect/type.go，这里描述的结构差不多，且注释更多。

```go
type structtype struct {
	typ     _type
	pkgPath name
	fields  []structfield
}
type structfield struct {
	name       name
	typ        *_type
	offsetAnon uintptr
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

type uncommontype struct {
	pkgpath nameOff
	mcount  uint16 // number of methods
	xcount  uint16 // number of exported methods
	moff    uint32 // offset from this uncommontype to [mcount]method
	_       uint32 // unused
}
type method struct {
	name nameOff
	mtyp typeOff
	ifn  textOff
	tfn  textOff
}
type nameOff int32
type typeOff int32
type textOff int32

func (t *_type) uncommon() *uncommontype {
	if t.tflag&tflagUncommon == 0 {
		return nil
	}
	switch t.kind & kindMask {
	case kindStruct:
		type u struct {
			structtype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	...
	}
}
```

## 值结构

所有属性按定义顺序在内存排列，遵循按word对齐。

## 具体案例

```go
package main

import "fmt"

type A struct {
	B int32	`haha`
	c string	`hehe`
}

func (a *A) Lala() {
	fmt.Println(a)
}

func (a A) lele() {
	fmt.Println(a)
}

func main() {
	a := A{2, "kaka"}
	fmt.Println(a)
}
```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".(*A).Lala STEXT size=159 args=0x8 locals=0x70 funcid=0x0
...
"".A.lele STEXT size=242 args=0x18 locals=0x80 funcid=0x0
...
"".main STEXT size=300 args=0x0 locals=0x98 funcid=0x0
	0x0000 00000 (main.go:18)	TEXT	"".main(SB), ABIInternal, $152-0
	0x0000 00000 (main.go:18)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:18)	LEAQ	-24(SP), AX
	0x000e 00014 (main.go:18)	CMPQ	AX, 16(CX)
	0x0012 00018 (main.go:18)	PCDATA	$0, $-2
	0x0012 00018 (main.go:18)	JLS	290
	0x0018 00024 (main.go:18)	PCDATA	$0, $-1
	0x0018 00024 (main.go:18)	SUBQ	$152, SP
	0x001f 00031 (main.go:18)	MOVQ	BP, 144(SP)
	0x0027 00039 (main.go:18)	LEAQ	144(SP), BP
	0x002f 00047 (main.go:18)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x002f 00047 (main.go:18)	FUNCDATA	$1, gclocals·5a97ff06ec2b95a706eb55096385c78b(SB)
	0x002f 00047 (main.go:18)	FUNCDATA	$2, "".main.stkobj(SB)
	0x002f 00047 (main.go:19)	MOVL	$0, "".a+72(SP)
	0x0037 00055 (main.go:19)	XORPS	X0, X0
	0x003a 00058 (main.go:19)	MOVUPS	X0, "".a+80(SP)
	0x003f 00063 (main.go:19)	MOVL	$2, "".a+72(SP)
	0x0047 00071 (main.go:19)	LEAQ	go.string."kaka"(SB), AX
	0x004e 00078 (main.go:19)	MOVQ	AX, "".a+80(SP)
	0x0053 00083 (main.go:19)	MOVQ	$4, "".a+88(SP)
	0x005c 00092 (main.go:20)	MOVL	"".a+72(SP), CX
	0x0060 00096 (main.go:20)	MOVL	CX, ""..autotmp_1+120(SP)
	0x0064 00100 (main.go:20)	MOVQ	AX, ""..autotmp_1+128(SP)
	0x006c 00108 (main.go:20)	MOVQ	$4, ""..autotmp_1+136(SP)
	0x0078 00120 (main.go:20)	XORPS	X0, X0
	0x007b 00123 (main.go:20)	MOVUPS	X0, ""..autotmp_2+56(SP)
	0x0080 00128 (main.go:20)	LEAQ	""..autotmp_2+56(SP), AX
	0x0085 00133 (main.go:20)	MOVQ	AX, ""..autotmp_4+48(SP)
	0x008a 00138 (main.go:20)	LEAQ	type."".A(SB), AX
	0x0091 00145 (main.go:20)	MOVQ	AX, (SP)
	0x0095 00149 (main.go:20)	LEAQ	""..autotmp_1+120(SP), AX
	0x009a 00154 (main.go:20)	MOVQ	AX, 8(SP)
	0x009f 00159 (main.go:20)	PCDATA	$1, $1
	0x009f 00159 (main.go:20)	NOP
	0x00a0 00160 (main.go:20)	CALL	runtime.convT2E(SB)
	0x00a5 00165 (main.go:20)	MOVQ	""..autotmp_4+48(SP), AX
	0x00aa 00170 (main.go:20)	TESTB	AL, (AX)
	0x00ac 00172 (main.go:20)	MOVQ	24(SP), CX
	0x00b1 00177 (main.go:20)	MOVQ	16(SP), DX
	0x00b6 00182 (main.go:20)	MOVQ	DX, (AX)
	0x00b9 00185 (main.go:20)	LEAQ	8(AX), DI
	0x00bd 00189 (main.go:20)	PCDATA	$0, $-2
	0x00bd 00189 (main.go:20)	CMPL	runtime.writeBarrier(SB), $0
	0x00c4 00196 (main.go:20)	JEQ	200
	0x00c6 00198 (main.go:20)	JMP	281
	0x00c8 00200 (main.go:20)	MOVQ	CX, 8(AX)
	0x00cc 00204 (main.go:20)	JMP	206
	0x00ce 00206 (main.go:20)	PCDATA	$0, $-1
	0x00ce 00206 (main.go:20)	MOVQ	""..autotmp_4+48(SP), AX
	0x00d3 00211 (main.go:20)	TESTB	AL, (AX)
	0x00d5 00213 (main.go:20)	JMP	215
	0x00d7 00215 (main.go:20)	MOVQ	AX, ""..autotmp_3+96(SP)
	0x00dc 00220 (main.go:20)	MOVQ	$1, ""..autotmp_3+104(SP)
	0x00e5 00229 (main.go:20)	MOVQ	$1, ""..autotmp_3+112(SP)
	0x00ee 00238 (main.go:20)	MOVQ	AX, (SP)
	0x00f2 00242 (main.go:20)	MOVQ	$1, 8(SP)
	0x00fb 00251 (main.go:20)	MOVQ	$1, 16(SP)
	0x0104 00260 (main.go:20)	PCDATA	$1, $0
	0x0104 00260 (main.go:20)	CALL	fmt.Println(SB)
	0x0109 00265 (main.go:21)	MOVQ	144(SP), BP
	0x0111 00273 (main.go:21)	ADDQ	$152, SP
	0x0118 00280 (main.go:21)	RET
	0x0119 00281 (main.go:20)	PCDATA	$0, $-2
	0x0119 00281 (main.go:20)	CALL	runtime.gcWriteBarrierCX(SB)
	0x011e 00286 (main.go:20)	NOP
	0x0120 00288 (main.go:20)	JMP	206
	0x0122 00290 (main.go:20)	NOP
	0x0122 00290 (main.go:18)	PCDATA	$1, $-1
	0x0122 00290 (main.go:18)	PCDATA	$0, $-2
	0x0122 00290 (main.go:18)	CALL	runtime.morestack_noctxt(SB)
	0x0127 00295 (main.go:18)	PCDATA	$0, $-1
	0x0127 00295 (main.go:18)	JMP	0
	0x0000 65 48 8b 0c 25 00 00 00 00 48 8d 44 24 e8 48 3b  eH..%....H.D$.H;
	0x0010 41 10 0f 86 0a 01 00 00 48 81 ec 98 00 00 00 48  A.......H......H
	0x0020 89 ac 24 90 00 00 00 48 8d ac 24 90 00 00 00 c7  ..$....H..$.....
	0x0030 44 24 48 00 00 00 00 0f 57 c0 0f 11 44 24 50 c7  D$H.....W...D$P.
	0x0040 44 24 48 02 00 00 00 48 8d 05 00 00 00 00 48 89  D$H....H......H.
	0x0050 44 24 50 48 c7 44 24 58 04 00 00 00 8b 4c 24 48  D$PH.D$X.....L$H
	0x0060 89 4c 24 78 48 89 84 24 80 00 00 00 48 c7 84 24  .L$xH..$....H..$
	0x0070 88 00 00 00 04 00 00 00 0f 57 c0 0f 11 44 24 38  .........W...D$8
	0x0080 48 8d 44 24 38 48 89 44 24 30 48 8d 05 00 00 00  H.D$8H.D$0H.....
	0x0090 00 48 89 04 24 48 8d 44 24 78 48 89 44 24 08 90  .H..$H.D$xH.D$..
	0x00a0 e8 00 00 00 00 48 8b 44 24 30 84 00 48 8b 4c 24  .....H.D$0..H.L$
	0x00b0 18 48 8b 54 24 10 48 89 10 48 8d 78 08 83 3d 00  .H.T$.H..H.x..=.
	0x00c0 00 00 00 00 74 02 eb 51 48 89 48 08 eb 00 48 8b  ....t..QH.H...H.
	0x00d0 44 24 30 84 00 eb 00 48 89 44 24 60 48 c7 44 24  D$0....H.D$`H.D$
	0x00e0 68 01 00 00 00 48 c7 44 24 70 01 00 00 00 48 89  h....H.D$p....H.
	0x00f0 04 24 48 c7 44 24 08 01 00 00 00 48 c7 44 24 10  .$H.D$.....H.D$.
	0x0100 01 00 00 00 e8 00 00 00 00 48 8b ac 24 90 00 00  .........H..$...
	0x0110 00 48 81 c4 98 00 00 00 c3 e8 00 00 00 00 66 90  .H............f.
	0x0120 eb ac e8 00 00 00 00 e9 d4 fe ff ff              ............
	rel 3+0 t=25 type."".A+0
	rel 5+4 t=17 TLS+0
	rel 74+4 t=16 go.string."kaka"+0
	rel 141+4 t=16 type."".A+0
	rel 161+4 t=8 runtime.convT2E+0
	rel 191+4 t=16 runtime.writeBarrier+-1
	rel 261+4 t=8 fmt.Println+0
	rel 282+4 t=8 runtime.gcWriteBarrierCX+0
	rel 291+4 t=8 runtime.morestack_noctxt+0
type..eq."".A STEXT dupok size=214 args=0x18 locals=0x48 funcid=0x0
...
"".(*A).lele STEXT dupok size=139 args=0x8 locals=0x38 funcid=0x16
...
go.cuinfo.packagename. SDWARFCUINFO dupok size=0
	0x0000 6d 61 69 6e                                      main
""..inittask SNOPTRDATA size=32
	0x0000 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00  ................
	0x0010 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 24+8 t=1 fmt..inittask+0
runtime.nilinterequal·f SRODATA dupok size=8
	0x0000 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 runtime.nilinterequal+0
runtime.memequal64·f SRODATA dupok size=8
	0x0000 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 runtime.memequal64+0
runtime.gcbits.01 SRODATA dupok size=1
	0x0000 01                                               .
type..namedata.*interface {}- SRODATA dupok size=16
	0x0000 00 00 0d 2a 69 6e 74 65 72 66 61 63 65 20 7b 7d  ...*interface {}
type.*interface {} SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 4f 0f 96 9d 08 08 08 36 00 00 00 00 00 00 00 00  O......6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*interface {}-+0
	rel 48+8 t=1 type.interface {}+0
runtime.gcbits.02 SRODATA dupok size=1
	0x0000 02                                               .
type.interface {} SRODATA dupok size=80
	0x0000 10 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0010 e7 57 a0 18 02 08 08 14 00 00 00 00 00 00 00 00  .W..............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 24+8 t=1 runtime.nilinterequal·f+0
	rel 32+8 t=1 runtime.gcbits.02+0
	rel 40+4 t=5 type..namedata.*interface {}-+0
	rel 44+4 t=6 type.*interface {}+0
	rel 56+8 t=1 type.interface {}+80
type..namedata.*[]interface {}- SRODATA dupok size=18
	0x0000 00 00 0f 2a 5b 5d 69 6e 74 65 72 66 61 63 65 20  ...*[]interface 
	0x0010 7b 7d                                            {}
type.*[]interface {} SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 f3 04 9a e7 08 08 08 36 00 00 00 00 00 00 00 00  .......6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*[]interface {}-+0
	rel 48+8 t=1 type.[]interface {}+0
type.[]interface {} SRODATA dupok size=56
	0x0000 18 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 70 93 ea 2f 02 08 08 17 00 00 00 00 00 00 00 00  p../............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*[]interface {}-+0
	rel 44+4 t=6 type.*[]interface {}+0
	rel 48+8 t=1 type.interface {}+0
type..namedata.*[1]interface {}- SRODATA dupok size=19
	0x0000 00 00 10 2a 5b 31 5d 69 6e 74 65 72 66 61 63 65  ...*[1]interface
	0x0010 20 7b 7d                                          {}
type.*[1]interface {} SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 bf 03 a8 35 08 08 08 36 00 00 00 00 00 00 00 00  ...5...6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*[1]interface {}-+0
	rel 48+8 t=1 type.[1]interface {}+0
type.[1]interface {} SRODATA dupok size=72
	0x0000 10 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0010 50 91 5b fa 02 08 08 11 00 00 00 00 00 00 00 00  P.[.............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 01 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.nilinterequal·f+0
	rel 32+8 t=1 runtime.gcbits.02+0
	rel 40+4 t=5 type..namedata.*[1]interface {}-+0
	rel 44+4 t=6 type.*[1]interface {}+0
	rel 48+8 t=1 type.interface {}+0
	rel 56+8 t=1 type.[]interface {}+0
type..eqfunc."".A SRODATA dupok size=8
	0x0000 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 type..eq."".A+0
type..namedata.*main.A. SRODATA dupok size=10
	0x0000 01 00 07 2a 6d 61 69 6e 2e 41                    ...*main.A
type..namedata.*func(*main.A)- SRODATA dupok size=17
	0x0000 00 00 0e 2a 66 75 6e 63 28 2a 6d 61 69 6e 2e 41  ...*func(*main.A
	0x0010 29                                               )
type.*func(*"".A) SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 a1 36 1a 92 08 08 08 36 00 00 00 00 00 00 00 00  .6.....6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func(*main.A)-+0
	rel 48+8 t=1 type.func(*"".A)+0
type.func(*"".A) SRODATA dupok size=64
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 9d 85 6c 88 02 08 08 33 00 00 00 00 00 00 00 00  ..l....3........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func(*main.A)-+0
	rel 44+4 t=6 type.*func(*"".A)+0
	rel 56+8 t=1 type.*"".A+0
type..namedata.Lala. SRODATA dupok size=7
	0x0000 01 00 04 4c 61 6c 61                             ...Lala
type..namedata.*func()- SRODATA dupok size=10
	0x0000 00 00 07 2a 66 75 6e 63 28 29                    ...*func()
type.*func() SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 9b 90 75 1b 08 08 08 36 00 00 00 00 00 00 00 00  ..u....6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func()-+0
	rel 48+8 t=1 type.func()+0
type.func() SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 f6 bc 82 f6 02 08 08 33 00 00 00 00 00 00 00 00  .......3........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00                                      ....
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func()-+0
	rel 44+4 t=6 type.*func()+0
type..namedata.lele- SRODATA dupok size=7
	0x0000 00 00 04 6c 65 6c 65                             ...lele
type.*"".A SRODATA size=104
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 f9 23 41 1c 09 08 08 36 00 00 00 00 00 00 00 00  .#A....6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 02 00 01 00  ................
	0x0040 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0060 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*main.A.+0
	rel 48+8 t=1 type."".A+0
	rel 56+4 t=5 type..importpath."".+0
	rel 72+4 t=5 type..namedata.Lala.+0
	rel 76+4 t=27 type.func()+0
	rel 80+4 t=27 "".(*A).Lala+0
	rel 84+4 t=27 "".(*A).Lala+0
	rel 88+4 t=5 type..namedata.lele-+0
	rel 92+4 t=27 type.func()+0
	rel 96+4 t=27 "".(*A).lele+0
	rel 100+4 t=27 "".(*A).lele+0
type..namedata.*func(main.A)- SRODATA dupok size=16
	0x0000 00 00 0d 2a 66 75 6e 63 28 6d 61 69 6e 2e 41 29  ...*func(main.A)
type.*func("".A) SRODATA dupok size=56
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 49 dd ac 15 08 08 08 36 00 00 00 00 00 00 00 00  I......6........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00                          ........
	rel 24+8 t=1 runtime.memequal64·f+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func(main.A)-+0
	rel 48+8 t=1 type.func("".A)+0
type.func("".A) SRODATA dupok size=64
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 02 08 e8 ac 02 08 08 33 00 00 00 00 00 00 00 00  .......3........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*func(main.A)-+0
	rel 44+4 t=6 type.*func("".A)+0
	rel 56+8 t=1 type."".A+0
type..namedata.B.haha SRODATA dupok size=10
	0x0000 03 00 01 42 00 04 68 61 68 61                    ...B..haha
type..namedata.c-hehe SRODATA dupok size=10
	0x0000 02 00 01 63 00 04 68 65 68 65                    ...c..hehe
type."".A SRODATA size=160
	0x0000 18 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0010 bd 5a 2e 34 07 08 08 19 00 00 00 00 00 00 00 00  .Z.4............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 01 00 00 00 40 00 00 00 00 00 00 00  ........@.......
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0090 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 24+8 t=1 type..eqfunc."".A+0
	rel 32+8 t=1 runtime.gcbits.02+0
	rel 40+4 t=5 type..namedata.*main.A.+0
	rel 44+4 t=5 type.*"".A+0
	rel 48+8 t=1 type..importpath."".+0
	rel 56+8 t=1 type."".A+96
	rel 80+4 t=5 type..importpath."".+0
	rel 96+8 t=1 type..namedata.B.haha+0
	rel 104+8 t=1 type.int32+0
	rel 120+8 t=1 type..namedata.c-hehe+0
	rel 128+8 t=1 type.string+0
	rel 144+4 t=5 type..namedata.lele-+0
	rel 148+4 t=27 type.func()+0
	rel 152+4 t=27 "".(*A).lele+0
	rel 156+4 t=27 "".A.lele+0
go.string."kaka" SRODATA dupok size=4
	0x0000 6b 61 6b 61                                      kaka
type..importpath.fmt. SRODATA dupok size=6
	0x0000 00 00 03 66 6d 74                                ...fmt
gclocals·1a65e721a2ccc325b382662e7ffee780 SRODATA dupok size=10
	0x0000 02 00 00 00 01 00 00 00 01 00                    ..........
gclocals·b54c265fcfcc2aae36a147f86c7f459f SRODATA dupok size=10
	0x0000 02 00 00 00 07 00 00 00 00 00                    ..........
"".(*A).Lala.stkobj SRODATA static size=24
	0x0000 01 00 00 00 00 00 00 00 d8 ff ff ff ff ff ff ff  ................
	0x0010 00 00 00 00 00 00 00 00                          ........
	rel 16+8 t=1 type.[1]interface {}+0
gclocals·ba30782f8935b28ed1adaec603e72627 SRODATA dupok size=11
	0x0000 03 00 00 00 02 00 00 00 02 00 00                 ...........
gclocals·4ce0688dd9bdf9adf636ab5bd6db4332 SRODATA dupok size=14
	0x0000 03 00 00 00 09 00 00 00 00 00 01 00 00 00        ..............
"".A.lele.stkobj SRODATA static size=40
	0x0000 02 00 00 00 00 00 00 00 c0 ff ff ff ff ff ff ff  ................
	0x0010 00 00 00 00 00 00 00 00 e8 ff ff ff ff ff ff ff  ................
	0x0020 00 00 00 00 00 00 00 00                          ........
	rel 16+8 t=1 type.[1]interface {}+0
	rel 32+8 t=1 type."".A+0
gclocals·69c1753bd5f81501d95132d08af04464 SRODATA dupok size=8
	0x0000 02 00 00 00 00 00 00 00                          ........
gclocals·5a97ff06ec2b95a706eb55096385c78b SRODATA dupok size=12
	0x0000 02 00 00 00 0c 00 00 00 00 00 01 00              ............
"".main.stkobj SRODATA static size=40
	0x0000 02 00 00 00 00 00 00 00 a8 ff ff ff ff ff ff ff  ................
	0x0010 00 00 00 00 00 00 00 00 e8 ff ff ff ff ff ff ff  ................
	0x0020 00 00 00 00 00 00 00 00                          ........
	rel 16+8 t=1 type.[1]interface {}+0
	rel 32+8 t=1 type."".A+0
gclocals·dc9b0298814590ca3ffc3a889546fc8b SRODATA dupok size=10
	0x0000 02 00 00 00 02 00 00 00 03 00                    ..........
gclocals·2589ca35330fc0fce83503f4569854a0 SRODATA dupok size=10
	0x0000 02 00 00 00 02 00 00 00 00 00                    ..........
gclocals·15b76348caca8a511afecadf603e9401 SRODATA dupok size=10
	0x0000 02 00 00 00 03 00 00 00 00 00
```

### 类型信息

```nasm
type."".A SRODATA size=160
	0x0000 18 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0010 bd 5a 2e 34 07 08 08 19 00 00 00 00 00 00 00 00  .Z.4............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 01 00 00 00 40 00 00 00 00 00 00 00  ........@.......
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0090 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 24+8 t=1 type..eqfunc."".A+0
	rel 32+8 t=1 runtime.gcbits.02+0
	rel 40+4 t=5 type..namedata.*main.A.+0
	rel 44+4 t=5 type.*"".A+0
	rel 48+8 t=1 type..importpath."".+0
	rel 56+8 t=1 type."".A+96
	rel 80+4 t=5 type..importpath."".+0
	rel 96+8 t=1 type..namedata.B.haha+0
	rel 104+8 t=1 type.int32+0
	rel 120+8 t=1 type..namedata.c-hehe+0
	rel 128+8 t=1 type.string+0
	rel 144+4 t=5 type..namedata.lele-+0
	rel 148+4 t=27 type.func()+0
	rel 152+4 t=27 "".(*A).lele+0
	rel 156+4 t=27 "".A.lele+0
```

`type."".A SRODATA size=160` 对应的结构如下：

- 0+48   structtype.typ
  - 0+8 structtype.typ.size 表示值结构的大小，值是 24，与 值结构对齐后预计的大小一致
  - 8+8 structtype.typ.ptrdata 表示值结构的前若干个字节包含所有的指针，值是16，正好是值结构对齐后 int32 和 string 的指向数组的指针 的大小之和
  - 20+1 structtype.typ.tflag 最低位是1，表示有 uncommontype，与实际情况一致，具体见 src/runtime/type.go 的 tflag
  - 32+8 structtype.typ.gcdata 表示值结构的那些word存放着指针，这里值是 runtime.gcbits.02，02 的 二进制是 10，表示值结构只有第二个word存放的指针，与值结构实际情况一致
- 48+8   structtype.pkgPath
- 56+24 structtype.fields
  - 56+8 structtype.fields.array 指向 type."".A+96
  - 64+8 structtype.fields.len 值为 2，表示 structtype.fields 有两个元素
  - 72+8 structtype.fields.cap 值为 2，表示 structtype.fields 容量为2
- 80+16 uncommontype
  - 84+2 uncommontype.mcount 值为 1，表示 本类型有 1 个方法
  - 86+2 uncommontype.xcount 值为 0，表示 被类型没有可以导出的方法
  - 88+4 uncommontype.moff 值为 0x40，即64，表示本类型对应的方法数组的相对uncommontype的偏移量，真实偏移量为：80+64 = 144
- 96+24 structtype.fields[0]
  - 96+8 structtype.fields[0].name 指向 type..namedata.B.haha，注意类型里面的属性的tag直接写在符号名里，属性名B后面是 `.` 表示该属性是可导出的
  - 104+8 structtype.fields[0].typ 指向 type.int32
  - 112+8 structtype.fields[0].offsetAnon 值为 0
    - 最低位，表示当前属性是否是内嵌属性，当前值为0，表示非内嵌
    - 高63位，表示当前属性的值相对于 structtype.fields[0] 的偏移量，当前值为0
- 120+24 structtype.fields[1]
  - 120+8 structtype.fields[1].name 指向 type..namedata.c-hehe，属性名c后面是 `-` 表示该属性是不可导出的
  - 128+8 structtype.fields[1].typ 指向 type.string
  - 136+8 structtype.fields[1].offsetAnon 值为 0x10
    - 最低位，当前值为0，表示非内嵌
    - 高63位，当前值为 0x10 >> 1，即 8，表示当前属性的值相对于 structtype.fields[0] 的偏移量为 8，这个 8 其实就是 structtype.fields[1].typ.size
- 144+16 method
  - 144+4 method.name 指向 type..namedata.lele-，表示当前类型的方法，方法名 lele 后面是 `-` 表示，它是不可导出的
  - 148+4 method.mtyp 指向 type.func()
  - 152+4 method.ifn 指向 "".(*A).lele
  - 156+4 method.tfn 指向 "".A.lele，从这里可以看出来，这个方法的 receiver 是个非指针的
  - 注意，这里的 method 只包含了 `func (a A) lele()` 而不包含 `func (a *A) Lala()`，也变相说明了 Lala 不能被类型为 `type."".A` 的对象调用，这与语法描述中的一致。

```go
type structtype struct {
	typ     _type
	pkgPath name
	fields  []structfield
}
type structfield struct {
	name       name
	typ        *_type
	offsetAnon uintptr // byte offset of field<<1 | isEmbedded
}
type uncommontype struct {
	pkgpath nameOff
	mcount  uint16 // number of methods
	xcount  uint16 // number of exported methods
	moff    uint32 // offset from this uncommontype to [mcount]method
	_       uint32 // unused
}
type method struct { // Method on non-interface type
	name nameOff // name of method
	mtyp typeOff // method type (without receiver)
	ifn  textOff // fn used in interface call (one-word receiver)
	tfn  textOff // fn used for normal method call
}
type nameOff int32
type typeOff int32
type textOff int32
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

### 值结构

```nasm
"".main STEXT size=300 args=0x0 locals=0x98 funcid=0x0
... 
	0x002f 00047 (main.go:19)	MOVL	$0, "".a+72(SP) # main.go:19 内容是 a := A{2, "kaka"}
	0x0037 00055 (main.go:19)	XORPS	X0, X0
	0x003a 00058 (main.go:19)	MOVUPS	X0, "".a+80(SP)
	0x003f 00063 (main.go:19)	MOVL	$2, "".a+72(SP) # "".a+72(SP) ~ "".a+75(SP) 存放 int32(2)， "".a+76(SP) ~ "".a+79(SP) 都是 0，按 8 字节宽度对齐了
	0x0047 00071 (main.go:19)	LEAQ	go.string."kaka"(SB), AX
	0x004e 00078 (main.go:19)	MOVQ	AX, "".a+80(SP) # "".a+80(SP) ~ "".a+87(SP) 存放字符串底层数组的地址
	0x0053 00083 (main.go:19)	MOVQ	$4, "".a+88(SP) # "".a+88(SP) ~ "".a+95(SP) 存放字符串长度
...
```

可以看出来值结构是连续的内存，并且按word对齐了。
