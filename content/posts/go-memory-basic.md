+++
date = '2025-11-28T23:28:52+08:00'
draft = true
title = 'Go Memory Basic'
searchHidden = true
ShowReadingTime =  true
ShowBreadCrumbs =  true
ShowPostNavLinks =  true
ShowWordCount =  true
ShowRssButtonInSectionTermList =  true
UseHugoToc = true
showToc = true
TocOpen = false
hidemeta = false
comments = false
description = ''
disableHLJS = true 
disableShare = false
hideSummary = false
tags = ["go", "memory"]
+++
本文介绍在go中如何更高效的使用内存，并且提高性能。 

## 内存对齐
在go中，声明一个struct 时，要根据字段类型的类型和大小，调整字段的顺序，以提高内存的利用。下面来看下具体的例子说明。

```go
package main

import (
	"fmt"
	"unsafe"
)

// ❌ Bad ordering: adds padding
type Bad struct {
	a int32 // 4 bytes
	b int64 // 8 bytes
}

// ✅ Good ordering: no wasted padding
type Good struct {
	s []int // 24 bytes
	b int64 // 8 bytes
	m map[string]any // 8 bytes 
	str string // 16 bytes
	a int32 // 4 bytes
	bl bool // 1 byte
}

func main() {
	fmt.Println("--- ❌ Bad Ordering ---")
	bad := Bad{}
	// 1. 结构体总大小 (Sizeof)
	fmt.Printf("Total size of Bad struct: %d bytes\n", unsafe.Sizeof(bad))
	// 2. 字段偏移量 (Offsetof)
	fmt.Printf("  Field a (int32) offset: %d\n", unsafe.Offsetof(bad.a))
	fmt.Printf("  Field b (int64) offset: %d\n", unsafe.Offsetof(bad.b))
	// 3. 结构体对齐值 (Alignof)
	fmt.Printf("  Struct alignment requirement: %d\n", unsafe.Alignof(bad))

	fmt.Println("\n--- ✅ Good Ordering ---")
	good := Good{}
	// 1. 结构体总大小 (Sizeof)
	fmt.Printf("Total size of Good struct: %d bytes\n", unsafe.Sizeof(good))
	fmt.Printf("Total size of Good.s struct: %d bytes\n", unsafe.Sizeof(good.s))
	// 2. 字段偏移量 (Offsetof)
	fmt.Printf("  Field s ([]int) offset: %d\n", unsafe.Offsetof(good.s))
	fmt.Printf("  Field b (int64) offset: %d\n", unsafe.Offsetof(good.b))
	fmt.Printf("  Field m (map) offset: %d\n", unsafe.Offsetof(good.m))
	fmt.Printf("  Field str (string) offset: %d\n", unsafe.Offsetof(good.str))
	fmt.Printf("  Field a (int32) offset: %d\n", unsafe.Offsetof(good.a))
	fmt.Printf("  Field bl (bool) offset: %d\n", unsafe.Offsetof(good.bl))
	// 3. 结构体对齐值 (Alignof)
	fmt.Printf("  Struct alignment requirement: %d\n", unsafe.Alignof(good))
}
```
我们先来看下运行结果：

```sh
--- ❌ Bad Ordering ---
Total size of Bad struct: 16 bytes
  Field a (int32) offset: 0
  Field b (int64) offset: 8
  Struct alignment requirement: 8

--- ✅ Good Ordering ---
Total size of Good struct: 64 bytes
Total size of Good.s struct: 24 bytes
  Field s ([]int) offset: 0
  Field b (int64) offset: 24
  Field m (map) offset: 32
  Field str (string) offset: 40
  Field a (int32) offset: 56
  Field bl (bool) offset: 60
  Struct alignment requirement: 8
```
上面的Bad struct 中，结构体大小是16字节，但是字段b的偏移量是8，这就导致了字段a和b之间有4字节的padding。内存并不紧凑。 如果定义成

```go
type Bad struct {
    b int64 // 8 bytes
    a int32 // 4 bytes
}
```
这样的话，虽然结构体大小仍是是16字节，但是字段a的偏移量是4，这就没有padding了。内存紧凑了。
那么 Good struct 中，总大小是64 字节，但是字段之间并没有padding。那么这里的思路就是字段大的放前面，字段小的放后面。减少字段之间的padding，提高内存的利用率。 
从Good 的定义中，可以看到slice 的大小是24字节，包括 指针、长度、容量。 而map的大小是8字段，实际是一个指针的大小。

## 逃逸分析
在go管理的内存中，有两种类型的内存：栈内存和堆内存。 栈内存的回收效率更高，不需要GC的参与，函数返回时，栈内存会被自动回收。 而堆内存的回收效率较低，需要GC的参与。 而大量存在堆内存的话，会导致GC频率增加，GC的运行会导致CPU使用率高，并且可能会产生长时间的STW，导致应用性能不稳定。

但我们无法直接判断函数内的变量是分配到栈内存还是堆内存，只有编译器能。这就需要通过逃逸分析来判断， 一般通过 `go build -gcflags '-m' main.go` 来检查。

通过下面的代码来看下逃逸分析的结果。

```
package main

func main() {
	_ = getInt()
	_ = getIntByPtr()
	_ = getSlice()
	_ = getSliceWithLocal(4)
	_ = getSliceWithLocal(100)
	_ = getMap()
	_ = getMapWithSize(10)
	_ = getMapWithSize(len([]string{"apple", "banana", "cherry"}))
	_ = getMapWithSlice([]string{"apple", "banana", "cherry"})
}
```
具体的运行命令是 `go build -gcflags '-m -m' main.go`, 其中 `-m -m` 表示打印所有的逃逸分析结果。

```go
//go:noinline
func getInt() int {
	x := 10
	return x
}
//go:noinline
func getIntByPtr() *int {
	x := 10
	return &x
}

```
`//go:noinline` 防止编译器内联优化，导致逃逸分析结果错误。

可以看到结果如下, getInt 没有任何的信息显示。 getInt 中返回x 实际是返回了一个副本，本身 x 函数返回后就会释放。 getIntByPtr 由于返回了一个指针，并且引用了本地变量，那么 x 就会逃逸到堆内存。

```sh
./main.go:21:2: x escapes to heap:
./main.go:21:2:   flow: ~r0 = &x:
./main.go:21:2:     from &x (address-of) at ./main.go:22:9
./main.go:21:2:     from return &x (return) at ./main.go:22:2
./main.go:21:2: moved to heap: x

```
再来看下slice 的逃逸分析结果。
```go
//go:noinline
func getSlice() []string {
	localSlice := make([]string, 16)
	for i := 0; i < 16; i++ {
		localSlice[i] = "item"
	}
	ret := make([]string, 0, 16)
	return ret
}

//go:noinline
func getSliceWithLocal(size int) []string {
	localSlice := make([]string, size)
	for i := 0; i < size; i++ {
		localSlice[i] = "item"
	}
	ret := make([]string, 0, 16)
	return ret
}
```
结果如下, getSlice 中 localSlice 是一个本地变量，并且初始化时编译器就知道了它的大小，会直接分配到栈上。 `./main.go:26:20: make([]string, 16) does not escape` 这里直接说明了。而ret 是作为函数的返回值，只能分配到堆上。否则，函数返回之后，栈空间就回收了，如果不分配到堆上，内存就是无效了。

再来看下getSliceWithLocal这里的localSlice的分配情况，`./main.go:36:20: make([]string, size) escapes to heap:` 这里说明了分配到了堆上，那是由于 (non-constant size) 导致。编辑器无法决定大小，只能运行时决定。和上面相同，ret 也直接分配到了堆上，供函数返回使用。
```sh
./main.go:30:13: make([]string, 0, 16) escapes to heap:
./main.go:30:13:   flow: ret = &{storage for make([]string, 0, 16)}:
./main.go:30:13:     from make([]string, 0, 16) (spill) at ./main.go:30:13
./main.go:30:13:     from ret := make([]string, 0, 16) (assign) at ./main.go:30:6
./main.go:30:13:   flow: ~r0 = ret:
./main.go:30:13:     from return ret (return) at ./main.go:31:2
./main.go:26:20: make([]string, 16) does not escape
./main.go:30:13: make([]string, 0, 16) escapes to heap
./main.go:36:20: make([]string, size) escapes to heap:
./main.go:36:20:   flow: {heap} = &{storage for make([]string, size)}:
./main.go:36:20:     from make([]string, size) (non-constant size) at ./main.go:36:20
./main.go:40:13: make([]string, 0, 16) escapes to heap:
./main.go:40:13:   flow: ret = &{storage for make([]string, 0, 16)}:
./main.go:40:13:     from make([]string, 0, 16) (spill) at ./main.go:40:13
./main.go:40:13:     from ret := make([]string, 0, 16) (assign) at ./main.go:40:6
./main.go:40:13:   flow: ~r0 = ret:
./main.go:40:13:     from return ret (return) at ./main.go:41:2
./main.go:36:20: make([]string, size) escapes to heap
./main.go:40:13: make([]string, 0, 16) escapes to heap

```
再看下map 的情况。
```go
//go:noinline
func getMap() map[string]int {
	return map[string]int{
		"apple":  5,
	}
}

//go:noinline
func getMapWithSize(size int) map[string]int {
	m1 := make(map[string]int, size)
	m1["apple"] = 5
	for k, v := range m1 {
		_ = k
		_ = v
	}
	return map[string]int{
		"apple":  5,
	}
}
//go:noinline
func getMapWithSlice(strs []string) map[string]int {
	m1 := make(map[int]string, len(strs))
	for i, s := range strs {
		m1[i] = s
	}
	return map[string]int{
		"apple":  5,
	}
}
```
输出结果如下, getMap 中肯定要分配到堆上才行。

getMapWithSize 中m1 是分配到栈上的。和getSliceWithLocal中的localSlice 不同，但是这里分配到栈上的是m1的map 结构，本身m1 的底层数据是分配到堆上的。

getMapWithSlice 中的m1也是分配到栈上的，但是也只是map的本身的结构。map 的Key/Value数据是分配到堆上上的， `parameter strs leaks to {heap} with ` 这里说明 strs 的元素值会复制到map的value 中，元素值实际上在堆上，原先 strs 本身是在栈上，经过map的复制，元素值都到了堆上。

```sh
./main.go:46:23: map[string]int{...} escapes to heap:
./main.go:46:23:   flow: ~r0 = &{storage for map[string]int{...}}:
./main.go:46:23:     from map[string]int{...} (spill) at ./main.go:46:23
./main.go:46:23:     from return map[string]int{...} (return) at ./main.go:46:2
./main.go:46:23: map[string]int{...} escapes to heap
./main.go:59:23: map[string]int{...} escapes to heap:
./main.go:59:23:   flow: ~r0 = &{storage for map[string]int{...}}:
./main.go:59:23:     from map[string]int{...} (spill) at ./main.go:59:23
./main.go:59:23:     from return map[string]int{...} (return) at ./main.go:59:2
./main.go:53:12: make(map[string]int, size) does not escape
./main.go:59:23: map[string]int{...} escapes to heap
./main.go:64:22: parameter strs leaks to {heap} with derefs=1:
./main.go:64:22:   flow: {temp} = strs:
./main.go:64:22:   flow: s = *{temp}:
./main.go:64:22:     from for loop (range-deref) at ./main.go:66:14
./main.go:64:22:   flow: {heap} = s:
./main.go:64:22:     from m1[i] = s (assign) at ./main.go:67:9
./main.go:69:23: map[string]int{...} escapes to heap:
./main.go:69:23:   flow: ~r0 = &{storage for map[string]int{...}}:
./main.go:69:23:     from map[string]int{...} (spill) at ./main.go:69:23
./main.go:69:23:     from return map[string]int{...} (return) at ./main.go:69:2
./main.go:64:22: leaking param content: strs
./main.go:65:12: make(map[int]string, len(strs)) does not escape
./main.go:69:23: map[string]int{...} escapes to heap

```


