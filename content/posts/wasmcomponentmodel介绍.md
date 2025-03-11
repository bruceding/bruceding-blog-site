+++
date = '2025-03-11T22:27:01+08:00'
draft = false 
title = 'Wasm component model介绍'
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
tags = ["wasm", "component model"]
+++

在前面的文章中，可以看到如何生成 wasip1 的wasm 模块。wasip1 中wasm 的使用中还有很多限制：

* 函数的入参只支持整型和浮点类型，不支持字符串类型。
* 使用共享内存的方式传递数据，数据要传递的话，入参只能使用指针和数据长度。

wasip1中一个wasm 文件就可以认为是一个module. wasip2 引入了 component model的概念，它扩展了module的功能，使用更高抽象的WIT(wasm interface type) 来描述module的接口。通过WIT， 可以把各个语言的实现翻译成canonical ABI ，定义了底层的内存是如何组织的，更高级的对象如何在内存中表示等等。

目前 component model 还是发展中，对于guest 语言来说，都有成熟的demo介绍如何使用WIT并生成wasm 文件。但是对于 host 来说，目前只有rust支持运行，还没有看到其他语言的例子。

WIT 的定义可以参考 [https://component-model.bytecodealliance.org/design/wit.html](https://component-model.bytecodealliance.org/design/wit.html)。从[这里](https://component-model.bytecodealliance.org/introduction.html)也能看到具体语言的应用。

wasmtime 也支持了如何运行component model,只不过目前只能运行 "command" component。也就是说需要引入了[wasi:cli/command world](https://github.com/WebAssembly/wasi-cli/blob/main/wit/command.wit)。

## C 支持 component model
在介绍之前，首先安装 [wit-bindgen](https://github.com/bytecodealliance/wit-bindgen#cli-installation), [wasm-tools](https://github.com/bytecodealliance/wasm-tools),   [WASI SDK](https://github.com/webassembly/wasi-sdk) , [wkg](https://github.com/bytecodealliance/wasm-pkg-tools)这几个工具。

如果安装过 rust, 可以通过 cargo 安装。

安装 wkg 
```
cargo install wkg
```
安装 wit-bindgen
```
cargo install wit-bindgen-cli
```
安装 wasm-tools
```
cargo install wasm-tools
```
安装 WASI SDK,可以通过[这里](https://github.com/WebAssembly/wasi-sdk/releases)下载已经预编译的。

本文介绍如何通过 wasmtime 命令行运行 component model 文件。

1. 首先定义 command.wit 文件
```
package example:component;

world example {
    include wasi:cli/imports@0.2.4;
    export wasi:cli/run@0.2.4;
}
```
上面只导出了wasi:cli/run 函数，并引入了在命令行运行的依赖。

2. 使用 wkg 生成 wasm 文件
```
wkg wit build --wit-dir ./
```
此时会生成 example:component.wasm 文件。

3. 使用 wit-bindgen 生成绑定的代码
```
wit-bindgen c example:component.wasm
```
输出
```
Generating "example.c"
Generating "example.h"
Generating "example_component_type.o"
```
可以看到生成了接口文件。从 `example.h` 中可以找到 导出函数的定义。
```c
 // Exported Functions from `wasi:cli/run@0.2.4`
 bool exports_wasi_cli_run_run(void);

```
我们现在要做的是实现这个函数。

4. 实现 exports_wasi_cli_run_run 函数
创建 command.c 文件，并且写入
```c
#include "example.h"
#include <stdio.h>
bool exports_wasi_cli_run_run()
{
    printf("run command component model\n");
    return true;
}

```

5. 编译生成 wasm 文件
使用 wasi-sdk 中的 clang 进行编译。
```
 ~/wasi-sdk-22.0/bin/clang command.c example.c example_component_type.o -o command-core.wasm -mexec-model=reactor
```

6. 生成 component 文件
下载 [wasi_snapshot_preview1.reactor.wasm](https://github.com/bytecodealliance/wasmtime/releases/download/v17.0.0/wasi_snapshot_preview1.reactor.wasm) 并将其重命名为 `wasi_snapshot_preview1.wasm`。 通过这个文件来导入使用 wasi api 的函数。
```
wasm-tools component new command-core.wasm --adapt wasi_snapshot_preview1.wasm -o command-component.wasm

```
可以通过`wasm-tools component wit command-component.wasm` 查看 component 文件的内容。

7. 运行 component 文件
```
wasmtime run command-component.wasm
```
输出
```
run command component model
```



