---
title: 如何确保一个协程在超时后顺利退出
tags: 
- golang
- 编程
date: 2022-11-22 08:00:00
---

Go 中的协程由于其非常易于使用的特性，在实际的使用中被广泛的应用于各个场中心。在有些场的使用可能并不是很恰当，甚至在特定的场景下定带来其他的问题。

## 一个场景

在 Go 中，通过 crontab 来调度一个任务 AsyncTask() 来处理一些异步工作，调度器每分钟调度 1 次。

## 问题

如果 AsyncTask 的执行时间，超过了调度间隔，而恰好，AsyncTask 的处理又比较占用系统资源，那么就会有大问题。

资源的占用会进一步延长 AsyncTask 的处理是间，如此更形成了恶性循环，直至耗尽全部资源。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h8doqt4xpuj30lp0avaac.jpg)

## 超时结束

提到超时，我们首先一定会想到 `context.WithTimeout`，它提供了简单的方法，可以轻而易举的实现超时功能，于下，我们可以写下如下的代码。

```go
func AsyncTask() {  
   ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)  
   defer cancel()  
  
   select {  
   case <-ctx.Done():  
      fmt.Println("AsyncTask has done")  
   default:  
      fmt.Println("AsyncTask is running")  
      time.Sleep(time.Second * 3)  
      return  
   }  
}
```

确实，这是一个超时的任务处理器，且考虑了两个方面：

- 当 AsyncTask 任务提前处理完成时，退出
- 当 AsyncTask 处理完，但时间超过了 1 分钟时，退出

看似能很完美的运行， AsyncTask 运行时间小于 1 分钟，没有问题。

但是回到我们上面的问是，当 AsyncTask 运行时间远远超过 1 分钟时，我们前面提到的问是还是存在的，前面的任务运行没有结束，后面的任务又到来了。

## 超时后协程会退出吗

我们可以运行一个 test，在结束时打印一下当前进程空间中的所有协程数量以判断协程是否正确退出：

```go
  
func TestTimeoutContextWrapper(t *testing.T) {  
   t.Helper()  
   for i := 0; i < 1000; i++ {  
      go AsyncTask()  
   }  
   time.Sleep(time.Second * 4)  
   t.Log(runtime.NumGoroutine())  
}

```

将 AsyncTask 使用协程方式，异步的运行 1000 次，AsyncTask 内部会睡眠 3s 以模拟实际的业务处理耗时。主程序睡眠 4s，最后再打印所有的协程数量。

运行，并等待 4 秒之后，得到如下的输出：

```bash
    timeout_context_test.go:32: goroutines:  2
--- PASS: TestTimeoutContextWrapper (4.00s)
PASS
```

证明在超时 3s 后，所以创建的协程都已正确退出。

那，如果 AsyncTask() 运行时间超过 3s 呢？假设以阻塞 IO 方式运行了 10s，我再次来模拟一下：

```bash
    timeout_context_test.go:32: goroutines:  1002
--- PASS: TestTimeoutContextWrapper (4.00s)
PASS
```

测试结果证明了我们一开始的问题：在同步阻塞 IO 情况下，select 循环也需要至少等待一次主业务逻辑执行完成（10s），在下次循环时才会检测到超时，然后协程退出。

## 如何在超时后直接退出

我们试试异步非阻塞 IO。将上面的测试代码稍微改一改，把同步 IO 替换成异步 IO 的：

```go
func AsyncTask2(c chan bool) {  
   ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)  
   defer cancel()  
   select {  
   case <-ctx.Done():  
      fmt.Println("AsyncTask2 has done")  
      c <- true  
   default:  
      fmt.Println("AsyncTask2 is running")  
   }}  
  
func AsyncTaskRunner() {  
   defer func() {  
      fmt.Println("AsyncTaskRunner has done")  
   }()  
   done := make(chan bool, 1)  
   go AsyncTask2(done)  
  
   select {  
   case <-done:  
      fmt.Println("AsyncTask2 done")  
   case <-time.After(time.Second * 2):  
      fmt.Println("AsyncTaskRunner timeout")  
      return  
   }  
}  
  
func TestAsyncTaskRunner(t *testing.T) {  
   t.Helper()  
   for i := 0; i < 5; i++ {  
      go AsyncTaskRunner()  
   }  
   time.Sleep(time.Second * 3)  
   t.Log("goroutines: ", runtime.NumGoroutine())  
}
```

再次运行测试，结果如下：

```bash
    timeout_context_test.go:72: goroutines:  2
--- PASS: TestAsyncTaskRunner (3.00s)
PASS
```

没错，使用异步 IO，在 runner 结束之后，AsyncTask2 也结束了。

## 通过业务逻辑保证，以解决问题

由于 go 的协程没有主协程/子协程一说，协程一旦创建之后都会平等的接受调度与运行。因此我们并不能直接的结束调一个已创建的子协程。

于是根据上面的异步的思路，进一步封装了一个如下的异步任务限时处理器：

```go
type TimeoutTaskHandler interface {  
   HandleTimeoutTask(ctx context.Context) bool  
}  
  
// TimeoutContextWrapper 一个简单的超时处理器  
// 处理器会在指定的最大时间内执行任务  
func TimeoutContextWrapper(ctx context.Context, timeoutSec int, handler TimeoutTaskHandler) {  
   defer func() {  
      if err := recover(); err != nil {  
         log.Error(fmt.Errorf("timeout context wrapper panic: %s", err))  
      }   
   }()  
   
   ctx, cancel := context.WithTimeout(ctx, time.Duration(timeoutSec)*time.Second)  
   defer cancel()  
  
   var exit = make(chan bool, 1)  
   go func() {  
      for {  
         select {  
         case <-ctx.Done():  
            exit <- true  
            return  
  
         default:  
            // handler 的运行时间可能会超过 timeoutSec            
            // 所以需要通在后面配置一个超时时间，超过 timeoutSec 就退出  
            haveDone := handler.HandleTimeoutTask(ctx)  
            if haveDone {  
               exit <- true  
               return  
            }  
         }      
	 }   
   }()  
   
   // 开始一个计时器  
   // 超过 timeoutSec 或者 未到 timeoutSec 但是 handler 决定退出时，结束本次处理周期  
   select {  
   case <-exit:  
      return  
   case <-time.After(time.Duration(timeoutSec) * time.Second):  
      return  
   }  
}
```

如上代码其内部的原理和前面模拟的测试代码大同小异。要想让上面的代码按预期方式正常运行，有个逻辑需要在业务层面来保证：

`HandleTimeoutTask()` 接口的实现，用来处理一个最小单位的任务，并且会在每次循环中调用。这就意味着在实现内部需要有机制来避免死循环且保证“向前”推进任务进程，同时处理时间不能超过整个调度周期的时间。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h8dov0bwxpj30lp0avq33.jpg)

此代码目前在线上运行良好，顺利的解决了一开始提出的问题。

## 总结

- 无法直接通过 kill 机制结束一个已创建的协程
- 建议协程中要有保障退出的机制
- 建议使用异部 IO，如果写成处于阻塞中，也是需要等至结束之后才能退出
- 避免在协程中使用死循环（或要能退出）
- 如果需要使用循环来处理业务，需要考虑极端情况，推荐将耗时的长任务拆分为多步执行
