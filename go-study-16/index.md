# Go Study 16


## 复制
将切片a中的元素复制到切片b中。

最简单的、最常用的方法就是使用内置的copy方法
```go
// 一次将内存申请到位
b = make([]T, len(a))
copy(b, a)
```

除了使用内置的copy函数外，还有下面两种使用append函数复制切片的方法
```go
b = append([]T(nil), a...)
b = append(a[:0:0], a...)
```

这两种方法通常比使用copy函数复制的方法要慢一点，但是如果在复制之后有更多的元素要添加到b中，那么它们的效率会更高。

## 剪切
将切片a中索引i~j位置的元素剪切掉

可以按照下面的方式，使用append函数完成
```go
a = append(a[:i], a[j:]...)
```

## 删除
将切片a中索引位置为i的元素删除。

同样可以使用上面剪切的方式使用append函数完成删除操作
```go
a = append(a[:i], a[i+1:]...)
```

或者搭配copy函数使用切片表达式完成删除操作
```go
a = a[:i + copy(a[i:], a[i+1])]
```

此外，如果只需要删除索引为i的元素，无需保留切片元素原有的顺序，那么还可以使用下面这种简单的方式删除
```go
// 将最后一个元素移到索引i处
a[i] = a[len(a) - 1]
// 截掉最后一个元素
a = a[:len(a) - 1]
```

## 剪切或删除操作可能引起的内存泄露
需要特别注意的是，如果切片a中的元素是一个指针类型或者包含指针字段的结构体类型(需要被垃圾回收)，上面的剪切和删除的示例代码会存在一个潜在的内存泄露问题：一些具有值的元素仍被切片a引用，因此无法被垃圾回收机制回收掉。下面代码可以解决这个问题。

### 剪切
```go
copy(a[i:], a[j:])
for k, n := len(a)-j+i, len(a); k < n; k++ {
    // 或类型T的零值
    a[k] = nil
}
a = a[:len(a)-j+i]
```

### 删除
```go
copy(a[i:], a[i+1:])
// 或类型T的零值
a[len(a)-1] = nil
a = a[:len(a)-1]
```

### 删除但不保留元素原有顺序
```go
a[i] = a[len(a)-1]
a[len(a)-1] = nil
a = a[:len(a)-1]
```

## 内部扩张
在切片a的索引i之后扩张j个元素

使用两个append函数完成，即先将索引i之后的元素追加到一个长度为j的切片后，再将这个切片中的所有元素追加到切片a的索引i之后
```go
a = append(a[:i], append(make([]T, j), a[i:]...)...)
```

扩张的这一部分元素为T类型的零值
## 尾部扩张
将切片a的尾部扩张j个元素的空间。
```go
a = append(a, make([]T, j)...)
```

扩张的这一部分元素同样为T类型的零值

## 过滤
按照一定的规则将切片a中的元素进行就地过滤

这里假设过滤的条件已经封装为keep函数，使用for range遍历切片a的所有元素逐一调用keep函数进行过滤
```go
n := 0
for _, x := range a {
    if keep(x) {
        // 保留该元素
        a[n] = x
        n++
    }
}
// 截取切片中需保留的元素
a = a[:n]
```

## 插入
将元素x插入切片a的索引i处

还是使用两个append函数完成插入x的操作
```go
a = append(a[:i], append([]T{x}, a[i:...])...)
```

第二个append函数创建了一个具有自己底层数组的新切片，并将a[i:] 中的元素复制到该切片，然后由第一个append函数将这些元素复制回切片。

我们可以通过使用另一个方法来避免新切片的创建(以及由此产生的内存垃圾)和第二个副本：
```go
// 这里应使用元素的零值
a = append(a, 0)
copy(a[i+1:], a[i:])
a[i] = x
```

## 追加
将元素x追加到切片a的最后

这里使用append函数即可
```go
a = append(a, x)
```

## 弹出
将切片a的最后一个元素弹出

这里使用切片表达式完成弹出操作
```go
x, a = a[len(a)-1], a[:len(a)-1]
```

弹出切片a的第一个元素
```go
x, a = a[0], a[i:]
```

## 前插
将元素x前插到切片a的最开始
```go
a = append([]T{x}, a...)
```

## 其他技巧

### 过滤而不分配内存
此技巧使用了一个事实，即切片b与原始切片a共享相同的底层数组和容量，因此原存储空间已重新用于过滤后的切片。当然原始切片的内容被修改了。
```go
b := a[:0]
for _, x := range a {
    if f(x) {
        b = append(b, x)
    }
}
```
对于必须被垃圾回收的元素，在完成上述操作后可以添加一下代码
```go
for i := len(b); i < len(a); i++ {
    a[i] = nil
}
```

### 翻转
将切片a的元素顺序翻转

通过迭代两两互换元素完成
```go
for i := len(a)/2-1; i >= 0; i-- {
    opp := len(a)-1-i
    a[i], a[opp] = a[opp], a[i]
}
```
同样的操作：
```go
for left, right := 0, len(a)-1; left < right; left, right = left+1, right-1 {
    a[left], a[right] = a[right], a[left]
}
```

### 洗牌
打乱切片a中元素的顺序

Fisher-Yates算法：
```go
for i := len(a)-1; i > 0; i-- {
    j := rand.Intn(i+1)
    a[i], a[j] = a[j], a[i]
}
```
从go1.10开始，可以使用math/rand.Shuffle
```go
rand.Shuffle(len(a), func(i, j int) {
    a[i], a[j] = a[j], a[i]
})
```

### 使用最小分配进行批处理
如果你想对一个大型切片a的元素分批进行处理，这会很有用。
```go
actions := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
batchSize := 3
batches := make([][]int, 0, (len(actions)+batchSize-1)/batchSize)

for batchSize < len(actions) {
    actions, batches = actions[batchSize:], append(batches, actions[0:batchSize:batchSize])
}
batches = append(batches, actions)
```

得到的效果如下：
```bash
[[0, 1, 2], [3, 4, 5], [6, 7, 8], [9]]
```

### 原地删除重复元素(元素可比较)
```go
import "sort"

// 切片元素可以是任何可排序的类型
in := []int{3, 2, 1, 4, 3, 2, 1, 4, 1}
sort.Ints(in)
j := 0
for i := 1; i < len(in); i++ {
    if in[j] == in[i] {
        continue
    }
    j++
    // 需要保存原始数据时
    // in[i], in[j] = in[j], in[i]
    // 只需要保存需要的数据时
    in[j] = in[i]
}
result := in[:j+1]
// [1, 2, 3, 4]
fmt.Println(result)
```

### 存在就移到前面，不存在就插到前面
如果给定的元素在切片中存在则把该元素移到切片的头部，如果不存在则将该元素插入到切片的头部。

```go
func moveToFront(needle string, haystack []string) []string {
    if len(haystack) != 0 && haystack[0] == needle {
        return haystack
    }
    prev := needle
    for i, elem := range haystack {
        switch {
        case i == 0:
            haystack[0] = needle
            prev = elem
        case elem == needle:
            haystack[i] = prev
            return haystack
        default:
            haystack[i] = prev
            prev = elem
        }
    }
    return append(haystack, prev)
}

// [a, b, c, d, e]
haystack := []string{"a", "b", "c", "d", "e"}
haystack = moveToFront("c", haystack)
haystack = moveToFront("f", haystack)
```

### 滑动窗口
将切片input生成size大小的滑动窗口

```go
func slidingWindow(size int, input []int) [][]int {
    // 返回入参的切片作为第一个元素
    if len(input) <= size {
        return [][]int{input}
    }

    // 以所需的精确大小分配切片
    r := make([][]int, 0, len(input)-size+1)

    for i, j := 0, size; j <= len(input); i, j = i+1, j+1 {
        r = append(r, input[i:j])
    }

    return r
}
```

示例：
```go
a := []int{1, 2, 3, 4, 5}
res L= slidingWindow(2, a)
fmt.Println(res)
```

输出：
```go
[[1, 2] [2, 3] [3, 4] [4, 5]]
```
