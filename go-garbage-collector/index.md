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
