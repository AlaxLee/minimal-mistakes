---
title: "golang 基本类型之 Map types"
toc: true
toc_icon: align-left
classes: wide
categories:
  - golang
  - 基本类型
tags:
  - "Go: 1.16.3"
---



*map type* 字典类型，不是线程安全的，sync.Map 才是线程安全的

## 官方定义

官方文档地址 [https://golang.google.cn/ref/spec#Map_types](https://golang.google.cn/ref/spec#Map_types)

A map is an unordered group of elements of one type, called the element type, indexed by a set of unique *keys* of another type, called the key type. The value of an uninitialized map is `nil`.

## 类型信息

具体见：src/runtime/type.go

```go
type maptype struct {
	typ    _type
	key    *_type
	elem   *_type
	bucket *_type // internal type representing a hash bucket
	// function for hashing keys (ptr to key, seed) -> hash
	hasher     func(unsafe.Pointer, uintptr) uintptr
	keysize    uint8  // size of key slot
	elemsize   uint8  // size of elem slot
	bucketsize uint16 // size of bucket
	flags      uint32
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

一个指针，指向 hmap。具体见：src/runtime/map.go

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
/*
count     len()的返回值，即key的数量（或者说value的数量）
B         hmap 有 2^B 个 bucket 链表 
noverflow 所有 overflow buckets 的数量，与 len(*extra.overflow) 数量一致
buckets   已验证过，确实是个指向数组的指针，相当于 *[2^B]bmap，数组的每个元素bmap是一个单链表的首元素

nevacuate 
*/

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
/*** 相当于
type bmap struct {
    tophash   [bucketCnt]uint8
    keys      [bucketCnt]keytype
    elems     [bucketCnt]elemtype
    overflow  uintptr
}
*/

const (
	// Maximum number of key/elem pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits
    ...
)
```

下图是值结构的示意图

![image-20210729152845675](/assets/images/image/map.png)

下图是 bmap 的示意图， 一个bucket就是一个bmap结构，`HOBHash` 指的就是 top hash。注意到 key 和 value 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

top hash 有三个作用：

- 值为 0 时，标志对应位置的key、value字段里面没有有效值（就算有数据，也被认为是无效的）
- 值小于 minTopHash 时，标志对应位置的key、value处于 evacuated 状态（即，处于扩容状态中）
- 值大于等于 minTopHash 时，存放的是 key 的 hash 的高8位，主要用于快速对比 key 是否一致，同时也表明对应位置的key、value是有效的

```go
//具体见：src/runtime/map.go
	minTopHash     = 5 // minimum tophash for a normal filled cell.
```

![图片](/assets/images/image/640-20210805102816336.jpeg)

## 具体案例

```go
package main

import "fmt"

func main() {
	a := map[int]int{1: 4, 2: 5, 3: 6}
	fmt.Println(a)
}

```

```nasm
AlaxdeMacBook-Pro:test alax$ go tool compile -S -N -l main.go 
"".main STEXT size=426 args=0x0 locals=0xa0 funcid=0x0
...
	0x002f 00047 (main.go:6)	PCDATA	$1, $0
	0x002f 00047 (main.go:6)	CALL	runtime.makemap_small(SB) # func makemap_small() *hmap 创建一个map并返回它
	0x0034 00052 (main.go:6)	MOVQ	(SP), AX
	0x0038 00056 (main.go:6)	MOVQ	AX, "".a+64(SP)           # "".a+64(SP) 存放了一个 *hmap 即 a
	0x003d 00061 (main.go:6)	MOVQ	$1, ""..autotmp_3+56(SP)  # 第一个 key 1
	0x0046 00070 (main.go:6)	MOVQ	$4, ""..autotmp_4+48(SP)  # 第一个 value 4
	0x004f 00079 (main.go:6)	MOVQ	""..autotmp_3+56(SP), AX
	0x0054 00084 (main.go:6)	MOVQ	"".a+64(SP), CX
	0x0059 00089 (main.go:6)	LEAQ	type.map[int]int(SB), DX
	0x0060 00096 (main.go:6)	MOVQ	DX, (SP)                  # type.map[int]int(SB)
	0x0064 00100 (main.go:6)	MOVQ	CX, 8(SP)                 # *hmap 即 a
	0x0069 00105 (main.go:6)	MOVQ	AX, 16(SP)                # 1
	0x006e 00110 (main.go:6)	PCDATA	$1, $1
	0x006e 00110 (main.go:6)	CALL	runtime.mapassign_fast64(SB) # func mapassign_fast64(t *maptype, h *hmap, key uint64) unsafe.Pointer  这个返回的指针是个啥？从下面五行汇编来看，似乎返回的是 key 对应的 value 的地址
	0x0073 00115 (main.go:6)	MOVQ	24(SP), AX
	0x0078 00120 (main.go:6)	MOVQ	AX, ""..autotmp_5+96(SP)
	0x007d 00125 (main.go:6)	TESTB	AL, (AX)
	0x007f 00127 (main.go:6)	MOVQ	""..autotmp_4+48(SP), CX  # 4
	0x0084 00132 (main.go:6)	MOVQ	CX, (AX)                  # 把 4 写入 key 1 的对应的地址里
	0x0087 00135 (main.go:6)	MOVQ	$2, ""..autotmp_3+56(SP)  # 第二个 key ，以下逻辑与处理第一个 key value 相同
	0x0090 00144 (main.go:6)	MOVQ	$5, ""..autotmp_4+48(SP)
	0x0099 00153 (main.go:6)	MOVQ	""..autotmp_3+56(SP), AX
	0x009e 00158 (main.go:6)	MOVQ	"".a+64(SP), CX
	0x00a3 00163 (main.go:6)	LEAQ	type.map[int]int(SB), DX
	0x00aa 00170 (main.go:6)	MOVQ	DX, (SP)
	0x00ae 00174 (main.go:6)	MOVQ	CX, 8(SP)
	0x00b3 00179 (main.go:6)	MOVQ	AX, 16(SP)
	0x00b8 00184 (main.go:6)	CALL	runtime.mapassign_fast64(SB)
	0x00bd 00189 (main.go:6)	MOVQ	24(SP), AX
	0x00c2 00194 (main.go:6)	MOVQ	AX, ""..autotmp_6+88(SP)
	0x00c7 00199 (main.go:6)	TESTB	AL, (AX)
	0x00c9 00201 (main.go:6)	MOVQ	""..autotmp_4+48(SP), CX
	0x00ce 00206 (main.go:6)	MOVQ	CX, (AX)
	0x00d1 00209 (main.go:6)	MOVQ	$3, ""..autotmp_3+56(SP)  # 第三个 key ，以下逻辑与处理第一个 key value 相同
	0x00da 00218 (main.go:6)	MOVQ	$6, ""..autotmp_4+48(SP)
	0x00e3 00227 (main.go:6)	MOVQ	""..autotmp_3+56(SP), AX
	0x00e8 00232 (main.go:6)	MOVQ	"".a+64(SP), CX
	0x00ed 00237 (main.go:6)	LEAQ	type.map[int]int(SB), DX
	0x00f4 00244 (main.go:6)	MOVQ	DX, (SP)
	0x00f8 00248 (main.go:6)	MOVQ	CX, 8(SP)
	0x00fd 00253 (main.go:6)	MOVQ	AX, 16(SP)
	0x0102 00258 (main.go:6)	CALL	runtime.mapassign_fast64(SB)
	0x0107 00263 (main.go:6)	MOVQ	24(SP), AX
	0x010c 00268 (main.go:6)	MOVQ	AX, ""..autotmp_7+80(SP)
	0x0111 00273 (main.go:6)	TESTB	AL, (AX)
	0x0113 00275 (main.go:6)	MOVQ	""..autotmp_4+48(SP), CX
	0x0118 00280 (main.go:6)	MOVQ	CX, (AX)
...
type.noalg.map.bucket[int]int SRODATA dupok size=176
	0x0000 90 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 b3 28 ed bb 02 08 08 19 00 00 00 00 00 00 00 00  .(..............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 04 00 00 00 00 00 00 00 04 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0090 90 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x00a0 00 00 00 00 00 00 00 00 10 01 00 00 00 00 00 00  ................
	rel 32+8 t=1 runtime.gcbits.+0
	rel 40+4 t=5 type..namedata.*map.bucket[int]int-+0
	rel 44+4 t=6 type.*map.bucket[int]int+0
	rel 48+8 t=1 type..importpath..+0
	rel 56+8 t=1 type.noalg.map.bucket[int]int+80
	rel 80+8 t=1 type..namedata.topbits-+0
	rel 88+8 t=1 type.[8]uint8+0
	rel 104+8 t=1 type..namedata.keys-+0
	rel 112+8 t=1 type.noalg.[8]int+0
	rel 128+8 t=1 type..namedata.elems-+0
	rel 136+8 t=1 type.noalg.[8]int+0
	rel 152+8 t=1 type..namedata.overflow-+0
	rel 160+8 t=1 type.uintptr+0
...
type.map[int]int SRODATA dupok size=88
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 50 1b 58 23 02 08 08 35 00 00 00 00 00 00 00 00  P.X#...5........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0050 08 08 90 00 04 00 00 00                          ........
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*map[int]int-+0
	rel 44+4 t=6 type.*map[int]int+0
	rel 48+8 t=1 type.int+0
	rel 56+8 t=1 type.int+0
	rel 64+8 t=1 type.noalg.map.bucket[int]int+0
	rel 72+8 t=1 runtime.memhash64·f+0
...
```

### `type.map[int]int SRODATA dupok size=88` 具体分析如下：

- 对应 src/runtime/type.go 的 maptype 
- size 88 与 maptype 的大小一致
- 0+8 对应的是 maptype.typ.size，表示值结构的size，具体值是8，说明在内存里，map 的值只有8字节
- 8+8 对应的是 maptype.typ.ptrdata，表示值结构里包含指针的size，具体值也是8，可以推测出 map 的值应该是一个指针
- 32+8 的 runtime.gcbits.01，说明值结构的第一个 word 是一个指针，与上一条推测一致
- 48+8 对应的是 maptype.key，表示 key 类型，是 type.int
- 56+8 对应的是 maptype.elem，表示 value 类型，是 type.int
- 64+8 对应的是 maptype.bucket，表示 bucket 类型，这里是 type.noalg.map.bucket[int]int
- 72+8 对应的是 maptype.hasher，应该是用来给 key 做hash的函数

### `type.noalg.map.bucket[int]int SRODATA dupok size=176` 具体分析如下：

此类型对应的值结构由 src/cmd/compile/internal/gc/reflect.go 的 `func bmap(t *types.Type) *types.Type` 函数创建，与 src/runtime/map.go 里 bmap 结构体的描述基本一致，只有第一个属性名称略有不同。

```go
// bmap makes the map bucket type given the type of the map.
func bmap(t *types.Type) *types.Type {
...
	bucket := types.New(TSTRUCT)
...
	field := make([]*types.Field, 0, 5)

	// The first field is: uint8 topbits[BUCKETSIZE].
	arr := types.NewArray(types.Types[TUINT8], BUCKETSIZE)
	field = append(field, makefield("topbits", arr))

	arr = types.NewArray(keytype, BUCKETSIZE)
	arr.SetNoalg(true)
	keys := makefield("keys", arr)
	field = append(field, keys)

	arr = types.NewArray(elemtype, BUCKETSIZE)
	arr.SetNoalg(true)
	elems := makefield("elems", arr)
	field = append(field, elems)

...
	otyp := types.NewPtr(bucket)
	if !elemtype.HasPointers() && !keytype.HasPointers() {
		otyp = types.Types[TUINTPTR]
	}
	overflow := makefield("overflow", otyp)
	field = append(field, overflow)
...
	bucket.SetFields(field[:])
...
	t.MapType().Bucket = bucket

	bucket.StructType().Map = t
	return bucket
}
```

可以认为，map.bucket 值结构相当于：

```go
type BMap struct {
	topbits  [8]uint8
	keys     [8]keytype
	elems    [8]elemtype
    overflow *BMap  // 如果 key 和 elem 里面不存在指针的话，overflow 的类型会变成 uintptr
}
```

对于本例来说，值结构相当于如下，其大小为 8 + 8x8 + 8x8 + 8 = 144 字节

```go
type BMap struct {
	topbits  [8]uint8
	keys     [8]int
	elems    [8]int
	overflow uintptr
}
```

接下来具体分析其二进制：

```nasm
type.noalg.map.bucket[int]int SRODATA dupok size=176
	0x0000 90 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 b3 28 ed bb 02 08 08 19 00 00 00 00 00 00 00 00  .(..............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 04 00 00 00 00 00 00 00 04 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0090 90 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x00a0 00 00 00 00 00 00 00 00 10 01 00 00 00 00 00 00  ................
	rel 32+8 t=1 runtime.gcbits.+0
	rel 40+4 t=5 type..namedata.*map.bucket[int]int-+0
	rel 44+4 t=6 type.*map.bucket[int]int+0
	rel 48+8 t=1 type..importpath..+0
	rel 56+8 t=1 type.noalg.map.bucket[int]int+80
	rel 80+8 t=1 type..namedata.topbits-+0
	rel 88+8 t=1 type.[8]uint8+0
	rel 104+8 t=1 type..namedata.keys-+0
	rel 112+8 t=1 type.noalg.[8]int+0
	rel 128+8 t=1 type..namedata.elems-+0
	rel 136+8 t=1 type.noalg.[8]int+0
	rel 152+8 t=1 type..namedata.overflow-+0
	rel 160+8 t=1 type.uintptr+0
```

- noalg 是 no algorithms 的 缩写，表示当前类型不含 hash 和 eq，具体解释可以参考 src/cmd/compile/internal/gc/reflect.go 的 `typeHasNoAlg` 函数的注释
- 对应 src/runtime/type.go 的 structtype
- size=176，structtype（48+8+24=80）和 4 个 field（24*4=96）大小加起来正好 176

- 0+48   structtype.typ
  - 0+8 structtype.typ.size 表示值结构的大小，值是 0x90 即 144，与 值结构的大小一致
  - 8+8 structtype.typ.ptrdata 表示值结构的前若干个字节包含所有的指针，值是0，表示值结构中不含指针
  - 32+8 structtype.typ.gcdata 表示值结构的那些word存放着指针，这里值是 `runtime.gcbits.`，表示值结构中不含指针，与值结构实际情况一致
- 48+8   structtype.pkgPath
- 56+24 structtype.fields
  - 56+8 structtype.fields.array 指向 `type.noalg.map.bucket[int]int+80`
  - 64+8 structtype.fields.len 值为 4，表示 structtype.fields 有4个元素
  - 72+8 structtype.fields.cap 值为 4，表示 structtype.fields 容量为 4
- 80+24 structtype.fields[0]
  - 80+8 structtype.fields[0].name 指向 type..namedata.topbits-，属性名topbits后面是 `-` 表示该属性是不可导出的
  - 88+8 structtype.fields[0].typ 指向 type.[8]uint8
  - 96+8 structtype.fields[0].offsetAnon 值为0
    - 最低位，表示当前属性是否是内嵌属性，当前值为0，表示非内嵌
    - 高63位，表示当前属性 topbits 相对于整个值结构的偏移量，当前值为0
- 104+24 structtype.fields[1]
  - 104+8 structtype.fields[1].name 指向 type..namedata.keys-，属性名keys后面是 `-` 表示该属性是不可导出的
  - 112+8 structtype.fields[1].typ 指向 type.noalg.[8]int
  - 120+8 structtype.fields[1].offsetAnon 值为 0x10
    - 最低位，当前值为0，表示非内嵌
    - 高63位，当前值为 0x10 >> 1，即 8，表示当前属性 keys 相对于 整个值结构 的偏移量为 8，这个 8 其实就是 structtype.fields[1].typ.size，与当前的实际情况 `topbits [8]uint8` 大小一致。注意，这种偏移量计算的时候要考虑结构体的对齐。
- 128+24 structtype.fields[2]
  - 128+8 structtype.fields[2].name 指向 type..namedata.elems-，属性名elems后面是 `-` 表示该属性是不可导出的
  - 1+8 structtype.fields[2].typ 指向 type.noalg.[8]int
  - 144+8 structtype.fields[2].offsetAnon 值为 0x90
    - 最低位，当前值为0，表示非内嵌
    - 高63位，当前值为 0x90 >> 1，即 72，表示当前属性 elems 相对于 整个值结构 的偏移量为 72，这个 72 其实就是属性 topbits（8）与 keys（64）之和，与当前的实际情况大小一致
- 152+24 structtype.fields[3]
  - 152+8 structtype.fields[3].name 指向 type..namedata.overflow-，属性名overflow后面是 `-` 表示该属性是不可导出的
  - 160+8 structtype.fields[3].typ 指向 type.noalg.[8]int
  - 168+8 structtype.fields[3].offsetAnon 值为 0x110
    - 最低位，当前值为0，表示非内嵌
    - 高63位，当前值为 0x110> 1，即 136，表示当前属性 overflow 相对于 整个值结构 的偏移量为 136，这个 136 其实就是属性 topbits （8）、 keys（64） 与 elems（64） 之和，与当前的实际情况大小一致

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
```

## 
