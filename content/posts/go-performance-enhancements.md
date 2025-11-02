+++
date = '2025-11-02T07:59:41+08:00'
draft = false 
title = 'GO性能提升的相关改动'
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
+++
随着GO版本的迭代，GO语言的性能也在不断提升。下文梳理下GO种提升性能的几处改动。

## math/rand/v2
Go 1.22 版本引入，提供了一个新的随机数生成器，性能更好。
代码测试如下：
```go
package main

import (
	"math/rand"
	randV2 "math/rand/v2"
	"testing"
)

func BenchmarkRand(b *testing.B) {
	b.StartTimer()
	for range b.N {
		rand.Intn(100)
	}
}

func BenchmarkRandV2(b *testing.B) {
	b.StartTimer()
	for range b.N {
		randV2.IntN(100)
	}
}
```
测试结果

```shell
goos: darwin
goarch: amd64
pkg: test
cpu: Intel(R) Core(TM) i7-8850H CPU @ 2.60GHz
BenchmarkRand-12                     	84788529	        14.24 ns/op
BenchmarkRandV2-12                   	154128462	         7.790 ns/op
```
math/rand/v2 不需要设置随机种子，使用加密安全的随机数生成器。

## Go maps with Swiss Tables
Go 1.24版本引入，使用 Swiss Table 的理念，全新重写了map。直接使用Go1.24 版本的话，是与之前的map使用方式完全兼容，自动切换到新的map实现。使用swiss table 的好处体现在：

* 提升性能：Swiss Table 通过优化数据布局和访问模式，提高了map的查找和插入性能。充分利用了现代CPU特性，向量化的高效的位计算。
* 优化内存使用，尤其是大型map下，内存占用更小。

官方博客的原理说明参考这里。[Go maps with Swiss Tables](https://go.dev/blog/swisstable)。

这里有做具体的测试比较。参考[这里](https://www.bytesizego.com/blog/go-124-swiss-table-maps?__readwiseLocation=)。这里看到查找和插入性能都有明显的提升，但是删除性能有下降。

这个[博文](https://www.datadoghq.com/blog/engineering/go-swiss-tables/?__readwiseLocation=)从生成实践的角度说明了新map带来的性能提升和内存占比。并发现了一处bug，推荐使用1.24.6之后的版本。

## 容器感知的 GOMAXPROCS
Go 1.25 中引入。 GOMAXPROCS 可以控制GMP模型中的P的数量，默认值为运行时的CPU核心数。但是在容器环境中，由于容器的cpu 只是宿主机的一部分，之前 GOMAXPROCS 会设置成宿主机的CPU数量，这实际会带来性能问题。之前有两种解法：

* 这篇[文章](https://kanishk.io/posts/cpu-throttling-in-containerized-go-apps/?__readwiseLocation=)详细说明了这个问题和解决方法。是通过设置 GOMAXPROCS 环境变量，引用 cpu limit 来控制的。
* 通过这个[库](https://github.com/uber-go/automaxprocs) 来实现。

那从1.25版本之后，GO 官方库就可以根据容器环境的cpu limit 来设置 GOMAXPROCS 了, 而无需额外的设置。
如果想看下官方支持的详细讨论，参考[这里](https://github.com/golang/go/issues/73193#top)。

## encode/json/v2
Go 1.25 版本作为实验特性引入，需要设置GOEXPERIMENT=jsonv2 来开启。 

GOEXPERIMENT=jsonv2 开启后，marshal 和 unmarshal 会使用新的实现，性能会有提升。 但是并不完全兼容。尤其是对于 map 的处理，原有的实现会对map进行key排序之后，再编码。但是新版本不会。这样的话，对于map生成的字符串，不能保证完全一致。新版本也会slice 或者map 为nil 的情况的处理也不同，新版本的实现更符合json 的规范。

详细的介绍参考官方博客[Go 1.25: JSON Encoding and Decoding](https://go.dev/blog/jsonv2-exp?__readwiseLocation=)


## Green Tea 垃圾回收算法
目前GO 1.25 作为实验特性引入，构建时需要设置GOEXPERIMENT=greenteagc。

GO 使用两阶段的垃圾回收算法，分别是标记和清除。影响性能的主要是标记阶段。这篇[博文](https://go.dev/blog/greenteagc?__readwiseLocation=) 详细解释了Green Tea 算法的运行原理和性能优势。主要体现在：
* 通过扫描objects， 改成扫描 page，减少了扫描的次数。 
* 通过对page 的扫描，充分利用的缓存的局部特性。
* 扫描的过程还可以利用现代CPU特性，使用高效的位计算。