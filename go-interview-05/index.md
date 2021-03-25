# Go Interview 05

<!--more-->
## Go的select机制
1. select机制用来处理异步IO问题
2. select机制最大的一条限制就是每个case语句里必须是一个IO操作
3. golang在语言级别支持select关键字
## 进程、线程、协程之间的区别
进程是资源的分配和调度的一个独立单元，而线程是CPU调度的基本单位

同一个进程中可以包含多个线程

进程结束后，她拥有的所有线程都将销毁，而线程的结束不会影响同个进程中其他线程的结束。

线程共享整个进程的资源(寄存器、堆栈、上下文)，一个进程至少包含一个线程

进程的创建使用fork或vfork，而线程的创建使用pthread_create

线程中执行时，一般要进行同步和互斥，因为他们共享同一进程的所有资源

进程是资源分配的单位

线程是操作系统调度的单位

进程切换需要的资源最大，效率很低。线程切换需要的资源一般，效率一般。协程切换需要的资源很小，效率高。多进程，多线程根据cpu核数不一样可能是并行也可能是并发的。协程的本质就是使用当前进程在不同的函数代码中切换执行，可以理解为并行。协程是一个用户层面的概念，不同协程的模型实现可能是单线程的，也可能是多线程的

进程拥有独立的堆和栈，既不共享堆，也不共享栈。进程由操作系统调度。(全局变量保存在堆中，局部变量及函数保存在栈中)

线程拥有独立的栈和共享的堆。共享堆，不共享栈，线程由操作系统调度。(标准线程)

协程和线程一个共享堆，不共享栈，协程由程序员在协程里显式调度

一个应用程序一般对应一个进程，一个进程一般有一个主线程，还有若干个辅助线程，线程之间是平行运行的，在线程里面可以开协程，让程序在特定时间运行

协程和线程的区别：协程避免了无意义的调度，由此可以提高性能，但也因此，程序员必须自己承担调度的责任，同时协程也失去了标准线程使用多CPU的能力