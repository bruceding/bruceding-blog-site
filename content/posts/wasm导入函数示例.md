+++
date = '2025-03-08T11:42:13+08:00'
draft = true
title = 'Wasm导入函数示例'
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
description = 'Desc Text.'
disableHLJS = true 
disableShare = false
hideSummary = false
tags = ["Wasm"]
+++
从[wasm技术介绍](../wasm技术介绍) 里可以看到如果导出函数，并且在宿主机里进行调用。本文介绍如何进行导入函数，从宿主机导入函数并在wasm函数里使用。
还是基于tinygo 来进行演示。

首先定义导入函数
```go
/go:build tinygo

package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

//export  CallFromHost
func CallFromHost() int32

//export Add
func Add(a, b int32) int32 {
	return a + b
}

type User struct {
	Name string

	Age int
}

//export CallHost
func CallHost() int32 {
	return CallFromHost()
}

//export AddUser
func AddUser(nameData *byte, nameLen int32, age int32) bool {
	fmt.Println(nameLen, age)
	name := RawBytePtrToString(nameData, int(nameLen))

	user := User{Name: name, Age: int(age)}
	fmt.Printf("Add User:%v\n", user)
	return true
}

func main() {
	// 这里必须引用下
	CallFromHost()
	fmt.Println("Hello, WebAssembly!")
}

func RawBytePtrToString(raw *byte, size int) string {
	//nolint
	return *(*string)(unsafe.Pointer(&reflect.SliceHeader{
		Data: uintptr(unsafe.Pointer(raw)),
		Len:  size,
		Cap:  size,
	}))
}

func RawBytePtrToByteSlice(raw *byte, size int) []byte {
	//nolint
	return *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: uintptr(unsafe.Pointer(raw)),
		Len:  size,
		Cap:  size,
	}))
}

```
定义 CallFromHost 函数是从宿主机导入的。也是通过 `//export` 标识的。只不过这里只定义了函数体，没有定义实现。需要注意的一点，需要在 main 函数下引用才行。否则在生成的wasm文件中找不到。

然后又定义了一个导出函数 CallHost ， 来调用 CallFromHost 函数。

首先进行编译
```
tinygo build -target wasip1 -o main.wasm main.go

```
可以通过wasm2wat 来查看是否包含了导入函数。
```
wasm2wat main.wasm | grep import
  (import "wasi_snapshot_preview1" "fd_write" (func $runtime.fd_write (type 6)))
  (import "wasi_snapshot_preview1" "proc_exit" (func $runtime.proc_exit (type 2)))
  (import "wasi_snapshot_preview1" "clock_time_get" (func $runtime.clock_time_get (type 19)))
  (import "wasi_snapshot_preview1" "args_sizes_get" (func $runtime.args_sizes_get (type 7)))
  (import "wasi_snapshot_preview1" "args_get" (func $runtime.args_get (type 7)))
  (import "env" "CallFromHost" (func $CallFromHost (type 9)))
  (import "wasi_snapshot_preview1" "random_get" (func $__imported_wasi_snapshot_preview1_random_get (type 7)))
        call $__imported_wasi_snapshot_preview1_random_get
```
可以看到 通过env 模块导入了 CallFromHost 函数。

那么，在宿主机上如何进行函数导入呢？还是使用 wasmtime-go sdk 说明
```go
package main

import (
	"fmt"
	"log"

	"github.com/bytecodealliance/wasmtime-go/v29"
)

func main() {
	engine := wasmtime.NewEngine()
	store := wasmtime.NewStore(engine)
	// 配置 WASI 上下文
	wasiConfig := wasmtime.NewWasiConfig()
	wasiConfig.InheritStdout() // 继承标准输出
	store.SetWasi(wasiConfig)
	module, err := wasmtime.NewModuleFromFile(engine, "main.wasm")
	check(err)

	// 创建 Linker 并配置默认的 WASI 导入
	linker := wasmtime.NewLinker(engine)
	err = linker.DefineWasi()
	check(err)

	linker.DefineFunc(store, "env", "CallFromHost", func() int32 {
		return 12
	})

	// 实例化模块
	instance, err := linker.Instantiate(store, module)
	check(err)

	callHostF := instance.GetExport(store, "CallHost").Func()
	ret, err := callHostF.Call(store)
	check(err)
	fmt.Println("CallHost ret", ret, ret.(int32) == 12)
	// 调用导出的函数
	// 获取导出的 Add 函数并调用
	addF := instance.GetExport(store, "Add").Func()
	val, err := addF.Call(store, 6, 27)
	check(err)
	fmt.Printf("add(6, 27) = %d\n", val.(int32))

	memory := instance.GetExport(store, "memory").Memory()
	fmt.Println(memory.DataSize(store), memory.Data(store))
	// 获取导出的 malloc 函数
	malloc := instance.GetExport(store, "malloc").Func()
	if malloc == nil {
		log.Fatalf("malloc function not found")
	}

	addUser := instance.GetExport(store, "AddUser").Func()
	if addUser == nil {
		log.Fatalf("AddUser function not found")
	}
	name := "Alice"
	nameData := []byte(name)
	nameLen := int32(len(name))
	// 调用 malloc 分配内存
	mallocResult, err := malloc.Call(store, nameLen+1)
	if err != nil {
		log.Fatalf("Failed to call malloc: %s", err)
	}
	namePtr := mallocResult.(int32)

	fmt.Println("namePtr", namePtr)
	fmt.Println(memory.DataSize(store), memory.Data(store))
	mem := memory.UnsafeData(store)
	fmt.Println("mem", len(mem))
	copy(mem[namePtr:], nameData)
	age := int32(30)

	// 调用 AddUser 函数
	result, err := addUser.Call(store, namePtr, nameLen, age)
	if err != nil {
		log.Fatalf("Failed to call AddUser: %s", err)
	}
	fmt.Println(result)

	// 获取导出的 free 函数并释放内存
	free := instance.GetExport(store, "free").Func()
	if free != nil {
		_, err = free.Call(store, namePtr)
		if err != nil {
			log.Fatalf("Failed to call free: %s", err)
		}
	}
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}

```
我们通过 linker.DefineFunc 定义了导入函数，此函数没有输入参数，只有int32的返回值。
通过对  CallHost 拿到了最终的返回值。
```go
	callHostF := instance.GetExport(store, "CallHost").Func()
	ret, err := callHostF.Call(store)
	check(err)
	fmt.Println("CallHost ret", ret, ret.(int32) == 12)
```
