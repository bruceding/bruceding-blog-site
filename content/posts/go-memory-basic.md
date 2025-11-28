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

