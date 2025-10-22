+++
date = '2025-10-22T21:34:32+08:00'
draft = true
title = 'io_uring 学习'
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
io_uring 是 Linux 内核提供的异步 IO 接口，它可以用于替代传统的同步 IO 接口，如 `read`、`write` 等。io_uring 提供了一种基于事件循环的异步 IO 模型，能够在单个线程中处理多个 IO 操作，从而提高 IO 操作的并发性和效率。io_uring 从内核5.1版本开始引入，在这之前，AIO 提供了异步IO 的功能。但是存在诸多的限制，包括       

* 只能作用于文件IO, 不能作用于网络IO. 在文件IO 中，只能使用O_DIRECT标记打开文件，无法使用buffer cache 。这样使用的范围比较少，只有在数据库领域有应用。
* 不能完全提供异步功能，如果元数据不可用时，也得同步等待
* 不能完全做到zero-copy
* 在某些设备场景中，无法做到真正异步，也得同步等待

io_uring 使用统一的接口解决异步IO问题，包括文件IO 和 网络IO。非阻塞的接口，减少了线程上下文的切换。并且可以使用系统的buffer cache 。
io_uring 主要的数据结构包括两个ring buffer 。
![img](images/wechat_2025-10-22_215429_869.png)

* Submission Queue (SQ)：用于提交IO请求的环形缓冲区。应用程序将IO请求放入SQ中，内核从SQ中获取请求并处理。 IO 请求放入tail 中， 内核通过head 读取进行处理。
* Completion Queue (CQ)：用于存储IO操作完成的结果。内核将IO操作的完成结果放入tail中，应用程序从head中获取结果并处理。

io_uring 提供一系列系统调用的接口，目前可以通过[liburing](https://github.com/axboe/liburing) 库来更方便的使用。

在这个库的examples 目录下自带了很多的使用例子。具体的例子也可以通过[这里](https://unixism.net/loti/tutorial/index.html)有更多的解释。

## liburing-cp 
这是一个文件IO的例子，使用liburing 实现的 cp 命令。从[这里](https://github.com/axboe/liburing/blob/master/examples/io_uring-cp.c) 看到最新的代码。

初始化 io_uring 实例， entries 是 SQ 的大小， 一般设置为 2 的幂次方。 通常情况下CQ 是SQ 大小的2倍。

```c
static int setup_context(unsigned entries, struct io_uring *ring)
{
	int ret;

	ret = io_uring_queue_init(entries, ring, 0);
	if (ret < 0) {
		fprintf(stderr, "queue_init: %s\n", strerror(-ret));
		return -1;
	}

	return 0;
}
```


## 参考
* [https://medium.com/oracledevs/an-introduction-to-the-io-uring-asynchronous-i-o-framework-fad002d7dfc1](https://medium.com/oracledevs/an-introduction-to-the-io-uring-asynchronous-i-o-framework-fad002d7dfc1)
* [https://www.bilibili.com/video/BV1JB4y1R7QY/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=e91f1e0e2fd6e7afd65d0e39c8cb3c68](https://www.bilibili.com/video/BV1JB4y1R7QY/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=e91f1e0e2fd6e7afd65d0e39c8cb3c68)
* [https://kernel.dk/io_uring.pdf](https://kernel.dk/io_uring.pdf)