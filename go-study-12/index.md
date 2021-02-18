# Go Study 12

<!--more-->
## defer调用
- 确保调用在函数结束时发生
- 多defer相当于栈(先进后出)
- 参数在defer语句时计算 
## 何时使用defer调用
- Open/Close
- Lock/Unlock
- PrintHeader/PrintFooter
## 错误处理二
- 如何实现统一的错误处理逻辑
## panic
- 停止当前函数运行
- 一直向上返回，执行每一层的defer
- 如果没有遇见recover，程序退出
## recover
- 仅在defer调用中使用
- 获取panic的值
- 如果无法处理，可重新panic
## error vs panic
- 意料之中的：使用error。如：文件打不开
- 意料之外的：使用panic。如：数组越界
## 错误处理综合实例
- defer + panic + recover
- Type Assertion
- 函数式编程的应用
