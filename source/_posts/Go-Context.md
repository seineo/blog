---
title: Go Context
date: 2024-06-27 21:18:03
tags:
---

## 为什么需要context？

在Go语言中，对于IO密集、并行计算等任务，我们一般都使用轻量的协程来提高程序的效率。比如我们用Go写了一个http服务器，那么一般一个请求就用一个协程处理，在这个请求中我们可能需要去请求数据库、请求别的服务等等，如果一切顺利就最好不过，但是很可能碰到以下情况：

1.  用户退出（比如关闭网页）

2.  下游服务迟迟不返回

对于这些情况，如果我们任由协程继续运行，既让无用任务占用了资源，也很可能影响用户的体验。因此我们需要解决这样一个问题：==如何在协程间传递状态与数据?== 

“根据CSP的思想，用channel不就好了。”

是的，我们可以用channel来解决这个问题。比如我们想实现三秒超时则关闭协程：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(done chan bool, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-done:
			fmt.Println("Worker goroutine is closing.")
			return 
		default:
			// 执行工作逻辑
			fmt.Println("Worker is working...")
			time.Sleep(1 * time.Second)
		}
	}
}

func main() {
	var wg sync.WaitGroup

	done := make(chan bool)

	wg.Add(1)
	go worker(done, &wg)

	// 使用time.After设置三秒后的超时
	timeout := time.After(3 * time.Second)

	// 主goroutine等待worker完成或者超时
	select {
	case <-timeout:
		fmt.Println("Timeout reached. Closing worker goroutine.")
		close(done) // 发送信号到done channel，尝试关闭worker
	case <-done:
		fmt.Println("Worker finished before timeout.")
	}

  // 等待所有goroutine完成
	wg.Wait()
}
```


​    

这样当然可以，但是一般的应用开发会有多层协程调用，比如A -> B -> C，如果A想关闭B，当B接收到终止信号并决定退出时，B应该也要负责通知C也退出。那么每个协程都需要有上述代码这样的channel处理逻辑，这对开发人员来说负担较重。

因此，Go 1.7版本引入了context。

> Context provides a means of transmitting deadlines, caller cancellations, and other request-scoped values across API boundaries and between processes.

## 如何使用context？

我们以http服务的优雅关闭为例来简要阐述下context的常见用法。

当收到进程退出的信号时，我们捕获信号，来执行协程的优雅关闭：

```go
ctxWeb, cancelWeb = context.WithCancel(context.Background())
svr := api_server.NewApiServer(ctxWeb, p)
svr.Run()

sigs := make(chan os.Signal, 1)
signal.Notify(sigs,
    syscall.SIGHUP,
    syscall.SIGINT,
    syscall.SIGTERM,
    syscall.SIGQUIT)
defer signal.Stop(sigs)
_ = <-sigs
cancelWeb()
```

`cancelWeb`执行后，`context.Done()`返回的通道则会被关闭，那么由于通道的广播机制，select也就会收到事件，来执行http.Server的优雅关闭函数：

```go
func (h *ApiServer) Run() error {
  	errCh := make(chan error)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	select {
	case e := <-errCh:
		return e
	case <-h.context.Done():
		return srv.Shutdown(h.context) // Shutdown gracefully shuts down the server without interrupting any active connections.
	}
}
```



## context是如何运作的？

context的常用接口有以下三个：

*   WithCancel：创建可取消的context，返回cancelFunc用于手动调用取消。

*   WithTimeout：指定超时的相对时间，超时则自动取消。

*   WithDeadline：指定超时的绝对时间，超时自动取消。

其中WithTimeout就是对WithDeadline的封装：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

三个接口重点都是如何取消协程context与调用的所有子孙context，这其实涉及两个问题：

1.  如何链接父子context？

2.  如何取消当前context与所有子孙context？

对于第一个问题，由`propagateCancel`函数解决：

1.  如果父context已经被cancel，则cancel子context，然后直接返回；

2.  否则将子context加入父context的map字段children中。

对于第二个问题，由`cancel`函数解决：

1.  如果context已经被cancel，直接返回；

2.  否则设置cancel的error和cause，然后关闭done这个channel；【由于Go channel的广播机制（当一个channel被close时，所有通过select监听这个channel IO事件的goroutine，都会收到相关事件），监听的协程可以做相应的处理】

3.  递归处理子context。

上述内容涉及一些细节问题：

1.  如何确定context是否已经被cancel？

判断方法是看err是否为空，因为cancel是一定要设置error的，如果error不为空则是被cancel了。

2.  done channel什么时候分配？

并不是在创建context结构体时就分配空间，而是在调用Done()命令获取done channel时才分配（懒汉式）。

## 参考

1.  [https://golang.design/go-questions/stdlib/context/why/](https://golang.design/go-questions/stdlib/context/why/)

2.  [https://go.dev/blog/context-and-structs](https://go.dev/blog/context-and-structs)
