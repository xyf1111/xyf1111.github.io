# Go语言数组实现原理


## 概述

数组是由相同类型元素的集合组成的数据结构，计算机会为数组分配一块连续的内存来保存其中的元素，我们可以利用数组中的索引快速访问特定元素，常见的数组都是一堆的线性数组，而多维数组在数值和图形计算领域却有比较常见的应用

!["多维数组"](/images/3D-array.jpg "多维数组")

数组作为一种基本的数据类型，通常会从两个维度描述数组，也就是数组中存储的元素类型和数组最大能存储的元素个数，在Go语言中往往会使用如下所示的方式来表示数组类型

```go
[10]int
[200]interface{}
```

Go语言数组在初始化之后大小就无法改变，存储元素类型相同、但是大小不同的数组类型在Go语言看来也是完全不同的，只有两个条件都相同才是同一类型

```go
func NewArray(elem *Type, bound int64) *Type {
	if bound < 0 {
		Fatalf("NewArray: invalid bound %v", bound)
	}
	t := New(TARRAY)
	t.Extra = &Array{Elem: elem, Bound: bound}
	t.SetNotInHeap(elem.NotInHeap())
	return t
}
```

编译期间的数组类型是由上述的`cmd/compile/internal/types.NewArray`函数生成的，该类型包含两个字段，分别是元素类型`Elem`和数组的大小`Bound`，这两个字段共同构成了数组类型，而当前数组是否应该在堆栈中初始化也在编译期就确定了。

## 初始化

Go语言的数组有两种不同的创建方式，一种是显式的指定数组大小，另一种是使用`[...]T`声明数组，Go语言会在编译期间通过源代码推导数组的大小：

```go
arr1 := [3]int{1, 2, 3}
arr2 := [...]int{1, 2, 3}
```

上述两种声明方式在运行期间得到的结果是完全相同的，后一种声明方式在编译期间就会被转换成前一种，这也就是编译器对数组大小的推导，下面介绍编译器的推导过程

**上限推导**

两种不同的声明方式会导致编译器做出完全不同的处理，如果使用第一种方式`[10]T`，那么变量的类型在编译进行到类型检查阶段就会被提取出来，随后使用`cmd/compile/internal/types.NewArray`创建包含数组大小的`cmd/compile/internal/types.Array`结构体

当我们使用`[...]T`的方式声明数组时，编译器会在`cmd/compile/internal/gc.typecheckcomplit`函数中对该数组的大小进行推导：

```go
func typecheckcomplit(n *Node) (res *Node) {
	...
	if n.Right.Op == OTARRAY && n.Right.Left != nil && n.Right.Left.Op == ODDD {
		n.Right.Right = typecheck(n.Right.Right, ctxType)
		if n.Right.Right.Type == nil {
			n.Type = nil
			return n
		}
		elemType := n.Right.Right.Type

		length := typecheckarraylit(elemType, -1, n.List.Slice(), "array literal")

		n.Op = OARRAYLIT
		n.Type = types.NewArray(elemType, length)
		n.Right = nil
		return n
	}
	...

	switch t.Etype {
	case TARRAY:
		typecheckarraylit(t.Elem(), t.NumElem(), n.List.Slice(), "array literal")
		n.Op = OARRAYLIT
		n.Right = nil
	}
}
```

这个删减后的`cmd/compile/internal/gc.typecheckcomplit`会调用`cmd/compile/internal/gc.typecheckarraylist`通过遍历元素的方式来计算数组中的元素的数量

所以可以看出`[...]T{1, 2, 3}`和`[3]T{1, 2, 3}`在运行时是完全等价的，`[...]T`这种初始化方式也只是Go语言为我们提供的一种语法糖，当不想计算数组中的元素个数时可以通过这种方法减少一些工作量。

**语句转换**

对于一个有字面量组成的数组，根据数组元素数量的不同，编译器会在负责初始化字面量的`cmd/compile/internal/gc.anylit`函数中做两种不同的优化：
1. 当元素数量小于或者等于四个时，会直接将数组中的元素置在栈上
2. 当元素数量大于四个时，会将数组中的元素放置到静置区并在运行时取出

```go
func anylit(n *Node, var_ *Node, init *Nodes) {
	t := n.Type
	switch n.Op {
	case OSTRUCTLIT, OARRAYLIT:
		if n.List.Len() > 4 {
			...
		}

		fixedlit(inInitFunction, initKindLocalCode, n, var_, init)
	...
	}
}
```

当数组元素小于或者等于四个时，`cmd/compile/internal/gc.fixedlit`会负责在函数编译之前将`[3]{1,2,3}`转换成更加原始的语句

```go
func fixedlit(ctxt initContext, kind initKind, n *Node, var_ *Node, init *Nodes) {
	var splitnode func(*Node) (a *Node, value *Node)
	...

	for _, r := range n.List.Slice() {
		a, value := splitnode(r)
		a = nod(OAS, a, value)
		a = typecheck(a, ctxStmt)
		switch kind {
		case initKindStatic:
			genAsStatic(a)
		case initKindLocalCode:
			a = orderStmtInPlace(a, map[string][]*Node{})
			a = walkstmt(a)
			init.Append(a)
		}
	}
}
```

当数组中元素的个数小于或者等于四个并且`cmd/compile/internal/gc.fixedlit`函数接收的`kind`是`initKindLocalCode`时，上述代码会将原有的初始化语句`[3]int{1, 2, 3}`拆分成一个声明变量的表达式和几个赋值表达式，这些表达式会完成对数组的初始化：

```
var arr [3]int

arr[0] = 1
arr[1] = 2
arr[2] = 3
```

但是如果当前数组的元素大于四个，`cmd/compile/internal/gc.anylit`会先获取一个唯一的`staticname`，然后调用`cmd/compile/internal/gc.fixedlit`函数在静态存储区初始化数组中的元素并将临时变量赋值给数组

```go
func anylit(n *Node, var_ *Node, init *Nodes) {
	t := n.Type
	switch n.Op {
	case OSTRUCTLIT, OARRAYLIT:
		if n.List.Len() > 4 {
			vstat := staticname(t)
			vstat.Name.SetReadonly(true)

			fixedlit(inNonInitFunction, initKindStatic, n, vstat, init)

			a := nod(OAS, var_, vstat)
			a = typecheck(a, ctxStmt)
			a = walkexpr(a, init)
			init.Append(a)
			break
		}

		...
	}
}
```

假设代码需要初始化`[5]int{1,2,3,4,5}`，那么可以将上述过程理解成以下的伪代码

```go
var arr [5]int
statictmp_0[0] = 1
statictmp_0[1] = 2
statictmp_0[2] = 3
statictmp_0[3] = 4
statictmp_0[4] = 5
arr = statictmp_0
```

总结起来，在不考虑逃逸分析的情况下，如果数组中的元素小于或者等于四个，那么所有的变量会直接在栈上初始化，如果数组元素大于四个，变量就会在静态存储区初始化然后拷贝到栈上，这些转换后的代码才会继续进入中间代码生成和机器生成两个阶段，最后生成可以执行的二进制文件

## 访问和赋值

无论是在栈上还是静态存储区，数组在内存中都是一连串的内存空间，通过指向数组开头的指针、元素的数量以及元素类型占的空间大小表示数组。如果不知道数组中元素的数量，访问时可能会发生越界；而如果不知道数组中元素类型的大小，就没有办法知道应该一次取出多少字节的数据，无论丢失了哪个信息，都无法知道这片连续的内存空间到底存储了什么数据

!["数组的内存空间"](/images/golang-array-memory.jpg "数组的内存空间")

数组访问越界是非常严重的错误，Go语言中可以在编译期间的静态类型检查判断数组越界，`cmd/compile/internal/gc.typecheck1`会验证访问数组的索引

```go
func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	case OINDEX:
		ok |= ctxExpr
		l := n.Left  // array
		r := n.Right // index
		switch n.Left.Type.Etype {
		case TSTRING, TARRAY, TSLICE:
			...
			if n.Right.Type != nil && !n.Right.Type.IsInteger() {
				yyerror("non-integer array index %v", n.Right)
				break
			}
			if !n.Bounded() && Isconst(n.Right, CTINT) {
				x := n.Right.Int64()
				if x < 0 {
					yyerror("invalid array index %v (index must be non-negative)", n.Right)
				} else if n.Left.Type.IsArray() && x >= n.Left.Type.NumElem() {
					yyerror("invalid array index %v (out of bounds for %d-element array)", n.Right, n.Left.Type.NumElem())
				}
			}
		}
	...
	}
}
```

1. 访问数组的索引是非整数时，报错"non-integer array index %v"
2. 访问数组的索引是负数时，报错"invalid array index %v(index must be non-negative)"
3. 访问数组的索引越界时，报错"invalid array index %v(out of bounds for %d-element array)"

数组和字符串的一些简单越界错误都会在编译期间发现，例如：直接使用整数或者常量访问数组；但是如果使用变量去访问数组或者字符串时，编译器就无法提前发现错误，我们需要Go语言运行时阻止不合法的访问

```go
arr[4]: invalid array index 4 (out of bounds for 3-element array)
arr[i]: panic: runtime error: index out of range [4] with length 3
```

Go语言运行时在发现数组、切片和字符串的越界操作会由运行时的`runtime.panicIndex`和`runtime.goPanicIndex`触发程序的运行时错误并导致崩溃退出

```go
TEXT runtime·panicIndex(SB),NOSPLIT,$0-8
	MOVL	AX, x+0(FP)
	MOVL	CX, y+4(FP)
	JMP	runtime·goPanicIndex(SB)

func goPanicIndex(x int, y int) {
	panicCheck1(getcallerpc(), "index out of range")
	panic(boundsError{x: int64(x), signed: true, y: y, code: boundsIndex})
}
```

当数组的访问操作`OINDEX`成功通过编译器的检查后，会被转换成几个SSA指令，假设有如下Go语言代码，通过如下的方式进行编译会得到ssa.html文件：

```go
package check

func outOfRange() int {
	arr := [3]int{1, 2, 3}
	i := 4
	elem := arr[i]
	return elem
}

$ GOSSAFUNC=outOfRange go build array.go
dumped SSA to ./ssa.html
```

`start`阶段生成的SSA代码就是优化之前的第一版中间代码，下面展示的部分是`elem := arr[i]`对应的中间代码，在这段中间代码中我们发现Go语言为数组的访问操作生成了判断数组上限的指令`IsInBounds`以及当条件不满足时触发程序崩溃的`PanicBounds`指令：

```go
b1:
    ...
    v22 (6) = LocalAddr <*[3]int> {arr} v2 v20
    v23 (6) = IsInBounds <bool> v21 v11
If v23 → b2 b3 (likely) (6)

b2: ← b1-
    v26 (6) = PtrIndex <*int> v22 v21
    v27 (6) = Copy <mem> v20
    v28 (6) = Load <int> v26 v27 (elem[int])
    ...
Ret v30 (+7)

b3: ← b1-
    v24 (6) = Copy <mem> v20
    v25 (6) = PanicBounds <mem> [0] v21 v11 v24
Exit v25 (6)
```

编译器会将`PanicBounds`指令转换成上面提到的`runtime.panicIndex`函数，当数组下标没有越界时，编译器会先获取数组的内存地址和访问的下标、利用`PtrIndex`计算出目标元素的地址，最后使用`Load`操作将指针中的元素加载到内存中。

当然只有当编译器无法对数组下标是否越界做出判断时才会加入`PanicBounds`指令交给运行时进行判断，在使用字面量整数访问数组下标时会生成非常简单的中间代码，当将上述代码中的`arr[i]`改成`arr[2]`时，就会得到如下所示的代码：

```go
b1:
    ...
    v21 (5) = LocalAddr <*[3]int> {arr} v2 v20
    v22 (5) = PtrIndex <*int> v21 v14
    v23 (5) = Load <int> v22 v20 (elem[int])
    ...
```

Go语言对于数组的访问还是有着比较多的检查，它不仅会在编译期间提前发现一些简单的越界错误并插入用于检测数组上限的函数调用，还会在运行期间通过插入的函数保证不会发生越界

数组的赋值和更新操作`a[i]=2`也会生成SSA，生成期间计算出数组当前元素的内存地址，然后修改当前内存地址的内容，这些赋值语句会被转换成如下所示的SSA代码

```go
b1:
    ...
    v21 (5) = LocalAddr <*[3]int> {arr} v2 v19
    v22 (5) = PtrIndex <*int> v21 v13
    v23 (5) = Store <mem> {int} v22 v20 v19
    ...
```

赋值的过程中会先确定目标数组的地址，再通过`PtrIndex`获取目标元素的地址，最后使用`Store`指令将数据存入地址中，从上面的这些SSA代码中可以看出上述数组寻址和赋值都是在编译阶段完成的，没有运行时的参与

## 小结

数组是Go语言中最重要的数据结构，了解它的实现能够帮助我们更好的理解这门语言，通过对其实现的分析，我们知道了对数组的访问和赋值需要同时依赖编译器和运行时，它的大多数操作在编译期间都会转换成直接读写内存，在中间代码生成期间，编译器还会插入运行时方法`runtime.panicIndex`调用防止发生越界错误
