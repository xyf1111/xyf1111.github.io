# Go Interview 04

<!--more-->
## Go中的beego框架
1. beego是一个golang实现的轻量级HTTP框架
2. beego可以通过注释路由、正则路由等多种方式完成url路由注入
3. 可以使用bee new工具生成空工程，然后使用bee run命令自动热编译
## Go中的goconvey框架
1. goconvey是一个支持golang的单元测试框架
2. goconvey能够自动监控文件并启动测试，并可以将测试结果实时输出到web界面
3. goconvey提供了非常丰富的断言简化测试用例的编写
## Go中，GoStub的作用是什么
1. GoStub可以对全局变量打桩
2. GoStub可以对函数打桩
3. GoStub不可以对类的成员方法打桩
4. GoStub可以打动态桩，例如对一个函数打桩后，多次调用该函数会有不同的行为
