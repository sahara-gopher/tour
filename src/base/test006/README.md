# Select 语句

当多个需要从多个chan中读取或写入时，会先轮询一遍所有的case，然后在所有处于就绪（可读/可写）的chan中随机挑选一个进行读取或写入操作，并执行其语句块。如果所有case都未就绪，则执行default语句，如未提供default语句，则当前协程被阻塞。

```

package main

import (
	"fmt"
	"time"
)

func main() {	
	select {
	case resp := <-AsyncCall(50):
		fmt.Println(resp)
	case resp := <-AsyncCall(200):
		fmt.Println(resp)
	case resp := <-AsyncCall2(1000):
		fmt.Println(resp)
	}
}

func AsyncCall(t int) <-chan int {
	c := make(chan int, 1)
	go func() {
		time.Sleep(time.Microsecond * time.Duration(t))
		c <- t
	}()
	return c
}

func AsyncCall2(t int) <-chan int {
	c := make(chan int, 1)
	go func() {
		time.Sleep(time.Microsecond * time.Duration(t))
		c <- t
	}()
	// gc or some other reason cost some time
	time.Sleep(200 * time.Microsecond)
	return c
}

```
[代码](./main.go)

# 温柔的陷阱

这段代码运行的结果会是200和50两种结果随机出现。

这就引出了select的公平性问题。

凭什么第二个调用能在完成时间落后的情况下被select选中？select不是能保证先就绪的case被先执行吗？难道是golang的bug？

相信仔细看过代码你就能发现，问题主要出在c的异步调用AsyncCall2上，由于这个异步调用本身的执行的时间为200ms，超过了前两个的任务执行时间。而根据我们之前提到的，select语句在判断chan就绪之前，是会把所有的case语句的判断语句执行一遍的，这就包括了在case语句中的函数调用，因此上边的代码执行逻辑为：AsyncCall(50)--->AsyncCall(200)--->AsyncCall3(3000)，然后再等待三者返回的chan就绪，谁先就绪就执行谁；如果三者同时就绪，则随机挑选一个执行；没有就绪就阻塞直到有一个就绪。显然当AsyncCall3(3000)执行完成时，前两个异步任务已经完成，因此在判断chan就绪时就是随机选择前两个chan了，故而打印结果为50和200随机出现。