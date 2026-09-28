
## 什么是协程？

协程（Goroutine）可以理解为由 Go 程序自己管理的“轻量级小任务”。在函数调用前加上 `go`，这个函数就能与其他任务并发执行；协程比操作系统线程更轻，因此 Go 程序可以同时运行大量协程。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func() {
		defer wg.Done()
		fmt.Println("协程正在执行任务")
	}()

	wg.Wait() // 等待协程执行完毕
}
```

这里的 `go func()` 启动了一个新协程，主协程通过 `wg.Wait()` 等待它完成，避免程序提前退出。
