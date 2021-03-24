# Go Interview 03

<!--more-->
## Go中的for循环
for循环支持continue和break来控制循环，但是她提供了一个更高级break，可以选择中断那个循环，for循环不支持以逗号为间隔的多个赋值语句，必须使用平行赋值的方式来初始化多个变量
## Go中的switch
当个case中可以出现多个结果

只有在case中明确添加fallthrough关键字，才会继续执行紧跟的下一个case
## Go中没有隐藏的this指针，这句话是什么意思
方法施加的对象显示传递，没有被隐藏起来

golang的面向对象表达更直观，对于面向过程只是换了一种语法形式来表达

方法施加的对象不需要非得是指针，也不用非得叫this
## Go中引用类型包含什么
数组切片，字典(map)，通道(channel)，接口(interface)
## Go中指针运算有哪些
可以通过&取指针的地址

可以通过*取指针指向的数据
## 说说Go的main函数
main函数不能带参数

main函数不能定义返回值

main函数所在的包必须是main包

main函数中可以使用flag包来获取和解析命令行参数
## Go同步锁
1. 当一个goroutine获得了Mutex后，其他的goroutine就只能乖乖的等待了，除非该goroutine释放这个Mutex
2. RWMutex在读锁占用的情况下，会阻止写，但不会阻止写
3. RWMutex在写锁占用的情况下，会阻止任何goroutine(无论是读还是写)，整个锁相当于由该goroutine独占
## channel特性
1. 给一个nil channel发送数据，造成永久阻塞
2. 从一个nil channel接收数据，造成永久阻塞
3. 给一个已经关闭的channel发送数据，引起panic
4. 从一个已经关闭的channel接收数据，如果缓冲区为空，则返回一个零值
5. 无缓冲的channel是同步的，有缓冲的channel是非同步的
## Go触发异常的场景有哪些
1. 空指针解析
2. 下标越界
3. 除数为0
4. 调用panic函数
