# Go Interview-09


## Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量

Golang中Goroutine可以通过Channel进行安全读写共享变量

## 无缓冲Chan的发送和接收是否同步

ch := make(chan int)    无缓冲的channel由于没有缓冲，发送和接收需要同步
ch := make(chan int, 2) 有缓冲channel不要求发送和接收操作同步

- channel无缓冲时，发送堵塞直到数据被接收，接收堵塞直到读到数据
- channel有缓冲时，当缓冲满时发送堵塞，当缓冲空时接收堵塞

## go语言的并发机制以及它所使用的CSP并发模型

CSP模型是上世纪七十年代提出的，不同于传统的多线程通过共享内存来通信，CSP讲究的是"以通信方式来共享内存"。用于描述两个独立的并发实体通过共享的通信channel(管道)进行通信的并发模型。CSP中channel是第一类对象，它不关注发送消息的实体，而关注与发送消息时使用的channel

Golang中channel是被单独创建并且可以在进程之间传递，它的通信模式类似于boss-worker模式，一个实体通过将消息发送到channel中，然后又监听这个channel的实体处理，两个实体之间是匿名的，这个就是实现实体中间的解耦，其中channel是同步的一个消息被发送到channel中，最终是一定要被另外的实体消费掉的，在实现原理上类似一个堵塞的消息队列

Goroutine是Golang实际并发执行的实体，它底层是使用协程(coroutine)实现并发的，coroutine是一种运行在用户态的用户线程，类似于greenthread，go底层选择使用coroutine的出发点是因为，它具有以下特点：

- 用户空间 避免了内核态和用户态的切换导致的成本
- 可以由语言和框架进行调度
- 更小的栈空间允许创建大量的实例

Golang中Goroutine的特性

Golang内部有三个对象：P对象(processor)代表上下文(或者可以认为是cpu)，M(work thread)代表工作线程，G对象(goroutine)

正常情况向下一个cpu对象启动一个工作线程，线程去检查并执行goroutine对象。碰到goroutine对象阻塞的时候，会启动一个新的工作线程，以充分利用cpu资源。所以有时候线程对象会比处理器对象多很多

G(Goroutine)：我们所说的协程，为用户级的轻量级线程，每个Goroutine对象中的sched保存着其上下文信息

M(Machine)：对内核级线程的封装，数量对象真实的CPU数(真正干活的对象)

P(Processor)：即为G和M的调度对象，用来调度G和M之间的关联关系，其数量可通过GOMAXPROCS()来设置，默认为核心数

在单核情况下，所有Goroutine运行在同一个线程(M0)中，每一个线程维护一个上下文(P)，任何时刻，一个上下文中只有一个Goroutine，其他Goroutine在runqueue中等待

一个Goroutine运行完自己的时间片后，让出上下文，自己回到runqueue中等待。

当正在运行的G0阻塞的时候(可能需要IO)，会再创建一个线程(M1)，P转到新的线程中去运行。

当M0返回时，它会尝试从其他线程中'偷'一个上下文过来，如果没有偷到，会把Goroutine放到Global runqueue中去，然后把自己放入线程缓存中。上下文会定时检查Gloabl runqueue。

Golang是为并发而生的语言，Go语言是为数不多的在语言层面实现并发的语言；也正是Go语言的并发特性，吸引了全球无数的开发者

Golang的CSP并发模型是通过Goroutine和Channel来实现的

Goroutine是Go语言中并发的执行单位。有点抽象，其实就是和传统概念上的"线程"类似，可以理解为"线程"，Channel是Go语言中各个并发结构体(Goroutine)之前的通信机制。通常Channel，是各个Goroutine之间通信的"管道"，有点类似于Linux中的管道

通信机制channel也很方便，传数据用channel <- data，取数据用 <- channel

在通信过程中，传数据channel <- data 和取数据 <- channel必然会成对出现，因为这边传，那边取，两个goroutine之间才会实现通信

而且不管传还是取，必阻塞，直到另外的goroutine传或者取为止

## Golang中常用的并发模型

Golang中常用的并发模型有三种：

- 通过channel通知实现并发控制

    无缓冲的通道指的是通道的大小为0，也就是说，这种类型的通道在接受前没有能力保存任何值，它要求发送goroutine和接收goroutine同时准备好，才可以完成发送和接收的操作

    从上面无缓冲的通道定义来看，发送goroutine和接收goroutine必须是同步的，同时准备后，如果没有同时准备好的话，先执行的操作就会阻塞等待，直到另一个相对应的操作准备好为止。这种无缓冲通道我们也称为同步通道

    ```go
        func main() {
            ch := make(chan struct{})

            go func() {
                fmt.Printlf("start working") 
                time.Sleep(time.Second * 1)
                ch <- struct{} {}
            } ()

            <- ch

            fmt.Println("finished")
        }
    ```

    当主goroutine运行到 <- ch 接收channel的值的时候，如果该channel中没有数据，就会一直阻塞等待，直到有值，这样就可以实现简单的并发控制

- 通过sync包中的WaitGroup实现并发控制

    Goroutine是异步执行的，有的时候为了防止在结束main函数的时候结束掉Goroutine，所以需要同步等待，这个时候就需要用WaitGroup了，在sync包中提供了WaitGroup，它会等待它收集的所有goroutine任务全部完成。在WaitGroup里主要有三个方法

    1. Add 可以添加或者减少goroutine的数量
    2. Done 相当于Add(-1)
    3. Wait 执行后会阻塞主线程，直到WaitGroup里的值减至0

    在主goroutine中Add(delta int)索要等待goroutine的数量。在每一个goroutine完成后Done()表示这一个goroutine已经完成，当所有的goroutine都完成后，在主goroutine中WaitGroup返回

- 在Go 1.7 以后引入了强大的Context上下文，实现并发控制

    通常在一些简单场景下使用channel和WaitGroup已经足够了，但是当面临一些复杂多变的网络并发场景下channel和WaitGroup显得有些力不从心了。比如一个网路请求Request，每个Request都需要开启一个goroutine做一些事情，这些goroutine又可能开启其他的goroutine，比如数据库和RPC服务。所以我们需要一种可以跟踪goroutine的方案，才可以达到控制他们的目的，这就是Go语言为我们提供的Context，称之为上下文非常贴切，它就是goroutine的上下文。它是包括一个程序的运行环境、现场和快照等。每个程序要运行时都需要知道当前程序的运行状态，通常Go将这些封装在一个Context里，再将它传给要执行的goroutine。

    context包主要是用来处理多个goroutine之间共享数据，及多个goroutine的管理

## JSON标准库对 nil slice和 空 slice的处理是一致的吗

首先JSON标准库对 nil slice 和空slice的处理是不一致的

通常错误的用法，会报数组越界的错误，因为只是声明了slice，却没有个实例化的对象

```go
var slice []int
slice[1] = 0
```

此时slice的值是nil，这种情况可以用于需要返回slice的函数，当函数出现异常的时候，保证函数依然会有nil的返回值

empty slice是指slice不为nil，但是slice没有值，slice底层的空间是空的，此时的定义如下

```go
slice := make([]int, 0)
slice := []int{}
```

当我们查询或者处理一个空的列表的时候，这非常有用，它会告诉我们返回的是一个列表，但是列表内没有任何值

总之，nil slice 和 empty slice 是不同的东西，需要我们加以区分

## 协程，线程，进程的区别

- 进程

    进程是具有一定独立功能的程序关于某种数据集合上的一次运行活动，进程是系统进行资源分配和调度的一个独立单位。每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。由于进程比较重量，占据独立内存，所以上下文进程间的切换开销(栈，寄存器，虚拟内存，文件句柄等)比较大，但相对比较稳定安全。

- 线程
    
    线程是进程的一个实体，是CPU调度和分派的基本单位，它是比进程更小的能独立运行的基本单位，线程自己基本上不拥有系统资源，只拥有一点在运行中必不可少的资源(如程序计数器，一组寄存器和栈)，但是它可与同属一个进程的其他线程共享进程所拥有的的全部资源。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据

- 协程

    协程是一种轻量级的线程，协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存在寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。

## 互斥锁，读写锁，死锁问题是怎么解决的

- 互斥锁

    互斥锁就是互斥变量mutex，用来锁住临界区的

    条件锁就是条件变量，当进程的某些资源要求不满足时就进入休眠，也就是锁住了。当资源被分配了，条件锁打开了，进程继续进行；读写锁也类似，用于缓冲区等临界资源能互斥访问的。

- 读写锁

    通常有些公共数据修改的机会很少，但其读的机会很多。并且在读的过程中会伴随着查找，给这种代码加锁会降低我们的程序效率。读写锁可以解决这个问题

    注意：写独占，读共享，写锁优先级高

- 死锁

    一般情况下，如果同一个线程先后两次调用lock，在第二次调用时，由于锁已经被占用，该线程会挂起等待别的线程释放锁，然而锁正是被自己占用着的，该线程又被挂起而没有机会释放锁，因此就永远处于挂起等待状态了，这叫做死锁(Deadlock)。另一种情况是：若线程A获得了锁1，线程B获得了锁2，这时线程A调用lock试图获得锁2，结果是需要挂起等待线程B释放锁2，结果是需要挂起等待线程B释放锁2，而这时线程B也调用lock试图获得锁1，结果是需要挂起等待线程A释放锁1，于是线程A和B都永远处于挂起的状态

    死锁产生的四个必要条件：
    1. 互斥条件：一个资源每次只能被一个进程使用
    2. 请求和保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放
    3. 不剥夺资源：进程已获得的资源，在未使用完之前不能强行剥夺
    4. 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。
    
    这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁

- 预防死锁

可以把资源一次性分配： (破坏请求和保持条件)

然后剥夺资源：  即当某进程新的资源未满足时，释放已占有的资源(破坏不可剥夺条件)

资源有序分配法：    系统给每类资源赋予一个编号，每一个进程按编号递增的顺序请求资源，释放则相反(破坏环路等待条件)

- 避免死锁

预防死锁的几种策略，会严重地损坏系统性能。因此在避免死锁时，要施加较弱的限制，从而获得较满意的系统性能。由于在避免死锁的策略中，允许进程动态地申请资源。因而，系统在进行资源分配之前预先计算资源分配的安全性。若此次分配不会导致系统进入不安全状态，则将资源分配给进程；否则进程等待。其中最具代表性的避免死锁算法是银行家算法。

- 检测死锁

首先为每个进程和每个资源指定一个唯一的号码，然后建立资源分配表和进程等待表

- 解除死锁

当发现有进程死锁后，便应立即把它从死锁状态中解脱出来，常采用的方法有

    1. 剥夺资源

        从其他进程剥夺足够的资源给死锁进程，以解除死锁状态
    2. 撤销进程

        可以直接撤销死锁进程或撤销代价最小的进程，直至有足够的资源可用，死锁状态消除为止，所谓代价是指优先级、运行代价、进程的重要性和价值等

##  Golang的内存模型，为什么小对象多了会造成gc压力

通常小对象过多会导致GC三色法消耗过多的GPU。优化思路，减少对象分配

## Data Race问题怎么解决，能不能加锁解决这个问题

同步访问共享数据是处理数据竞争的一种有效的方法。golang在1.1之后引入了竞争检测机制，可以使用go run -race 或者 go build -race来进行静态检测。其在内部的实现是，开启多个协程执行同一个命令，并且记录下每个变量的状态。

想要解决数据竞争的问题可以使用互斥锁sync.Mutex，解决数据竞争(Data race)，也可以使用管道解决，使用管道的效率比互斥锁高

## 什么是channel，为什么它可以做到线程安全

Channel是Go中的一个核心类型，可以把它看成一个管道，通过它并发核心单元就可以发送或者接受数据进行通讯(communication)，Channel也可以理解是一个先进先出的队列，通过管道进行通信

Golang的channel，发送一个数据到Channel和从一个Channel接收一个数据都是原子性的。而且Go的设计思想就是不通过共享内存来通信，而是通过通信来共享内存，前者就是传统的加锁，后者就是Channel。也就是说，设计Channel的主要目的就是在多任务之间传递数据，这当然是安全的。

## Golang GC时会发生什么

- 什么是垃圾回收

    内存管理是程序员开发应用的一大难题。传统的系统级编程语言(主要指c/c++)中，程序开发者必须对内存小心的进行管理操作，控制内存的申请和释放。因为稍有不慎，就可能产生内存泄露问题，这种问题不易发现并且很难定位，一直成为困扰程序开发者的噩梦，解决方法
    1. 内存泄露检测工具。这种工具的原理一般是静态代码扫描，通过扫描程序检测可能出现内存泄露的代码段。然而检测工具难免会有疏漏和不足，只能起到辅助作用
    2. 智能指针。这是c++中引入的自动内存管理方法，通过拥有自动内存管理功能的指针对象来引用对象，使程序员不用太关注内存的释放，而达到内存自动释放的目的。这种方法是采用最广泛的做法，但是对程序开发者有一定的学习成本(并非语言层面的原生支持)，而且一旦有忘记使用的场景依然无法避免内存泄露。
- Golang GC时会发生什么

Golang 1.5后，采取的是"非分代的，非移动的，并发的，三色的"标记清除垃圾回收算法

golang中的gc基本上是标记清除的过程

gc的过程一共分为四个阶段

1. 栈扫描(开始时STW)
2. 第一次标记(并发)
3. 第二次标记(STW)
4. 清除(并发)

整个进程空间里申请每个对象占据的内存可以视为一个图，初始状态下每个内存对象都是白色标记。

1. 先STW，做一些准备工作，比如enable write barrier。然后取消STW，将扫描任务作为多个并发的goroutine立即入队给调度器，进而被CPU处理
2. 第一轮扫描root对象，包括全局指针和goroutine栈上的指针，标记为灰色放入队列
3. 第二轮将第一步队列中的对象引用的对象置为灰色加入队列，一个对象引用的所有对象都只会并加入队列后，这个对象才能置为黑色并从队列之中以取出。循环往复，最后队列为空时，整个图剩下的白色内存空间即不可到达的对象，即没有被引用的对象。
4. 第三轮再次STW，将第二轮过程中新增对象申请的内存进行标记及(灰色)，这里使用write barrier(写屏障)去记录

Golang gc优化的核心就是尽量使得STW(Stop The World)的时间越来越短

## 并发编程概念是什么

并行是指两个或者多个事件在同一时刻发生；并发是指两个或者多个时间在同一时间间隔发生。

并行是在不同实体上的多个事件，并发是在同一实体上的多个事件。在一台处理器上"同时"处理多个任务，在多台处理器上同时处理多个任务。

并发偏重于多个任务交替执行，而多个任务之间有可能还是串行的。而并行是真正意义上的"同时执行"

并发编程是指在一台处理器上"同时"处理多个任务。并发是在同一实体上的多个事件。多个事件在同一时间间隔发生。并发编程的目的是充分利用处理器的每一个核，以达到最高的处理性能 

## 负载均衡的原理是什么

负载均衡是高可用网络基础架构的关键组件，通常用于将工作负载分布到多个服务器来提高网站、应用、数据库或其他服务的性能和可靠性。负载均衡，其核心就是网络流量分发，分很多纬度。

负载均衡通常是分摊到很多个操作单元上执行，例如Web服务器，FTP服务器、企业关键应用服务器和其他关键任务服务器等，从而共同完成工作任务

负载均衡是建立在现有网络结构之上，它提供了一种廉价有效透明的方法扩展网络设备和服务器的带宽、增加吞吐量、加强网络数据处理能力、提高网络的灵活性和可用性


