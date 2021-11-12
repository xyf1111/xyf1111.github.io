# Go垃圾收集器


## 设计原理

今天的编程语言通常会使用以手动和自动两种方式管理内存，C、C++以及Rust等编程语言使用手动方式管理内存，工程师需要主动申请或者释放内存；而Python、Ruby、Java和Go等语言使用自动的内存管理系统，一般都是垃圾回收机制，不过Objective-C却选择了自动引用计数，虽然引用计数也是自动的内存管理机制。

相信很多人对垃圾收集器的印象都是暂停程序(Stop the world, STW)，随着用户程序申请越来越多的内存，系统中的垃圾也逐渐增多；当程序中的内存达到一定阈值时，整个应用程序就会全部暂停，垃圾收集器会扫描已经分配的所有对象并回收不再使用的内存空间，当这个过程结束后，用户程序才可以继续执行，Go语言在早期也使用这种策略实现垃圾收集，但是今天的实现已经复杂了很多。

!["内存管理的组件"](/images/memory-management-component.png "内存管理的组件")

在上图中，用户程序(Mutator)会通过内存分配器(Allocator)在堆上申请内存，而垃圾收集器(Collector)负责回收堆上的内存空间，内存分配器和垃圾收集器共同管理着程序中的堆内存空间。

### 标记清除

标记清除(Mark-Sweep)算法是最常见的垃圾收集算法，标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记(Mark)和清除(Sweep)两个阶段

- 标记阶段(从根对象出发查找并标记堆中所有存活的对象)
- 清除阶段(遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表)

如下图所示，内存空间中包含多个对象，我们从根对象出发依次遍历对象的子对象并将从根节点可达的对象都标记为存活状态，即A、C和D三个对象，剩余B、E和F三个对象因为从根节点不可达，所以会被当成垃圾。

!["标记清除的标记阶段"](/images/mark-sweep-mark-phase.png "标记清除的标记阶段")

标记阶段结束后会进入清除阶段，在该阶段中收集器会依次遍历堆中的所有对象，释放其中没有被标记的B、E和F三个对象并将新的空闲内存空间以链表的结构串联起来，方便内存分配器的使用

!["标记清除的清除阶段"](/images/mark-sweep-mark-phase2.png "标记清除的清除阶段")

这里介绍的是最传统的标记清除算法，垃圾收集器从垃圾收集的根对象出发，递归遍历这些对象指向的子对象并将所有可达的对象标记成存活；标记阶段结束后，垃圾收集器会依次遍历堆中的对象并清除其中的垃圾，整个过程中需要标记对象的存活状态，用户程序在垃圾收集的过程中也不能执行，我们需要用到更复杂的机制来解决STW的问题。

### 三色抽象

为了解决原始标记清除算法带来的长时间STW，多数现代的追踪式垃圾收集器都会实现三色标记算法的变种以缩短STW的时间。三色标记算法将程序中的对象分成白色，黑色和灰色三类

- 白色对象：潜在的垃圾，其内存可能会被垃圾收集器回收
- 黑色对象：活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象
- 灰色对象：活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象

!["三色的对象"](/images/tri-color-objects.png "三色的对象")

在垃圾收集器开始工作时，程序中不存在任何的黑色对象，垃圾收集的根对象会被标记为灰色，垃圾收集器只会从灰色对象集合中取出对象开始扫描，当灰色集合中不再存在对象时，标记阶段就会结束。

!["三色标记垃圾收集器的执行过程"](/images/tri-color-mark-sweep.png "三色标记垃圾收集器的执行过程")

三色标记垃圾收集器的工作原理很简单，我们可以将其归纳成一下几个步骤：

1. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色。
2. 将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收。
3. 重复上述两个步骤直到对象图中不存在灰色对象

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色垃圾，下面是使用三色标记垃圾收集器执行标记后的堆内存，堆中只有对象D待回收垃圾

!["三色标记后的堆"](/images/tri-color-mark-sweep-after-mark-phase.png "三色标记后的堆")

因为用户程序可能在标记执行的过程中修改对象的指针，所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要STW，在如下所示的三色标记过程中，用户程序建立了从A对象到D对象的引用，但是因为程序中已经不存在灰色对象了，所以D对象会被垃圾收集器错误的回收

!["三色标记与用户程序"](/images/tri-color-mark-sweep-and-mutator.png "三色标记与用户程序")

本来不应该被回收的对象却被回收了，这在内存管理中是非常严重的错误，我们将这种错误称为悬挂指针，即指针没有指向特定类型的合法对象，影响了内存的安全性，想要并发或者增量的标记对象还是需要使用屏障技术

### 屏障技术

内存屏障技术是一种屏障指令，它可以让CPU或者编译器在执行内存相关操作时遵循特定的约束，目前多数的现代处理器都会乱序执行指令以最大化性能，但是该技术能够保证内存操作的顺序性，在内存屏障执行前的操作一定会先于内存屏障后执行的操作。

想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性(Tri-color-invariant)中的一种：

- 强三色不变性：黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象
- 弱三色不变性：黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径

!["三色不变性"](/images/strong-weak-tricolor-invariant.png "三色不变性")

上图分别展示了遵循三色不变性和弱三色不变性的堆内存，遵循上述两个不变性的任意一个，我们都能保证垃圾收集算法的正确性，而屏障技术就是在并发或者增量标记过程中保证三色不变性的重要技术。

垃圾收集器中的屏障技术更像是一个钩子方法，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，根据操作类型的不同，我们可以将它们分成读屏障(Read barrier)和写屏障(Write barrier)两种，因为读屏障需要在读操作中加入代码片段，对用户程序的性能影响很大，所以编程语言往往会采用写屏障保证三色不变性。

我们在这里想要介绍的是Go语言中使用的两种写屏障技术，分别是Dijkstra提出的插入写屏障和Yuasa提出的删除写屏障，这里会分析它们如何保证三色不变性和垃圾收集器的正确性。

**插入写屏障**

Dijkstra在1978年提出了插入写屏障，通过如下所示的写屏障，用户程序和垃圾收集器可以在交替工作下保证程序执行的正确性

```code
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```

上述插入写屏障的伪代码非常好理解，每当执行类似`*slot=ptr`的表达式时，我们会执行上述写屏障通过`shade`函数尝试改变指针的颜色。如果ptr指针是白色的，那么该函数会将该对象设置为灰色，其他情况则不变。

!["Dijkstra插入写屏障"](/images/dijkstra-insert-write-barrier.png "Dijkstra插入写屏障")

假设我们在应用程序中使用Dijkstra提出的插入写屏障，在一个垃圾收集器和用户程序交替运行的场景中会出现上图所示的标记过程

1. 垃圾收集器将根对象指向A对象标记成黑色并将A对象指向的对象B标记为灰色
2. 用户程序修改A对象的指针，将原本指向B对象的指针指向C对象，这时触发写屏障将C对象标记成灰色
3. 垃圾收集器一次遍历程序中的其他灰色对象，分别将它们标记成黑色

Dijkstra的插入写屏障是一种相对保守的屏障技术，它会将`有存活可能的对象都标记成灰色`以满足三色不变性。在如上所示的垃圾收集过程中，实际上不再存活的B对象最后没有被回收；而如果我们在第二和第三步之间将指向C对象的指针改为指向B，垃圾收集器仍然认为C对象是存活的，这些被错误标记的垃圾对象只有在下一次循环才会被回收。

插入式的Dijkstra写屏障虽然实现非常简单并且能保证强三色不变性，但是它也有明显的缺点。因为栈上的对象在垃圾收集中也会被认为是根对象，所以为了保证内存的安全，Dijkstra必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之间做出权衡。

**删除写屏障**

Yuasa在1990年的论文`Real-time garbage collection on general-purpose machines`中提出了删除写屏障，因为一旦该写屏障开始工作，它会保证开启写屏障时堆上所有对象的可达，所以也被称作为快照垃圾收集(Snapshot GC)

```
This guarantees that no objects will become unreachable to the garbage collector traversal all objects which are live at the beginning of garbage collection will be reached even if the pointers to them are overwritten.
```

该算法会使用如下所示的写屏障保证增量或者并发执行垃圾收集时程序的正确性

```code
writePointer(slot, ptr)
    shade(*slot)
    *slot = ptr
```

上述代码会在老对象的引用被删除时，将白色的老对象涂成灰色的，这样删除写屏障可以保证弱三色不变性，老对象引用的下游对象一定可以被灰色对象引用。

!["Yuasa删除写屏障"](/images/yuasa-delete-write-barrier.png "Yuasa删除写屏障")

假设我们在应用程序中使用Yuasa提出的删除写屏障，在一个垃圾收集器和用户程序交替运行的场景中会出现上图所示的标记过程：

1. 垃圾收集器将根对象指向A对象标记成黑色并将A对象指向的对象B标记成灰色
2. 用户程序将A对象原本指向B的指针指向C，触发删除写屏障，但是因为B对象已经是灰色的，所以不做改变
3. 用户程序将B对象原本指向C的指针删除，触发删除写屏障，白色的C对象被涂成灰色
4. 垃圾收集器依次遍历程序中的其他灰色对象，将它们分别标记成黑色

上述过程中的第三步触发了Yuasa删除写屏障的着色，因为用户程序删除了B指向C对象的指针，所以C和D两个对象会分别违反强三色不变性和弱三色不变性

- 强三色不变性：黑色的A对象直接指向白色的C对象
- 弱三色不变性：垃圾收集器无法从某个灰色对象出发，经过几个连续的白色对象访问白色的C和D两个对象

Yuasa删除写屏障通过对C对象的着色，保证了C对象和下游的D对象能够在这一次垃圾收集的循环中存活，避免发生悬挂指针以保证用户程序的正确性

### 增量和并发

传统的垃圾收集算法会在垃圾收集的执行期间暂停应用程序，一旦触发垃圾收集，垃圾收集器会抢占CPU的使用权占据大量的计算资源以完成标记和清楚工作，然而 很多追求实时的应用程序无法接受长时间的STW

!["垃圾收集与暂停程序"](/images/stop-the-world-collector.png "垃圾收集与暂停程序")

远古时代的计算资源还没有今天这么丰富，今天的计算机往往都是多核的处理器，垃圾收集器一旦开始执行就会浪费大量的计算资源，为了减少应用程序暂停的最长时间和垃圾收集的总暂停时间，我们便会使用下面的策略优化现代的垃圾处理器

- 增量垃圾处理：增量的标记和清除垃圾，降低应用程序暂停的最长时间
- 并发垃圾收集：利用多核的计算资源，在用户程序执行时并发标记和清除垃圾

因为增量和并发这两种方式都可以与用户程序交替进行，所以我们需要`使用屏障技术`保证垃圾收集器的正确性；与此同时，应用程序也不能等到内存溢出时触发垃圾收集，因为当内存不足时，应用程序已经无法分配内存，这与直接暂停程序没有什么区别，增量和并发的垃圾收集需要提前触发并在内存不足前完成整个循环，避免程序的长时间暂停。

**增量收集器**

增量式(Incremental)的垃圾收集是减少程序最长暂停时间的一种方案，它可以将原本时间较长的暂停时间切分成多个更小的GC时间片，虽然从垃圾收集开始到结束的时间更长了，但是这也减少了应用程序暂停的最大时间

!["增量垃圾收集器"](/images/incremental-collector.png "增量垃圾收集器")

需要注意的是，增量式的垃圾收集需要与三色标记法一起使用，为了保证垃圾收集的正确性，我们需要在垃圾收集开始前打开写屏障，这样用户程序修改内存都会先经过写屏障的处理，保证了堆内存中对象的强三色不变性或弱三色不变性。虽然增量式垃圾收集器能够减少最大的程序暂停时间，但是增量式收集也会增加一次GC循环的总时间，在垃圾收集期间，因为写屏障的影响用户程序也要承担额外的计算开销，所以增量式的垃圾收集也不是只带来好处的，但是总体来说还是利大于弊的。

**并发收集器**

并发(Concurrent)的垃圾收集不仅能够减少程序的最长暂停时间，还能减少整个垃圾收集阶段的时间，通过开启读写屏障、利用多核优势与用户程序并行执行，并发垃圾收集器确实能够减少垃圾收集对应用程序的影响：

!["并发垃圾收集器"](/images/concurrent-collector.png "并发垃圾收集器")

虽然并发收集器能够与用户程序一起运行，但是并不是所有阶段都可以与用户程序一起运行，部分阶段还是需要暂停用户程序的，不过这与传统的算法相比，并发的垃圾收集器可以将能够并发执行的工作尽量并发执行；当然，因为读写屏障的引入，并发的垃圾收集器一定也会带来额外开销，不仅会增加垃圾收集的总时间，还会影响用户程序，这是我们在设计垃圾回收策略时必须要注意的。

## 演进过程

Go语言的垃圾收集器从诞生的第一天起就一直在演进，除了少数的几个版本没有大更新之外，几乎每次发布的小版本都会提升垃圾收集的性能，而与性能一同提升的还有垃圾收集代码的复杂度，本节将从Go语言的v1.0版本开始分析垃圾收集器的演进过程。

1. v1.0：完全串行的标记和清除过程，需要暂停整个程序
2. v1.1：在多核主机上并行执行垃圾收集的标记和清除阶段
3. v1.3：运行时基于`只有指针类型的值包含指针`的假设增加了对栈内存的精准扫描支持，实现了真正精准的垃圾收集
   - 将`unsafe.Pointer` 类型转化成整数类型的值认定为不合法的，可能会造成悬挂指针等严重问题
4. v1.5：实现了基于`三色标记清扫的并发`垃圾收集器
   - 大幅度降低垃圾收集的延迟从几百ms降低至10ms以下
   - 计算垃圾收集启动的合适时间并通过并发加速垃圾收集的过程
5. v1.6：实现了`去中心化`的垃圾收集协调器
   - 基于显式的状态机使得任意Goroutine都能触发垃圾收集的状态迁移
   - 使用密集的位图代替空闲链表表示的堆内存，降低清除阶段的CPU占用
6. v1.7：通过`并行栈收缩`将垃圾收集的时间缩短至2ms以内
7. v1.8：使用`混合写屏障`将垃圾收集的时间缩短至0.5ms以内
8. v1.9：彻底移除暂停程序的重新扫描栈的过程
9. v1.10：更新了垃圾收集调频器(Pacer)的实现，分离软硬堆大小的目标
10. v1.12：使用`新的标记终止算法`简化垃圾收集器的几个阶段
11. v1.13：通过新的Scavenger解决瞬间内存占用过高的应用程序向操作系统归还内存的问题
12. v1.14：使用全新的页分配器`优化内存分配的速度`

我们从Go语言垃圾收集器的演进能够看到该组件的实现和算法变得越来越复杂，最开始的垃圾收集器还是不精准的单线程STW收集器，但是最新版本的垃圾收集器却支持并发垃圾收集、去中心化协调等特性，我们在这里将介绍与最新版垃圾收集器相关的组件和特性。

### 并发垃圾收集

Go语言在v1.5中引入了并发的垃圾收集器，该垃圾收集器使用了我们上面提到的三色抽象和写屏障技术保证垃圾收集器执行的正确性，如何实现并发的垃圾收集器在这里就不展开介绍了，我们来了解一些并发垃圾收集器的工作流程。

首先，并发垃圾收集器必须在合适的时间点触发垃圾收集循环，假设我们的Go语言运行在一台4核的物理机上，那么在垃圾收集开始后，收集器会占用25%计算资源在后台来扫描并标记内存中的对象。

!["语言的并发收集"](/images/golang-concurrent-collector.png "语言的并发收集")

Go语言的并发垃圾收集器会在扫描对象之前暂停程序做一些标记对象的准备工作，其中包括启动后台标记的垃圾收集器以及开启写屏障，如果在后台执行的垃圾收集器不够快，应用程序申请内存的速度超过预期，运行时会让申请内存的应用程序辅助完成垃圾收集的扫描阶段，在标记和标记终止阶段结束之后就会进入异步的清理阶段，将不用的内存增量回收。

v1.5版本实现的并发垃圾收集策略由专门的Goroutine负责在处理器间同步和协调垃圾收集的状态。当其他的Goroutine发现需要触发垃圾收集时，它们需要将该信息通知给负责修改状态的主Goroutine，然而这个通知的过程会带来一定的延迟，这个延迟的时间窗口是不可控的，用户程序会在这段时间继续分配内存。

v1.6版本引入了去中心化的垃圾收集协调机制，将垃圾收集器变成一个显式的状态机，任意的Goroutine都可以调用方法触发状态的迁移，常见的状态迁移方法包括以下几个：

- `runtime.gcStart`：从`_GCoff`阶段转换至`_GCmark`阶段，进入并发标记阶段并打开写屏障
- `runtime.gcMarkDone`：如果所有可达对象都已经完成扫描调用`runtime.gcMarkTermination`
- `runtime.gcMarkTermination`：从`_GCmark`转换`_GCmarktermination`阶段，进入标记终止阶段并在完成后进入`_GCoff`

上述的三个方法是在与`runtime: replace GC coordinator with state machine`问题相关的提交中引入的，它们移除了过去中心化的状态迁移过程。

### 回收堆目标

STW的垃圾收集器虽然需要暂停应用程序，但是它能够有效地控制堆内存的大小，Go语言运行时默认配置会在堆内存达到上一次垃圾收集的两倍时，触发新一轮的垃圾收集，这个行为可以通过环境变量`GOGC`调整，在默认情况下它的值为100，即增长100%的堆内存才会触发GC。

!["STW垃圾收集器的垃圾收集时间"](/images/stop-the-world-garbage-collector-heap.png "STW垃圾收集器的垃圾收集时间")

因为并发垃圾收集器会和程序一起执行，所以它无法准确的控制内存的大小，并发收集器需要在达到目标前触发垃圾收集，这样才能够保证内存的大小可控，并发收集器需要尽可能保证垃圾收集结束时的堆内存与用户配置的`GOGC`一致。

!["并发收集器的堆内存"](/images/concurrent-garbage-collector-heap.png "并发收集器的堆内存")

Go语言v1.5引入并发垃圾收集器的同时使用垃圾收集调步(Pacing)算法计算触及的垃圾收集的最佳时间，确保触发的时间既不会浪费计算资源，也不会超出预期的堆大小。如上图所示，其中黑色的部分是上一次垃圾收集后标记的堆大小，绿色部分是上次垃圾收集结束后新分配的内存，因为我们使用并发垃圾收集，所以黄色部分就是在垃圾收集期间分配的内存，最后的红色部分是垃圾收集结束时与目标的差值，我们希望尽可能减少红色部分的内存，降低垃圾收集带来的额外开销以及程序的暂停时间。

垃圾收集调步算法是跟随v1.5一同引入的，该算法的目标是优化堆的增长速度和垃圾收集器的CPU利用率，而在v1.10版本中又对该算法进行了优化，将原有的目的堆大小拆分成软硬两个目标，因为调整垃圾收集的执行频率涉及较为复杂的公式，对理解垃圾收集原理帮助较为有限。

### 混合写屏障

在Go语言v1.7版本之前，运行时会使用Dijkstra插入写屏障保证强三色不变性，但是运行时并没有在所有的垃圾收集根对象上开启插入写屏障。因为应用程序可能包含成百上千的Goroutine，而垃圾收集的根对象一般包括全局变量和栈对象，如果运行时需要在几百个Goroutine的栈上都开启写屏障，会带来巨大的额外开销，所以Go团队在实现上选择了在标记阶段完成时`暂停程序、将所有栈对象标记为灰色并重新扫描`，在活跃Goroutine非常多的程序中，重新扫描的过程需要占用10~100ms的时间。

Go语言在v1.8组合Dijkstra插入写屏障和Yuasa删除写屏障构成了如下所示的混合写屏障，该写屏障会`将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色`

```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot=ptr
```

为了移除栈的重新扫描过程，除了引入混合写屏障之外，在垃圾收集的标记阶段，我们还需要`将创建的所有新对象都标记成黑色`，防止新分配的栈内存和堆内存中的对象被错误的回收，因为栈内存在标记阶段最终都会变为黑色，所以不需要再重新扫描栈空间。

## 实现原理

在介绍垃圾收集器的演进过程之前，我们需要初步了解最新垃圾收集器的执行周期，这对我们了解其全局的设计会有比较大的帮助。Go语言的垃圾收集可以分成清除终止、标记、标记终止和清除四个不同阶段，它们分别完成了不同的工作。

!["垃圾收集的多个阶段"](/images/garbage-collector-phaes.png "垃圾收集的多个阶段")

1. 清理终止阶段
   1. 暂停程序，所有的处理器在这时会进入安全点(Safe Point)
   2. 如果当前垃圾收集循环是强制触发的，我们还需要处理还未被清理的内存管理单元
2. 标记阶段
   1. 将状态切换至`_GCmark`、开启写屏障、用户程序协助(Mutator Assists)并将根对象入队
   2. 恢复执行程序，标记进程和用于协助的用户程序会开始并发标记内存中的对象，写屏障会将被覆盖的指针和新指针都标记成灰色，而所有新创建的对象都会被直接标记成黑色
   3. 开始扫描根对象，包括所有Goroutine的栈、全局对象以及不在堆中的运行时数据结构，扫描Goroutine栈期间会暂停当前处理器
   4. 依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色
   5. 使用分布式的终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段
3. 标记终止阶段
   1. 暂停程序、将状态切换至`_GCmarktermination`并关闭辅助标记的用户程序
   2. 清理处理器上的线程缓存
4. 清理阶段
   1. 将状态切换至`_GCoff`开始清理阶段，初始化清理状态并关闭写屏障
   2. 恢复用户程序，所有新创建的对象会标记成白色
   3. 后台并发清理所有的内存管理单元，当Goroutine申请新的内存管理单元时就会触发清理

运行时虽然只会使用`_GCoff`、`_GCmark`和`_GCmarktermination`三个状态表示垃圾收集的全部阶段，但是在实现上却复杂很多。本节将按照垃圾收集的不同阶段详细分析其实现原理

### 全局变量

在垃圾收集中有一些比较重要的全局变量，在分析其过程之前，会逐一介绍这些重要的变量，这些变量在垃圾收集的各个阶段会反复出现，所以理解他们的功能是非常重要的，先介绍一些简单的变量：

- `runtime.gcphase` 是垃圾收集器当前处于的阶段，可能处于`_GCoff`、`_GCmark`和`_GCmarktermination`，Goroutine在读取或者修改该阶段时需要保证原子性
- `runtime.gcBlackenEnabled` 是一个布尔值，当垃圾收集处于标记阶段时，该变量会被置为1，在这里辅助垃圾收集的用户程序和后台标记的任务可以将对象涂黑
- `runtime.gcController` 实现了垃圾收集的调步算法，它能够决定出发并行垃圾收集的时间和待处理的工作
- `runtime.gcpercent` 是触发垃圾收集的内存增长百分比，默认情况下为100，即堆内存相比上次垃圾收集增长100%时应该触发GC，并行的垃圾收集器会在到达该目标前完成垃圾收集
- `runtime.writerBarrier` 是一个包含写屏障状态的结构体，其中的`enable`字段表示写屏障的开启与关闭
- `runtime.worldsema` 是全局的信号量，获取该信号量的线程有权利暂停当前应用程序

除了上述全局的变量之外，我们在这里还需要简单了解一下`runtime.work`变量：

```go
var work struct {
    full    lfstack
    empty   lfstack
    pad0    cpu.CacheLinePad

    wbufSpans struct {
        lock    mutex
        free    mSpanList
        busy    mSpanList
    }
    ...
    nproc   uint32
    tstart  int64
    nwait   uint32
    ndone   uint32
    ...
    mode    gcMode
    cycles  uint32
    ...
    stwprocs, maxprocs  int32
    ...
}
```

该结构体中包含大量垃圾收集的相关字段，例如：表示完成的垃圾收集循环的次数、当前循环时间和CPU的利用率、垃圾收集的模式等等，我们会在后面的小节中见到该结构体中的更多字段

### 触发时机

运行时会通过如下所示的`runtime.gcTrigger.test`方法决定是否需要触发垃圾收集，当满足触发垃圾收集的基本条件时：允许垃圾收集、程序没有崩溃并且没有处于垃圾收集循环，该方法会根据三种不同方式触发进行不同的检查：

```go
func (t gcTrigger) test() bool {
    if !memstats.enablegc || panicking != 0 || gcphase != _GCoff {
        return false
    }
    switch t.kind {
    case gcTriggerHeap:
        return memstats.heap_live >= memstats.gc_trigger
    case gcTriggerTime:
        if gcpercent < 0 {
            return false
        }
        last := int64(atomic.Load64(&memstats.last_gc_nanotime))
        return lastgc != 0 && t.now-lastgc > forcegcperiod
    case gcTriggerCycle:
        return int32(t.n-work.cycles) > 0
    }
    return true
}
```

1. `gcTriggerHeap` 堆内存的分配达到控制器计算的触发堆大小
2. `gcTriggerTime` 如果一定时间内没有触发，就会触发新的循环，该触发条件有`runtime.forcegcperiod`变量控制，默认为2分钟
3. `gcTriggerCycle` 如果当前没有开启垃圾收集、则触发新的循环

用于开启垃圾收集的方法`runtime.goStart`会接收一个`runtime.gcTrigger`类型的谓词，所有出现`runtime.gcTrigger`结构体的位置都是触发垃圾收集的代码

- `runtime.sysmon`和`runtime.forcegchelper` 后台运行定时检查和垃圾收集
- `runtime.GC` 用户程序手动触发垃圾收集
- `runtime.mallocgc` 申请内存时根据堆大小触发垃圾收集

!["垃圾收集的触发"](/images/garbage-collector-trigger.png "垃圾收集的触发")

除了使用后台运行的系统监控器和强制垃圾收集助手触发垃圾收集之外，另外两个办法会从任意处理器上触发垃圾收集，这种不需要中心组件协调的方式是在v1.6版本中引入的，接下来将展开介绍这三种不同的触发时机

**后台触发**

运行时会在应用程序启动时在后台开启一个用于强制触发垃圾收集的Goroutine，该Goroutine的职责非常简单(调用`runtime.gcStart`尝试启动新一轮的垃圾收集)

```go
func init() {
    go forcegchelper()
}

func forcegchelper() {
    forcegc.g = getg()
    for {
        lock(&forcegc.lock)
        atomic.Store(&forcegc.idle, 1)
        goparkunlock(&forcegc.lock, waitReasonForceGGIdle, traceEvGoBlock, 1)
        gcStart(gcTrigger{
            kind:   gcTriggerTime,
            now:    nanotime()
        })
    }
}
```

为了减少对计算资源的占用，该Goroutine会在循环中调用`runtime.goparkunlock`主动陷入休眠等待其他Goroutine的唤醒，`runtime.forcegchelper`在大多数时间都是陷入休眠的，但是它会被系统监控器`runtime.sysmon`在满足垃圾收集条件时唤醒：

```go
func sysmon() {
    ...
    for {
        ...
        if t := (gcTrigger{
            kind:   gcTriggerTime,
            now:    now,
        }); t.test() && atomic.Load(&forcegc.idle) != 0 {
            lock(&forcegc.lock)
            forcegc.idle = 0
            var list gList
            list.push(forcegc.g)
            injectglist(&list)
            unlock(&forcegc.lock)
        }
    }
}
```

系统监控在每个循环中都会主动构建一个`runtime.gcTrigger`并检查垃圾收集的触发条件是否满足，如果满足条件，系统监控会将`runtime.forcegc`状态中持有的Goroutine加入全局队列等待调度器的调度

**手动触发**

用户程序会通过`runtime.GC`函数在程序运行期间主动通知运行时执行，该方法在调用时会阻塞调用方直到当前垃圾收集循环完成，在垃圾收集期间也可能会通过STW暂停整个程序

```go
func GC() {
    n := atomic.Load(&work.cycles)
    gcWaitOnMark(n)
    gcStart(gcTrigger{
        kind:   gcTriggerCycle,
        n:      n+1,
    })
    gcWaitOnMark(n+1)

    for atomic.Load(&work.cycles) == n+1 && sweepone() != ^uintptr(0) {
        sweep.nbgsweep++
        Gosched()
    }

    for atomic.Load(&work.cycles) == n+1 && atomic.Load(&mheap_.sweepers) != 0 {
        Gosched()
    }

    mp := acquirem()
    cycle := atomic.Load(&work.cycles)
    if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
        mProf_PostSweep()
    }
    releasem(mp)
}
```

1. 在正式开始垃圾收集前，运行时需要通过`runtime.gcWaitOnMark`等待上一个循环的标记终止、标记和清除终止阶段完成
2. 调用`runtime.gcStart`触发新一轮的垃圾收集并通过`runtime.gcWaitOnMark`等待该轮垃圾收集的标记终止阶段正常结束
3. 持续调用`runtime.sweepone`清理全部待处理的内存管理单元并等待所有的清理工作完成，等待期间会调用`runtime.Gosched`让出处理器
4. 完成本轮垃圾收集的清理工作后，通过`runtime.mProf_PostSweep`将该阶段的堆内存状态快照发布出来，我们可以获取这时的内存状态

手动触发垃圾收集的过程并不是特别常见，一般只会在运行时的测试代码中才会出现，不过如果我们认为触发主动垃圾收集是有必要的，也可以直接调用该方法，但是作者不认为这是一种推荐的做法

**申请内存**

最后一个可能会触发垃圾收集的就是`runtime.mallocgc`，Go运行时会将堆上的对象按大小分成微对象、小对象和大对象三类，这三类对象的创建都可能会触发细心地垃圾收集循环

```go
func mallocgc(size uintptr, type *_type, needzero bool) unsafe.Pointer {
    shouldhelpgc := false
    ...
    if size <= maxSmallSize {
        if noscan && size < maxTinySize {
            ...
            v := nextFreeFast(span)
            if v == 0 {
                v, _, shouldhelpgc = c.nextFree(tinySpanClass)
            }
            ...
        } else {
            ...
            v := nextFreeFast(span)
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(spc)
            }
            ...
        }
    } else {
        shouldhelpgc = true
        ...
    }
    ...
    if shouldhelpgc {
        if t := (gcTrigger{
            kind:   gcTriggerHeap
        }); t.test() {
            gcStart(t)
        }
    }
    return x
}
```

1. 当前线程的内存管理单元中不存在空闲空间时，创建微对象和小对象需要调用`runtime.mcache.nextFree`从中心缓存或者页堆中获取新的管理单元，在这时就可能触发垃圾收集
2. 当用户程序申请分配32KB以上的大对象时，一定会构建`runtime.gcTrigger`结构体尝试触发垃圾收集

通过堆内存触发垃圾收集需要比较`runtime.mstats`中的两个字段：表示垃圾收集中存活对象字节数的`heap_live`和表示触发标记的堆内存大小的`gc_trigger`；当内存中存活的对象字节数大于触发垃圾收集的堆大小时，新一轮的垃圾收集就会开始。这里将分别介绍这两个值的计算过程：
1. `heap_live`: 为了减少锁竞争，运行时只会在中心缓存分配或者释放内存管理单元以及在堆上分配大对象时才会更新
2. `gc_trigger`：在标记终止阶段调用`runtime.gcSetTriggerRatio`更新触发下一次垃圾收集的堆大小

`runtime.gcController`会在每个循环结束后计算触发比例并通过`runtime.gcSetTriggerRatio`设置`gc_trigger`，它能够决定触发垃圾收集的时间以及用户程序和后台处理的标记任务的多少，利用反馈控制的算法根据堆的增长情况和垃圾收集CPU利用率确定触发垃圾收集的时机

可以在`runtime.gcControllerState.endCycle`中找到v1.5提出的垃圾收集调步算法，并在`runtime.gcControllerState.revise`中找到v1.10引入的软硬堆目标分离算法

### 垃圾收集启动

垃圾收集在启动过程一定会调用`runtime.gcStart`，虽然该函数的实现比较复杂，但是它的主要职责是修改全局的垃圾收集状态到`_GCmark`并做一些准备工作，下面会分几个阶段介绍该函数的实现

1. 两次调用`runtime.gcTrigger.test`检查是否满足垃圾收集条件
2. 暂停程序、在后台启动用于处理标记任务的工作Goroutine、确定所有内存管理单元都被清理以及其他标记阶段开始前的准备工作
3. 进入标记阶段、准备后台的标记工作、根对象的标记工作以及微对象、恢复用户程序，进入并发扫描和标记阶段

验证垃圾收集条件的同时，该方法还会在循环中不断调用`runtime.sweepone`清理已经被标记的内存单元，完成上一个垃圾收集循环的收尾工作

```go
func gcStart(trigger gcTrigger) {
    for trigger.test() && sweepone() != ^uintptr(0) {
        sweep.nbgsweep++
    }

    semacquire(&work.startSema)
    if !trigger.test() {
        semrelease(&work.startSema)
        return
    }
    ...
}
```

在验证了垃圾收集的条件并完成了收尾工作后，该方法会通过`semacquire`获取全局的`worldsema`信号量、调用`runtime.gcBgMarkStartWorkers`启动后台标记任务、在系统栈中调用`runtime.stopTheWorldWithSema`暂停程序并调用`runtime.finishsweep_m`保证上一个内存单元的正常回收：

```go
func gcStart(trigger gcTrigger) {
    ...
    semacquire(&worldsema)
    gcBgMarkStartWorkers()
    work.stwprocs, work.maxprocs = gomaxprocs, gomaxprocs
    ...
    
    systemstack(stopTheWorldWithSema)
    systemstack(func() {
        finishsweep_m()
    })

    work.cycles++
    gcController.startCycle()
    ...
}
```

除此之外，上述过程还会修改全局变量`runtime.work`持有的状态，包括垃圾收集需要的Goroutine数量以及已完成的循环数

在完成全部的准备工作后，该方法就进入了执行的最后阶段。在该阶段中，我们会修改全局的垃圾收集状态到`_GCmark`并依次执行下面的步骤

1. 调用`runtime.gcBgMarkPrepare`初始化后台扫描需要的状态
2. 调用`runtime.gcMarkRootPrepare`扫描栈上、全局变量等根对象并将它们加入队列
3. 设置全局变量`runtime.gcBlackenEnabled`，用户程序和标记任务可以将对象涂黑
4. 调用`runtime.startTheWorldWithSema`启动程序，后台任务也会开始标记堆中的对象

```go
func gcStart(trigger gcTrigger) {
    ...
    setGCPhase(_GCmark)

    gcBgMarkPrepare()
    gcMarkRootPrepare()

    atomic.Store(&gcBlackenEnabled, 1)
    systemstack(func() {
        now = startTheWorldWithSema(trace.enabled)
        work.pauseNS += now - work.pauseStart
        work.tMark = now
    })
    semrelease(&work.startSema)
}
```

在分析垃圾收集的启动过程中，我们省略了几个关键的过程，其中包括暂停和恢复应用程序和后台任务的启动，下面将详细分析这几个过程的实现原理

**暂停与恢复程序**

`runtime.stopTheWorldWithSema`和`runtime.startTheWorldWithSema`是一对用于暂停和恢复程序的核心函数，它们有着完全相反的功能，但是程序的暂停会比恢复要复杂一些，看下前者的实现原理

```go
func stopTheWorldWithSema() {
    _g_ := getg()
    sched.stopwait = gomaxprocs
    atomic.Store(&sched.gcwaiting, 1)
	preemptall()
	_g_.m.p.ptr().status = _Pgcstop
	sched.stopwait--
	for _, p := range allp {
		s := p.status
		if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) {
			p.syscalltick++
			sched.stopwait--
		}
	}
	for {
		p := pidleget()
		if p == nil {
			break
		}
		p.status = _Pgcstop
		sched.stopwait--
	}
	wait := sched.stopwait > 0
	if wait {
		for {
			if notetsleep(&sched.stopnote, 100*1000) {
				noteclear(&sched.stopnote)
				break
			}
			preemptall()
		}
	}
}
```

暂停程序主要使用了`runtime.preemptall`，该函数会调用前面介绍的`runtime.preemtone`，因为程序中活跃的最大处理数为`gomaxprocs`，所以`runtime.stopTheWorldWithSema`在每次发现停止的处理器时都会对该变量减一，知道所有的处理器都停止运行。该函数会依次停止当前处理器、等待处于系统调用的处理器以及获取并抢占空闲的处理器，处理器的状态在该函数返回时都会被更新至`_Pgcstop`，等待垃圾收集器的重新唤醒

程序恢复过程会使用`runtime.startTheWorldWithSema`，该函数的实现也相对比较简单

1. 调用`runtime.netpoll`从网络轮询器中获取待处理的任务并加入全局队列
2. 调用`runtime.procresize`扩容或者缩容全局的处理器
3. 调用`runtime.notewakeup`或者`runtime.newm`依次唤醒处理器或者为处理器创建新的线程
4. 如果当前待处理的Goroutine数量过多，创建额外的处理器完成任务

```go
func startTheWorldWithSema(emitTraceEvent bool) int64 {
	mp := acquirem()
	if netpollinited() {
		list := netpoll(0)
		injectglist(&list)
	}

	procs := gomaxprocs
	p1 := procresize(procs)
	sched.gcwaiting = 0
	...
	for p1 != nil {
		p := p1
		p1 = p1.link.ptr()
		if p.m != 0 {
			mp := p.m.ptr()
			p.m = 0
			mp.nextp.set(p)
			notewakeup(&mp.park)
		} else {
			newm(nil, p)
		}
	}

	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
	...
}
```

程序的暂停和启动过程都比较简单，暂停程序会使用`runtime.preemptall`抢占所有的处理器，恢复程序会使用`runtime.notewakeup`或者`runtime.newm`唤醒程序中的处理器

**后台标记模式**

在垃圾收集启动期间，运行时会调用`runtime.gcBgMarkStartWorkers`为全局每个处理器创建用于执行后台标记任务的Goroutine，每一个Goroutine都会运行`runtime.gcBgMarkWorker`，所有运行`runtime.gcBgMarkWorker`的Goroutine在启动后都会陷入休眠等待调度器的唤醒：

```go
func gcBgMarkStartWorkers() {
	for gcBgMarkWorkerCount < gomaxprocs {
		go gcBgMarkWorker()

		notetsleepg(&work.bgMarkReady, -1)
		noteclear(&work.bgMarkReady)

		gcBgMarkWorkerCount++
	}
}
```

这些Goroutine与处理器也是一一对应的关系，当垃圾收集处于标记阶段并且当前处理器不需要做任何任务时，`runtime.findrunnable`会在当前处理器上执行该Goroutine辅助并发的对象标记

!["处理器与后台标记任务"](/images/p-and-bg-mark-worker.png "处理器与后台标记任务")

调度器在调度循环`runtime.schedule`中还可以通过垃圾收集控制器的`runtime.gcControllerState.findRunnabledGCWorker`获取并执行用于后台标记的任务。

用于并发扫描对象的工作协程Goroutine总共有三种不同的模式`runtime.gcMarkWorkerMode`，这三种模式的Goroutine在标记对象时使用完全不同的策略，垃圾收集控制器会按照需要执行不同类型的工作协程

- `gcMarkWorkerDedicateMode` 处理器专门负责标记对象，不会被调度器抢占
- `gcMarkWorkerFractionalMode` 当垃圾收集的后台CPU使用率达不到预期时(默认为25%)，启动该类型的工作协程帮助垃圾收集达到理由率的目标，因为它只占用同一个CPU的部分资源，所以可以被调度
- `gcMarkWorkerIdleMode` 当处理器没有可以执行的Goroutine时，它会运行垃圾收集的标记任务直到被抢占

`runtime.gcControllerState.startCycle` 会根据全局处理器的个数以及垃圾收集的CPU利用率计算出上述的`dedicatedMarkWorkersNeeded`和`fractionalUtilizationGoal`以决定不同模式的工作协程的数量

因为后台标记任务的CPU利用率为25%，如果主机是四核或者八核，那么垃圾收集需要一个或者两个专门处理相关任务的Goroutine，不过如果主机是三核或者六核，因为无法被四整除，所以这时需要零个或者一个专门处理垃圾收集的Goroutine，运行时需要占用某个CPU的部分时间，使用`gcMarkWorkerFractionalMode`模式的协程保证CPU的利用率

!["主机核数与垃圾收集任务模式"](/images/cpu-number-and-gc-mark-worker-mode.png "主机核数与垃圾收集任务模式")

垃圾收集控制器会在`runtime.gcControllerState.findRunnabledGCWorker` 方法中设置处理器的`gcMarkWorkerMode`

```go
func (c *gcControllerState) findRunnableGCWorker(_p_ *p) *g {
	...
	if decIfPositive(&c.dedicatedMarkWorkersNeeded) {
		_p_.gcMarkWorkerMode = gcMarkWorkerDedicatedMode
	} else if c.fractionalUtilizationGoal == 0 {
		return nil
	} else {
		delta := nanotime() - gcController.markStartTime
		if delta > 0 && float64(_p_.gcFractionalMarkTime)/float64(delta) > c.fractionalUtilizationGoal {
			return nil
		}
		_p_.gcMarkWorkerMode = gcMarkWorkerFractionalMode
	}

	gp := _p_.gcBgMarkWorker.ptr()
	casgstatus(gp, _Gwaiting, _Grunnable)
	return gp
}
```

上述方法的实现比较清晰，控制器通过`dedicatedMarkWorkersNeeded`决定专门执行标记任务的Goroutine数量并根据执行标记任务的时间和总时间决定是否启动

`gcMarkWorkerFractionalMode`模式的Goroutine；除了这两种控制器要求的工作协程之外，调度器还会在`runtime.findrunnable`中利用空闲的处理器执行垃圾收集以加速该过程

```go
func findrunnable() (gp *g, inheritTime bool) {
	...
stop:
	if gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != 0 && gcMarkWorkAvailable(_p_) {
		_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
		gp := _p_.gcBgMarkWorker.ptr()
		casgstatus(gp, _Gwaiting, _Grunnable)
		return gp, false
	}
	...
}
```

三种不同模式的工作协程会互相协同保证垃圾收集的CPU利用率达到期望的阈值，在到达目标堆大小前完成标记任务

### 并发扫描与标记辅助

`runtime.gcBgMarkWorker`是后台的标记任务执行的函数，该函数的循环中执行了对内存中对象图的扫描和标记，我们分三个部分介绍该函数的实现原理

1. 获取当前处理器以及Goroutine打包成`runtime.gcBgMarkWorkerNode` 类型的结构并主动陷入休眠等待唤醒
2. 根据处理器上的`gcMarkWorkerMode`模式决定扫描任务的策略
3. 所有标记任务都完成后，调用`runtime.gcMarkDone`方法完成标记阶段

首先我们来看后台标记工作的准备工作，运行时在这里创建了`runtime.gcBgMarkWorkerNode`，该结构会预先存储处理器和当前Goroutine，当我们调用`runtime.gopark`触发休眠时，运行时会在系统栈中安全的建立处理器和后台标记任务的绑定关系

```go
func gcBgMarkWorker() {
	gp := getg()

	gp.m.preemptoff = "GC worker init"
	node := new(gcBgMarkWorkerNode)
	gp.m.preemptoff = ""

	node.gp.set(gp)

	node.m.set(acquirem())
	notewakeup(&work.bgMarkReady)

	for {
		gopark(func(g *g, parkp unsafe.Pointer) bool {
			node := (*gcBgMarkWorkerNode)(nodep)
			if mp := node.m.ptr(); mp != nil {
				releasem(mp)
			}

			gcBgMarkWorkerPool.push(&node.node)
			return true
		}, unsafe.Pointer(node), waitReasonGCWorkerIdle, traceEvGoBlock, 0)
	...
}
```

通过`runtime.gopark`陷入休眠的Goroutine不会进入运行队列，它只会等待垃圾收集控制器或者调度器的直接唤醒；在唤醒后，我们会根据处理器`gcMarkWorkerMode`选择不同的标记执行策略，不同的执行策略都会调用`runtime.gcDrain`扫描工作缓冲区`runtime.gcWork`

```go
node.m.set(acquirem())

		atomic.Xadd(&work.nwait, -1)
		systemstack(func() {
			casgstatus(gp, _Grunning, _Gwaiting)
			switch pp.gcMarkWorkerMode {
			case gcMarkWorkerDedicatedMode:
				gcDrain(&_p_.gcw, gcDrainUntilPreempt|gcDrainFlushBgCredit)
				if gp.preempt {
					lock(&sched.lock)
					for {
						gp, _ := runqget(_p_)
						if gp == nil {
							break
						}
						globrunqput(gp)
					}
					unlock(&sched.lock)
				}
				gcDrain(&_p_.gcw, gcDrainFlushBgCredit)
			case gcMarkWorkerFractionalMode:
				gcDrain(&_p_.gcw, gcDrainFractional|gcDrainUntilPreempt|gcDrainFlushBgCredit)
			case gcMarkWorkerIdleMode:
				gcDrain(&_p_.gcw, gcDrainIdle|gcDrainUntilPreempt|gcDrainFlushBgCredit)
			}
			casgstatus(gp, _Gwaiting, _Grunning)
		})
```

需要注意的是，`gcMarkWorkerDedicatedMode`模式的任务是不能被抢占的，为了减少额外开销，第一次调用`runtime.gcDrain`时是允许抢占的，但是 一旦处理器被抢占，当前Goroutine会将处理器上的所有可运行的Goroutine转移至全局队列中，保证垃圾收集占用的CPU资源

当所有的后台工作任务都陷入等待并且名优剩余工作时，我们就任务该轮垃圾收集的标记阶段结束了，这时我们会调用`runtime.gcMarkDone`

```go
incnwait := atomic.Xadd(&work.nwait, +1)
		if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
			releasem(node.m.ptr())
			node.m.set(nil)

			gcMarkDone()
		}
```

`runtime.gcDrain`是用于扫描和标记堆内存中对象的核心方法，除了该方法之外，我们还会介绍工作池、写屏障以及辅助标记的实现原理

**工作池**

在调用`runtime.gcDrain`时，运行时会传入处理器上的`runtime.gcWorker`，这个结构体是垃圾收集器中工作池的抽象，它实现了一个生产者和消费者的模型，我们可以以该结构为起点从整体理解标记工作

!["垃圾收集器工作池"](/images/gc-work-pool.png "垃圾收集器工作池")

写屏障、根对象扫描和栈扫描都会向工作池中添加额外的灰色对象等待处理，而对象的扫描过程会将灰色对象标记成黑色，同时也可能发现新的灰色对象，当工作对垒中不包含灰色对象时，整个扫描过程就会结束

为了减少锁竞争，运行时在每个处理器上会保存独立的带扫描工作，然而这回遇到和调度器一样的问题(不同处理器的资源不平均，导致部分处理器无事可做，调度器引入了工作窃取来解决这个问题，垃圾收集器也使用了差不多的机制平衡不同处理器上的待处理任务)

!["全局任务与本地任务"](/images/global-work-and-local-work.png "全局任务与本地任务")

`runtime.gcWork`为垃圾收集器提供了生产和消费任务的抽象，该结构体持有了两个重要的工作缓冲区`wbug1`和`wbuf2`，这两个缓冲区分别是主缓冲区和备缓冲区

```go
type gcWork struct {
	wbuf1, wbuf2 *workbuf
	...
}

type workbufhdr struct {
	node lfnode // must be first
	nobj int
}

type workbuf struct {
	workbufhdr
	obj [(_WorkbufSize - unsafe.Sizeof(workbufhdr{})) / sys.PtrSize]uintptr
}
```

当我们向该结构体中增加或者删除对象时，它总会先操作主缓冲区，一旦主缓冲区空间不足或者没有对象，会触发主备缓冲区的切换；而当两个缓冲区空间都不足或者都为空时，会从全局的工作缓冲区中插入或者获取对象，该结构体相关方法的实现都非常简单

**扫描对象**

运行时会使用`runtime.gcDrain`扫描工作缓冲区中的灰色对象，它会根据传入`gcDrainFlags`的不同选择不同的策略

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	gp := getg().m.curg
	preemptible := flags&gcDrainUntilPreempt != 0
	flushBgCredit := flags&gcDrainFlushBgCredit != 0
	idle := flags&gcDrainIdle != 0

	initScanWork := gcw.scanWork
	checkWork := int64(1<<63 - 1)
	var check func() bool
	if flags&(gcDrainIdle|gcDrainFractional) != 0 {
		checkWork = initScanWork + drainCheckThreshold
		if idle {
			check = pollWork
		} else if flags&gcDrainFractional != 0 {
			check = pollFractionalWorkerExit
		}
	}
	...
}
```

- `gcDrainUntilPreemt` 当Goroutine的preempt字段被设置成true时返回
- `gcDrainIdle` 调用`runtime.pollWork`，当处理器上包含 其他待执行Goroutine时返回
- `gcDrainFractional` 调用`runtime.pollFractionalWorkerExit`，当CPU的占用率超过`fractionalUtilizationGoal`的20%时返回
- `gcDrainFlushBgCredit` 调用`runtime.gcFlushBgCredit`计算后台完成的标记任务量以减少并发标记期间的辅助垃圾收集的用户程序的工作量

运行时会使用本地变量中的check检查当前是否应该退出标记任务并让出该处理器。当我们做完准备工作后，就可以开始扫描变量中的根对象，这也是标记阶段中需要先被执行的任务

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	...
	if work.markrootNext < work.markrootJobs {
		for !(preemptible && gp.preempt) {
			job := atomic.Xadd(&work.markrootNext, +1) - 1
			if job >= work.markrootJobs {
				break
			}
			markroot(gcw, job)
			if check != nil && check() {
				goto done
			}
		}
	}
	...
}
```

扫描根对象需要使用`runtime.markroot`，该函数会扫描缓存、数据段、存放全局变量和静态变量的BSS段以及Goroutine的栈内存；一旦完成了根对象的扫描，当前Goroutine会开始从本地和全局工作缓存池中获取待执行的任务

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	...
	for !(preemptible && gp.preempt) {
		if work.full == 0 {
			gcw.balance()
		}

		b := gcw.tryGetFast()
		if b == 0 {
			b = gcw.tryGet()
			if b == 0 {
				wbBufFlush(nil, 0)
				b = gcw.tryGet()
			}
		}
		if b == 0 {
			break
		}
		scanobject(b, gcw)

		if gcw.scanWork >= gcCreditSlack {
			atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
			if flushBgCredit {
				gcFlushBgCredit(gcw.scanWork - initScanWork)
				initScanWork = 0
			}
			checkWork -= gcw.scanWork
			gcw.scanWork = 0

			if checkWork <= 0 {
				checkWork += drainCheckThreshold
				if check != nil && check() {
					break
				}
			}
		}
	}
	...
}
```

扫描对象会使用`runtime.scanobject`，该函数会从传入的位置开始扫描，扫描期间会调用`runtime.greyobject`为找到的活跃对象上色。

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	...
done:
	if gcw.scanWork > 0 {
		atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
		if flushBgCredit {
			gcFlushBgCredit(gcw.scanWork - initScanWork)
		}
		gcw.scanWork = 0
	}
}
```

当本轮的扫描因为外部条件变化而中断时，该函数会通过`runtime.gcFlushBgCredit`记录这次扫描的内存字节数用于减少辅助标记的工作量。

内存中对象的扫描和标记过程涉及很多位操作和指针操作，相关代码实现比较复杂，我们在这里就不展开介绍相关的内容了，感兴趣的读者可以将`runtime.gcDrain`作为入口研究三色标记的具体过程。

**写屏障**

写屏障是保证 Go 语言并发标记安全不可或缺的技术，我们需要使用混合写屏障维护对象图的弱三色不变性，然而写屏障的实现需要编译器和运行时的共同协作。在 SSA 中间代码生成阶段，编译器会使用`cmd/compile/internal/ssa.writebarrier`在`Store`、`Move`和`Zero`操作中加入写屏障，生成如下所示的代码：

```go
if writeBarrier.enabled {
  gcWriteBarrier(ptr, val)
} else {
  *ptr = val
}
```

当 Go 语言进入垃圾收集阶段时，全局变量`runtime.writeBarrier`中的`enabled`字段会被置成开启，所有的写操作都会调用 `runtime.gcWriteBarrier`：

```bash
TEXT runtime·gcWriteBarrier(SB),NOSPLIT,$28
	...
	get_tls(BX)
	MOVL	g(BX), BX
	MOVL	g_m(BX), BX
	MOVL	m_p(BX), BX
	MOVL	(p_wbBuf+wbBuf_next)(BX), CX
	LEAL	8(CX), CX
	MOVL	CX, (p_wbBuf+wbBuf_next)(BX)
	CMPL	CX, (p_wbBuf+wbBuf_end)(BX)
	MOVL	AX, -8(CX)	// 记录值
	MOVL	(DI), BX
	MOVL	BX, -4(CX)	// 记录 *slot
	JEQ	flush
ret:
	MOVL	20(SP), CX
	MOVL	24(SP), BX
	MOVL	AX, (DI) // 触发写操作
	RET

flush:
  ...
	CALL	runtime·wbBufFlush(SB)
  ...
	JMP	ret
```

在上述汇编函数中，DI 寄存器是写操作的目的地址，AX 寄存器中存储了被覆盖的值，该函数会覆盖原来的值并通过`runtime.wbBufFlush`通知垃圾收集器将原值和新值加入当前处理器的工作队列，因为该写屏障的实现比较复杂，所以写屏障对程序的性能还是有比较大的影响，之前只需要一条指令完成的工作，现在需要几十条指令。

我们在上面提到过 Dijkstra 和 Yuasa 写屏障组成的混合写屏障在开启后，所有新创建的对象都需要被直接涂成黑色，这里的标记过程是由`runtime.gcmarknewobject`完成的：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
	if gcphase != _GCoff {
		gcmarknewobject(span, uintptr(x), size, scanSize)
	}
	...
}

func gcmarknewobject(span *mspan, obj, size, scanSize uintptr) {
	objIndex := span.objIndex(obj)
	span.markBitsForIndex(objIndex).setMarked()

	arena, pageIdx, pageMask := pageIndexOf(span.base())
	if arena.pageMarks[pageIdx]&pageMask == 0 {
		atomic.Or8(&arena.pageMarks[pageIdx], pageMask)
	}

	gcw := &getg().m.p.ptr().gcw
	gcw.bytesMarked += uint64(size)
	gcw.scanWork += int64(scanSize)
}
```

`runtime.mallocgc`会在垃圾收集开始后调用该函数，获取对象对应的内存单元以及标记位`runtime.markBits`并调用`runtime.markBits.setMarked`直接将新的对象涂成黑色。

**标记辅助**

为了保证用户程序分配内存的速度不会超出后台任务的标记速度，运行时还引入了标记辅助技术，它遵循一条非常简单并且朴实的原则，`分配多少内存就需要完成多少标记任务`。每一个 Goroutine 都持有`gcAssistBytes`字段，这个字段存储了当前 Goroutine 辅助标记的对象字节数。在并发标记阶段期间，当 Goroutine 调用`runtime.mallocgc`分配新对象时，该函数会检查申请内存的 Goroutine 是否处于入不敷出的状态：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
	var assistG *g
	if gcBlackenEnabled != 0 {
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		assistG.gcAssistBytes -= int64(size)

		if assistG.gcAssistBytes < 0 {
			gcAssistAlloc(assistG)
		}
	}
	...
	return x
}
```

申请内存时调用的`runtime.gcAssistAlloc`和扫描内存时调用的`runtime.gcFlushBgCredit`分别负责借债和还债，通过这套债务管理系统，我们能够保证 Goroutine 在正常运行的同时不会为垃圾收集造成太多的压力，保证在达到堆大小目标时完成标记阶段。

!["辅助标记的动态平衡"](/images/gc-mutator-assist.png "辅助标记的动态平衡")

每个 Goroutine 持有的`gcAssistBytes`表示当前协程辅助标记的字节数，全局垃圾收集控制器持有的`bgScanCredit`表示后台协程辅助标记的字节数，当本地 Goroutine 分配了较多对象时，可以使用公用的信用`bgScanCredit`偿还。我们先来分析`runtime.gcAssistAlloc`的实现：

```go
func gcAssistAlloc(gp *g) {
	...
retry:
	debtBytes := -gp.gcAssistBytes
	scanWork := int64(gcController.assistWorkPerByte * float64(debtBytes))
	if scanWork < gcOverAssistWork {
		scanWork = gcOverAssistWork
		debtBytes = int64(gcController.assistBytesPerWork * float64(scanWork))
	}

	bgScanCredit := atomic.Loadint64(&gcController.bgScanCredit)
	stolen := int64(0)
	if bgScanCredit > 0 {
		if bgScanCredit < scanWork {
			stolen = bgScanCredit
			gp.gcAssistBytes += 1 + int64(gcController.assistBytesPerWork*float64(stolen))
		} else {
			stolen = scanWork
			gp.gcAssistBytes += debtBytes
		}
		atomic.Xaddint64(&gcController.bgScanCredit, -stolen)
		scanWork -= stolen

		if scanWork == 0 {
			return
		}
	}
	...
}
```

该函数会先根据 Goroutine 的`gcAssistBytes`和垃圾收集控制器的配置计算需要完成的标记任务数量，如果全局信用`bgScanCredit`中有可用的点数，那么会减去该点数，因为并发执行没有加锁，所以全局信用可能会被更新成负值，然而在长期来看这不是一个比较重要的问题。

如果全局信用不足以覆盖本地的债务，运行时会在系统栈中调用`runtime.gcAssistAlloc1`执行标记任务，它会直接调用`runtime.gcDrainN`完成指定数量的标记任务并返回：

```go
func gcAssistAlloc(gp *g) {
	...
	systemstack(func() {
		gcAssistAlloc1(gp, scanWork)
	})
	...
	if gp.gcAssistBytes < 0 {
		if gp.preempt {
			Gosched()
			goto retry
		}
		if !gcParkAssist() {
			goto retry
		}
	}
}
```

如果在完成标记辅助任务后，当前 Goroutine 仍然入不敷出并且 Goroutine 没有被抢占，那么运行时会执行`runtime.gcParkAssist`；如果全局信用仍然不足，运行时会通过 runtime.gcParkAssist 将当前 Goroutine 陷入休眠、加入全局的辅助标记队列并等待后台标记任务的唤醒。

用于还债的 runtime.gcFlushBgCredit 的实现比较简单，如果辅助队列中不存在等待的 Goroutine，那么当前的信用会直接加到全局信用 bgScanCredit 中：

```go
func gcFlushBgCredit(scanWork int64) {
	if work.assistQueue.q.empty() {
		atomic.Xaddint64(&gcController.bgScanCredit, scanWork)
		return
	}

	scanBytes := int64(float64(scanWork) * gcController.assistBytesPerWork)
	for !work.assistQueue.q.empty() && scanBytes > 0 {
		gp := work.assistQueue.q.pop()
		if scanBytes+gp.gcAssistBytes >= 0 {
			scanBytes += gp.gcAssistBytes
			gp.gcAssistBytes = 0
			ready(gp, 0, false)
		} else {
			gp.gcAssistBytes += scanBytes
			scanBytes = 0
			work.assistQueue.q.pushBack(gp)
			break
		}
	}

	if scanBytes > 0 {
		scanWork = int64(float64(scanBytes) * gcController.assistWorkPerByte)
		atomic.Xaddint64(&gcController.bgScanCredit, scanWork)
	}
}
```

如果辅助队列不为空，上述函数会根据每个 Goroutine 的债务数量和已完成的工作决定是否唤醒这些陷入休眠的 Goroutine；如果唤醒所有的 Goroutine 后，标记任务量仍然有剩余，这些标记任务都会加入全局信用中。

!["全局信用与本地信用"](/images/global-credit-and-assist-bytes.png "全局信用与本地信用")

用户程序辅助标记的核心目的是避免用户程序分配内存影响垃圾收集器完成标记工作的期望时间，它通过维护账户体系保证用户程序不会对垃圾收集造成过多的负担，一旦用户程序分配了大量的内存，该用户程序就会通过辅助标记的方式平衡账本，这个过程会在最后达到相对平衡，保证标记任务在到达期望堆大小时完成。

### 标记终止

当所有处理器的本地任务都完成并且不存在剩余的工作 Goroutine 时，后台并发任务或者辅助标记的用户程序会调用`runtime.gcMarkDone`通知垃圾收集器。当所有可达对象都被标记后，该函数会将垃圾收集的状态切换至`_GCmarktermination`；如果本地队列中仍然存在待处理的任务，当前方法会将所有的任务加入全局队列并等待其他 Goroutine 完成处理：

```go
func gcMarkDone() {
	...
top:
	if !(gcphase == _GCmark && work.nwait == work.nproc && !gcMarkWorkAvailable(nil)) {
		return
	}

	gcMarkDoneFlushed = 0
	systemstack(func() {
		gp := getg().m.curg
		casgstatus(gp, _Grunning, _Gwaiting)
		forEachP(func(_p_ *p) {
			wbBufFlush1(_p_)
			_p_.gcw.dispose()
			if _p_.gcw.flushedWork {
				atomic.Xadd(&gcMarkDoneFlushed, 1)
				_p_.gcw.flushedWork = false
			}
		})
		casgstatus(gp, _Gwaiting, _Grunning)
	})

	if gcMarkDoneFlushed != 0 {
		goto top
	}
	...
}
```

如果运行时中不包含全局任务、处理器中也不存在本地任务，那么当前垃圾收集循环中的灰色对象也都标记成了黑色，我们就可以开始触发垃圾收集的阶段迁移了：

```go
func gcMarkDone() {
	...
	getg().m.preemptoff = "gcing"
	systemstack(stopTheWorldWithSema)

	...

	atomic.Store(&gcBlackenEnabled, 0)
	gcWakeAllAssists()
	schedEnableUser(true)
	nextTriggerRatio := gcController.endCycle()
	gcMarkTermination(nextTriggerRatio)
}
```

上述函数在最后会关闭混合写屏障、唤醒所有协助垃圾收集的用户程序、恢复用户 Goroutine 的调度并调用`runtime.gcMarkTermination`进入标记终止阶段：

```go
func gcMarkTermination(nextTriggerRatio float64) {
	atomic.Store(&gcBlackenEnabled, 0)
	setGCPhase(_GCmarktermination)

	_g_ := getg()
	gp := _g_.m.curg
	casgstatus(gp, _Grunning, _Gwaiting)

	systemstack(func() {
		gcMark(startTime)
	})
	systemstack(func() {
		setGCPhase(_GCoff)
		gcSweep(work.mode)
	})
	casgstatus(gp, _Gwaiting, _Grunning)
	gcSetTriggerRatio(nextTriggerRatio)
	wakeScavenger()

	...

	injectglist(&work.sweepWaiters.list)
	systemstack(func() { startTheWorldWithSema(true) })
	prepareFreeWorkbufs()
	systemstack(freeStackSpans)
	systemstack(func() {
		forEachP(func(_p_ *p) {
			_p_.mcache.prepareForSweep()
		})
	})
	...
}
```

我们省略了该函数中很多数据统计的代码，包括正在使用的内存大小、本轮垃圾收集的暂停时间、CPU 的利用率等数据，这些数据能够帮助控制器决定下一轮触发垃圾收集的堆大小，除了数据统计之外，该函数还会调用`runtime.gcSweep`重置清理阶段的相关状态并在需要时阻塞清理所有的内存管理单元；`_GCmarktermination`状态在垃圾收集中并不会持续太久，它会迅速转换至`_GCoff`并恢复应用程序，到这里垃圾收集的全过程基本上就结束了，用户程序在申请内存时才会惰性回收内存。

### 内存清理

垃圾收集的清理中包含对象回收器（Reclaimer）和内存单元回收器，这两种回收器使用不同的算法清理堆内存：

- 对象回收器在内存管理单元中查找并释放未被标记的对象，但是如果`runtime.mspan`中的所有对象都没有被标记，整个单元就会被直接回收，该过程会被`runtime.mcentral.cacheSpan`或者`runtime.sweepone`异步触发；
- 内存单元回收器会在内存中查找所有的对象都未被标记的`runtime.mspan`，该过程会被`runtime.mheap.reclaim`触发；

`runtime.sweepone`是我们在垃圾收集过程中经常会见到的函数，它会在堆内存中查找待清理的内存管理单元：

```go
func sweepone() uintptr {
	...
	var s *mspan
	sg := mheap_.sweepgen
	for {
		s = mheap_.nextSpanForSweep()
		if s == nil {
			break
		}
		if state := s.state.get(); state != mSpanInUse {
			continue
		}
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			break
		}
	}

	npages := ^uintptr(0)
	if s != nil {
		npages = s.npages
		if s.sweep(false) {
			atomic.Xadduintptr(&mheap_.reclaimCredit, npages)
		} else {
			npages = 0
		}
	}

	_g_.m.locks--
	return npages
}
```

查找内存管理单元时会通过`state`和`sweepgen`两个字段判断当前单元是否需要处理。如果内存单元的`sweepgen`等于`mheap.sweepgen - 2`那么意味着当前单元需要清理，如果等于`mheap.sweepgen - 1，`那么当前管理单元就正在清理。

所有的回收工作最终都是靠`runtime.mspan.sweep`完成的，它会根据并发标记阶段回收内存单元中的垃圾并清除标记以免影响下一轮垃圾收集。

## 小结

Go 语言垃圾收集器的实现非常复杂，作者认为这是编程语言中最复杂的模块，调度器的复杂度与垃圾收集器完全不是一个级别，我们在分析垃圾收集器的过程中不得不省略很多的实现细节，其中包括并发标记对象的过程、清扫垃圾的具体实现，这些过程设计大量底层的位操作和指针操作，本节中包含所有的相关代码的链接，感兴趣的读者可以自行探索。

垃圾收集是一门非常古老的技术，它的执行速度和利用率很大程度上决定了程序的运行速度，Go 语言为了实现高性能的并发垃圾收集器，使用三色抽象、并发增量回收、混合写屏障、调步算法以及用户程序协助等机制将垃圾收集的暂停时间优化至毫秒级以下，从早期的版本看到今天，我们能体会到其中的工程设计和演进，作者觉得研究垃圾收集的是实现原理还是非常值得的。
