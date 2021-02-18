# Go Study 10

<!--more-->
## 接口
- 接口由使用者定义
## duck typing
- “像鸭子走路，像鸭子叫(长得像鸭子)，那么就是鸭子”
- 描述事物的外部行为而非内部结构
- 严格来说go属于结构化类型系统，类似duck typing
## 接口变量里有什么
!["接口变量1"](/images/interface1.png "接口变量1")
!["接口变量2"](/images/interface2.png "接口变量2")
- 接口变量自带指针
- 接口变量同样采用值传递，几乎不需要使用接口的指针
- 指针接收者实现只能以指针方式使用；值接收者都可以
## 查看接口变量
- 表示任何变量：interface{}
- Type Assertion
- Type Switch
## 特殊接口
- Stringer
- Reader/Writer
