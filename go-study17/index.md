# Go Study17


## select语句介绍
Go语言中的select语句用于监控并选择一组case语句执行相应的代码。它看起来类似于switch语句，但是select语句中所有case中的表达式必须是channel的发送或者接受操作。一个典型的select的使用示例如下：
```go
select {
case <- ch1:
    fmt.Println("liwenzhou.com")
case ch2 <- 1:
    fmt.Println("qimi")
}
```
Go语言中select关键字也能够让当前goroutine同时等待ch1的可读和ch2的可写，在ch1和ch2状态改变之前，select会一直堵塞下去，直到其中的一个channel转为就绪状态时执行对应的case分支的代码。如果多个channel同时就绪的话则随机选择一个case执行。

处理上面展示的典型实例外，接下来我们逐一介绍一些select的特殊示例

## 空select
空select指的是内部不包含任何case，例如：
```go
select {

}
```
空的select语句会直接堵塞当前的goroutine，使得该goroutine进入无法唤醒的永久休眠状态

## 只有一个case
如果select中只包含一个case，那么该select就变成了一个堵塞的channel读/写操作。

```go
select {
case <- ch1:
    fmt.Println("liwenzhou.com")
}
```

上面的代码，当ch1可读时会执行打印操作，否则就会堵塞

## 有default语句
如果select中还可以包含default语句，用于当其他case都不满足时执行一些默认操作。

```go
select {
case <- ch1:
    fmt.Println("liwenzhou.com")
default:
    time.Sleep(time.Second)
}
```

上面的代码，当ch1可读时会执行打印操作，否则就执行default语句中的代码，这里就相当于做了一个非堵塞的channel读取操作。

## 总结
1. select不存在任何的case：永久堵塞当前goroutine
2. select只存在一个case：堵塞的发送/接收
3. select存在多个case：随机选择一个满足条件的case执行
4. select存在default，其他case都不满足时，执行default语句中的代码

## 如何在select中实现优先级
已知，当select存在多个case时会随机选择一个满足条件的case执行

现在我们有一个需求：我们有一个函数会持续不间断地从ch1和ch2中分别接收任务1和任务2，如何确保当ch1和ch2同时达到就绪状态时，优先执行任务1，在没有任务1的时候再去执行任务2

```go
func worker (ch1, ch2 <-chan int, stopCh chan struct{}) {
    for {
        select {
        case <-stopCh:
            return
        case job1 := <-ch1:
            fmt.Println(job1)
        default:
            select {
            case job2 := <-ch2:
                fmt.Println(job2)
            default:
            }
        }
    }
}
```

上面的代码通过嵌套两个select实现了"优先级"，看起来是满足题目要求的。但是这代码有点问题，如果ch1和ch2都没有达到就绪状态的话，整个程序不会堵塞而是进入死循环。

```go
func worker2(ch1, ch2 <-chan int, stopCh chan struct{}) {
    for {
        select {
        case <-stopCh:
            return
        case job1 := ch1:
            fmt.Printl(job1)
        case job2 := ch2:
        priority:
            for {
                select {
                case job1 := <-ch1:
                    fmt.Println(job1)
                default:
                    break priority
                }
            }
            fmt.Println(job2)
        }
    }
}
```
这一次不仅使用了嵌套的select，还组合使用了for循环和LABEL来实现题目的要求。上面的代码在外层select选中执行job := <-ch2时，进入到内层select循环尝试执行job1 := <-ch1，当ch1就绪时就会一直执行，否则跳出内层select

## 实际应用场景
K8s的controller中就有关于上面这个技巧的实际使用示例，这里在关于select中实现优先级相关代码的关键处都已添加了注释。

```go
// kubernetes/pkg/controller/nodelifecycle/scheduler/taint_manager.go 
func (tc *NoExecuteTaintManager) worker(worker int, done func(), stopCh <-chan struct{}) {
	defer done()

	// 当处理具体事件的时候，我们会希望 Node 的更新操作优先于 Pod 的更新
	// 因为 NodeUpdates 与 NoExecuteTaintManager无关应该尽快处理
	// -- 我们不希望用户(或系统)等到PodUpdate队列被耗尽后，才开始从受污染的Node中清除pod。
	for {
		select {
		case <-stopCh:
			return
		case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
			tc.handleNodeUpdate(nodeUpdate)
			tc.nodeUpdateQueue.Done(nodeUpdate)
		case podUpdate := <-tc.podUpdateChannels[worker]:
			// 如果我们发现了一个 Pod 需要更新，我么你需要先清空 Node 队列.
		priority:
			for {
				select {
				case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
					tc.handleNodeUpdate(nodeUpdate)
					tc.nodeUpdateQueue.Done(nodeUpdate)
				default:
					break priority
				}
			}
			// 在 Node 队列清空后我们再处理 podUpdate.
			tc.handlePodUpdate(podUpdate)
			tc.podUpdateQueue.Done(podUpdate)
		}
	}
}
```
