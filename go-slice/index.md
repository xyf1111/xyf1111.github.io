# Go Slice


数组在Go语言中没那么常用，更常用的数据结构是切片，即动态数组，其长度并不固定，我们可以向切片中追加元素，它会在容量不足时自动扩容。

在Go语言中，切片类型的声明方式和数组有一些相似，不过由于切片的长度是动态的，所以声明时只需要指定切片中的元素类型

```go
[]int
[]interface{}
```

从切片的定义我们能推测出，切片在编译期间生成的类型会包含切片中的元素类型，即`int`或者`interface()`等。`cmd/compile/internal/type.NewSlice`就是编译期间用于创建切片类型的函数

```go
func NewSlice(elem *Type) *Type {
    if t := elem.Cache.slice; t != nil {
        if t.Elem() != elem {
            Fatalf("elem mismatch")
        }
        return t
    }

    t := New(TSLICE)
    t.Extra = Slice{
        Elem: elem,
    }
    elem.Cache.slice = t
    return t
}
```

上述方法返回结构体重的`Extra`字段是一个只包含切片内元素类型的结构，也就是说切片内元素的类型都是在编译期间确定的，编译器确定了类型之后，会将类型存储在`Extra`字段中帮助程序在运行时动态获取。

## 数据结构

编译期间的切片是`cmd/compile/internal/type.Slice`类型的，但是在运行时切片可以由如下`reflect.SliceHeader`结构体表示，其中

- `Data` 是指向数组的指针
- `Len` 是当前切片的长度
- `Cap` 是当前切片的容量，即`Data`数组的大小

```go
type SliceHeader struct {
    Data    uintptr
    Len     int
    Cap     int
}
```

`Data`是一片连续的内存空间，这片内存空间可以用于存储切片中的全部元素，数组中的元素只是逻辑上的概念，底层存储其实都是连续的，所以我们可以将切片理解成一片连续的内存空间加上长度和容量的标识

!["Go语言切片结构体"](/images/golang-slice-struct.png "Go语言切片结构体")

从上图中，我们会发现切片和数组的关系非常密切，切片引入了一个抽象层，提供了对数组中部分连续片段的引用，而作为数组的引用，我们可以在运行区间修改它的长度和范围。当切片底层的数组长度不足时就会触发扩容，切片指向的数组可能会发生变化，不过在上层看来切片是没有变化的，上层只需要与切片打交道不需要关心数组的变化

编译器在编译期间简化了获取数组大小、读写数组中的元素等操作：因为数组的内存固定且连续，多数操作都会直接读写内存的特定位置。但是切片是运行时才会确定内容的结构，所有操作还需要依赖Go语言的运行时。

## 初始化

Go语言包含三种初始化切片的方式

1. 通过下标的方式获得数组或者切片的一部分
2. 使用字面量初始化新的切片
3. 使用关键字`make`创建切片

```go
arr[0:3] or slice[0:3]
slice := []int{1, 2, 3}
slice := make([]int, 10)
```

**使用下标**

使用下标创建切片是最原始也最接近汇编语言的方式，它是所有方法中最为底层的一种，编译器会将`arr[0:3]`或者`slice[0:3]`等语句转换成`OpSliceMake`操作，可以通过下面代码来验证

```go
package opslicemake

func newSlice() []int {
    arr := [3]int{1, 2, 3}
    slice := arr[0:1]
    return slice
}
```

通过`GOSSAFUNC`变量编译上述代码可以得到一系列SSA中间代码，其中`slice := arr[0:1]`语句在"decompose builtin"阶段对应的代码如下所示

```go
v27 (+5) = SliceMake <[]int> v11 v14 v17

name &arr[*[3]int]: v11
name slice.ptr[*int]: v11
name slice.len[int]: v14
name slice.cap[int]: v17
```

`SliceMake`操作会接受四个参数创建新的切片，元素类型、数组指针、切片大小和容量，这也是我们在数据结构一节中提到的切片的几个字段，需要注意的是使用下标初始化切片不会拷贝原数组或者切片中的数据，它只会创建一个指向原数组的切片结构体，所以修改新切片的数据也会修改原切片

**字面量**

当使用字面量`[]int{1, 2, 3}`创建新的切片时，`cmd/compile/internal/gc.slicelit`函数会在编译期间将它展开成如下所示的代码片段

```go
var vstat [3]int
vstat[0] = 1
vstat[1] = 2
vstat[2] = 3
var vauto *[3]int = new([3]int)
*vauto = vstat
slice := vauto[:]
```

1. 根绝切片中的元素数量对底层数据的大小进行推断并创建一个数组
2. 将这些字面量元素存储到初始化的数组中
3. 创建一个同样指向`[3]int`类型的数组指针
4. 将静态存储区的数组`vstat`赋值给`vauto`指针所在的地址
5. 通过`[:]`操作获取一个底层使用`vauto`的切片

第五步中的`[:]`就是使用下标创建切片的方法，从这一点我们也能看出`[:]`操作切片最底层的一种方法

**关键字**

如果使用字面量的方式创建切片，大部分工作都会在编译期间完成。但是当我们使用`make`关键字创建切片时，很多工作都需要运行时参与；调用方必须向`make`函数传入切片的大小以及可选的容量，类型检查期间的`cmd/compile/internal/gc.typecheck1`函数会校验入参

```go
func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	...
	case OMAKE:
		args := n.List.Slice()

		i := 1
		switch t.Etype {
		case TSLICE:
			if i >= len(args) {
				yyerror("missing len argument to make(%v)", t)
				return n
			}

			l = args[i]
			i++
			var r *Node
			if i < len(args) {
				r = args[i]
			}
			...
			if Isconst(l, CTINT) && r != nil && Isconst(r, CTINT) && l.Val().U.(*Mpint).Cmp(r.Val().U.(*Mpint)) > 0 {
				yyerror("len larger than cap in make(%v)", t)
				return n
			}

			n.Left = l
			n.Right = r
			n.Op = OMAKESLICE
		}
	...
	}
}
```

上面的函数不仅会检查`len`是否传入，还会保证传入的容量`cap`一定大于或者等于`len`。除了校验参数之外，当前函数会将`OMAKE`节点转换成`OMAKESLICE`，中间代码生成的`cmd/compile/internal/gc.walkexpr`函数会依据下面两个条件转换`OMAKESLICE`类型的节点：

1. 切片的大小和容量是否足够小
2. 切片是否发生了逃逸，最终在堆上初始化

当切片发生逃逸或者非常大时，运行时需要`runtime.makeslice`在堆上初始化切片，如果当前的切片不会发生逃逸并且切片非常小的时候，`make([]int, 3, 4)`会被直接转换成如下所示的代码：

```go
var arr [4]int
n := arr[:3]
```

上述代码会初始化数组并通过下标`[:3]`得到数组对应的切片，这两部分都会在编译阶段完成，编译器会在栈上或者静态存储区创建数组并将`[:3]`转换成上一节提到的`OpSliceMake`操作

分析了主要由编译器处理的分支后，我们回到用于创建切片时的运行时函数`runtime.makeslice`，这个函数的实现很简单：

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

上述函数的主要工作是计算切片占用的内存空间并在堆上申请一片连续的内存，它使用如下的方式计算占用的内存

内存空间=切片中元素大小×切片容量

虽然编译期间可以检查出很多错误，但是在创建切片的过程中如果发生了以下错误会直接出发运行时错误并崩溃：

1. 内存空间的大小发生了溢出
2. 申请的内存大于最大可支配的内存
3. 传入的长度小于0或者长度大于容量

`runtime.makeslice`在最后调用的`runtime.mallocgc`是用于申请内存的函数，这个函数的实现还是比较复杂的，如果遇到了比较小的对象会直接初始化在Go语言调度器里面的P结构中，而大于32KB的对象会在堆上初始化。

在之前版本的Go语言中，数组指针、长度和容量会被合成一个`runtime.slice`结构，但是从`cmd/compile:move slice construction to callers of makeslice`提交之后，构建结构体`reflect.SliceHeader`的工作就交给了`runtime.makeslice`的调用方，该函数仅会返回指向底层数组的指针，调用方会在编译期间构建切片结构体：

```go
func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	...
	case OSLICEHEADER:
	switch 
		t := n.Type
		n.Left = typecheck(n.Left, ctxExpr)
		l := typecheck(n.List.First(), ctxExpr)
		c := typecheck(n.List.Second(), ctxExpr)
		l = defaultlit(l, types.Types[TINT])
		c = defaultlit(c, types.Types[TINT])

		n.List.SetFirst(l)
		n.List.SetSecond(c)
	...
	}
}
```

`OSLICEHEADER`操作会创建`reflect.SliceHeader`结构体，其中包含切片的长度和容量，它是切片在运行时的表示：

```go
type SliceHeader struct {
    Data    uintptr
    Len     int
    Cap     int
}
```

正是因为大多数对切片类型的操作并不需要直接操作原来的`runtime.slice`结构体，所以`reflect.SliceHeader`的引入能够减少切片初始化时的少量开销，该改动不仅能够减少~0.2%的Go语言包大小，还能够减少92个`runtime.panicIndex`的调用，占Go语言二进制的~3.5%

## 访问元素

使用`len`和`cap`获取长度或者容量是切片最常见的操作，编译器将它们看成两种特殊操作，即`OLEN`和`OCAP`，`cmd/compile/internal/gc.state.expr`函数会在SSA生成阶段将它们分别转换成`OpSliceLen`和`OpSliceCap`

```go
func (s *state) expr(n *Node) *ssa.Value {
	switch n.Op {
	case OLEN, OCAP:
		switch {
		case n.Left.Type.IsSlice():
			op := ssa.OpSliceLen
			if n.Op == OCAP {
				op = ssa.OpSliceCap
			}
			return s.newValue1(op, types.Types[TINT], s.expr(n.Left))
		...
		}
	...
	}
}
```

访问切片中的字段可能会触发"decompose builtin"阶段的优化，`len(slice)`或者`cap(slice)`在一些情况下会直接替换成切片的长度或者容量，不需要在运行时获取

```go
(SlicePtr (SliceMake ptr _ _ )) -> ptr
(SliceLen (SliceMake _ len _)) -> len
(SliceCap (SliceMake _ _ cap)) -> cap
```

除了直接获取切片的长度和容量之外，访问切片中元素使用`OINDEX`操作也会在中间代码生成期间转换成对地址的直接访问

```go
func (s *state) expr(n *Node) *ssa.Value {
	switch n.Op {
	case OINDEX:
		switch {
		case n.Left.Type.IsSlice():
			p := s.addr(n, false)
			return s.load(n.Left.Type.Elem(), p)
		...
		}
	...
	}
}
```

切片的操作基本都是在编译期间完成的，除了访问切片的长度、容量或者其中的元素之外，编译期间也会将包含`range`关键字的遍历转换成形式更简单的循环

## 追加和扩容

使用`append`关键字向切片中追加元素也是常见的切片操作，中间代码生成阶段的`cmd/compile/internal/gc.state.append`方法会根据返回值是否会覆盖原变量，选择进入两种流程，如果`append`返回的新切片不需要赋值回原有的变量，就会进入如下的处理流程：

```go
// append(slice, 1, 2, 3)
ptr, len, cap := slice
newlen := len + 3
if newlen > cap {
    ptr, len, cap = growslice(slice, newlen)
    newlen = len + 3
}
*(ptr+len) = 1
*(ptr+len+1) = 2
*(ptr+len+2) = 3
return makeslice(ptr, newlen, cap)
```

我们会先解构切片结构体获取它的数组指针、大小和容量，如果在追加元素后切片的大小大于容量，那么就会调用`runtime.growslice`对切片进行扩容并将新的元素依次加入切片。

如果使用`slice = append(slice, 1, 2, 3)`语句，那么 append 后的切片会覆盖原切片，这时`cmd/compile/internal/gc.state.append`方法会使用另一种方式展开关键字：

```go
// slice = append(slice, 1, 2, 3)
a := &slice
ptr, len, cap := slice
newlen := len + 3
if uint(newlen) > uint(cap) {
   newptr, len, newcap = growslice(slice, newlen)
   vardef(a)
   *a.cap = newcap
   *a.ptr = newptr
}
newlen = len + 3
*a.len = newlen
*(ptr+len) = 1
*(ptr+len+1) = 2
*(ptr+len+2) = 3
```

是否覆盖原变量的逻辑其实差不多，最大的区别在于得到的新切片是否会赋值回原变量。如果我们选择覆盖原有的变量，就不需要担心切片发生拷贝影响性能，因为Go语言编译器已经对这种常见的情况做出了优化。

!["Go语言的切片追加元素"](/images/golang-slice-append.png "Go语言的切片追加元素")

到这里我们已经清楚了Go语言如何在切片容量足够时向切片中追加元素，不过仍然需要研究切片容量不足时的处理流程。当切片的容量不足时，我们会调用`runtime.growslice`函数为切片扩容，扩容是为切片分配新的内存空间并拷贝原切片中元素的过程，我们先来看新切片的容量是如何确定的：

```go
func growslice(et *_type, old slice, cap int) slice {
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			if newcap <= 0 {
				newcap = cap
			}
		}
    }
}
```

在分配内存空间之前需要先确定新的切片容量，运行时根据切片的当前容量选择不同的策略进行扩容：

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果当前切片的长度小于 1024 就会将容量翻倍；
3. 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；

上述代码片段仅会确定切片的大致容量，下面还需要根据切片中的元素大小对齐内存，当数组中元素所占的字节大小为 1、8 或者 2 的倍数时，运行时会使用如下所示的代码对齐内存：

```go
    var overflow bool
	var lenmem, newlenmem, capmem uintptr
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		...
	default:
		...
	}
```

`runtime.roundupsize`函数会将待申请的内存向上取整，取整时会使用`runtime.class_to_size`数组，使用该数组中的整数可以提高内存的分配效率并减少碎片，我们会在内存分配一节详细介绍该数组的作用：

```go
var class_to_size = [_NumSizeClasses]uint16{
    0,
    8,
    16,
    32,
    48,
    64,
    80,
    ...,
}
```

在默认情况下，我们会将目标容量和元素大小相乘得到占用的内存。如果计算新容量时发生了内存溢出或者请求内存超过上限，就会直接崩溃退出程序，不过这里为了减少理解的成本，将相关的代码省略了。

```go
    var overflow bool
	var newlenmem, capmem uintptr
	switch {
	...
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, _ = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}
	...
	var p unsafe.Pointer
	if et.kind&kindNoPointers != 0 {
		p = mallocgc(capmem, nil, false)
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		p = mallocgc(capmem, et, true)
		if writeBarrier.enabled {
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)
	return slice{p, old.len, newcap}
```

如果切片中的元素不是指针类型，那么会调用`runtime.memclrNoHeapPointers`将超出切片当前长度的位置清空并在最后使用`runtime.memmove`将原数组内存中的内容拷贝到新申请的内存中。

`runtime.growslice`函数最终会返回一个新的切片，其中包含了新的数组指针、大小和容量，这个返回的三元组最终会覆盖原切片

```go
var arr []int64
arr = append(arr, 1, 2, 3, 4, 5)
```

简单总结一下扩容的过程，当我们执行上述代码时，会触发`runtime.growslice`函数扩容`arr`切片并传入期望的新容量5，这时期望分配的内存大小为40字节；不过因为切片中的元素大小等于`sys.PtrSize`，所以运行时会调用`runtime.roundupsize`向上取整内存的大小到48字节，所以心切片的容量为48/8=6。

## 拷贝切片

切片的拷贝虽然不是常见的操作，但是确实我们学习切片实现原理必须要涉及的。当我们使用`copy(a, b)`的形式对切片进行拷贝时，编译期间的`cmd/compile/internal/gc.copyany`也会分两种情况进行处理拷贝操作，如果当前`copy`不是在运行时调用的，`copy(a, b)`会被直接转换成下面的代码：

```go
n := len(a)
if n > len(b) {
    n = len(b)
}
if a.ptr != b.ptr {
    memmove(a.ptr, b.ptr, n*sizeof(elem(a))) 
}
```

上述代码中的`runtime.memmove`会负责拷贝内存。而如果拷贝是在运行时发生的，例如`go copy(a, b)`，编译器会使用`runtime.slicecopy`替换运行期间调用的`copy`，该函数的实现很简单：

```go
func slicecopy(to, fm slice, width uintptr) int {
	if fm.len == 0 || to.len == 0 {
		return 0
	}
	n := fm.len
	if to.len < n {
		n = to.len
	}
	if width == 0 {
		return n
	}
	...

	size := uintptr(n) * width
	if size == 1 {
		*(*byte)(to.array) = *(*byte)(fm.array)
	} else {
		memmove(to.array, fm.array, size)
	}
	return n
}
```

无论是编译期间拷贝还是运行时拷贝，两种拷贝方式都会通过`runtime.memmove`将整块内存的内容拷贝到目标的内存区域中：

!["Go语言切片的拷贝"](/images/golang-slice-copy.png "Go语言切片的拷贝")


相比于依次拷贝元素，`runtime.memmove`能够提供更好的性能。需要注意的是，整块拷贝内存仍然会占用非常多的资源，在大切片上执行拷贝操作时一定要注意对性能的影响

## 小结

切片的很多功能都是由运行时实现的，无论是初始化切片，还是对切片进行追加或扩容都需要运行时的支持，需要注意的是在遇到大切片扩容或者复制时可能会发生大规模的内存拷贝，一定要减少类似操作避免影响程序的性能
