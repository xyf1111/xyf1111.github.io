# Resume

<!--more-->
## 个人信息
---
- 姓名：陈进煌
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
性别：男
- 年龄：24
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
手机号：18759882615
- 邮箱：2219316464@qq.com
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
专业：计算机科学与技术
- 应聘岗位：Golang开发工程师
## 工作及教育经历
---
- 厦门美城行动科技有限公司
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
2019-08~至今
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
软件研发部-后端开发
- 厦门青叶软件股份有限公司
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
2019-03~2019-07
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
软件研发部-后端开发
- 厦门通元微智能科技有限公司
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
2018-03~2018~11
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
软件研发部-软件测试
- 三明学院
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
2014-09~2018-07
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
计算机科学与技术
## 专业技能
---
- 熟悉Golang，了解Java等编程语言
- 熟悉MySQL数据库
- 了解缓存系统Redis
- 了解网络编程，TCP/IP协议
- 熟悉Beego框架
- 了解React和Ant Design
## 项目经历
---
1. 厦门美城行动管理系统
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
后端开发
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
2019-08~至今
- 项目介绍：这个项目是公司的后台管理系统。该系统主要是对公司对应的小程序和app提供数据的统计展示和管理等。系统后端是采用Beego框架，数据库使用MySQL，部分功能使用到Redis进行缓存。系统前端采用React框架，大部分使用Ant Design的UI组件。

- 我的职责：
1. 根据实际业务需求，与产品经理确定具体功能及展现方式。
2. 依据具体功能进行数据表的设计。
3. 实现具体功能编写管理系统前端展示代码，与产品进行确认。
4. 根据前端界面确定后台的接口，进行后台代码数据操作代码的编写。
5. 若小程序，app有该功能相关的，为其提供相应的接口。
6. 提交Git，交付测试进行测试。
7. 根据测试提交的bug对代码进行调试修改。
<br/><br/><br/><br/><br/><br/>

- 主要实现功能：
```
考试模块
这个功能实现功能如下：
1、题库管理，在题库中对试卷题目，答案等进行数据操作。
2、试卷管理，编辑试卷名称等具体信息，从题库中筛选题目，从用户列表中筛选用户参与该试卷考试。
3、分数排行榜，对试卷分数进行统计，排行及数据展示。
功能难点：
1、用户小程序上提交试卷后展示考试成绩
2、试卷修改时，用户提交过的试卷内容不变
难点解决：
1、使用redis对答案数据进行缓存，用户提交后根据试卷id获取对应缓存数据，进行对比，统计分数并返回。
2、将用户提交的试卷数据序列化进行保存，展示时再反序列化。

大件垃圾订单模块
这个功能实现功能如下：
1、大件垃圾基本设置，根据不同的类型，对不同情况下价格的定制。
2、大件订单管理，管理人员分配订单给回收人员，回收人员通过小程序了解订单详情并完成。
3、回收人员管理，对回收人员添加身份标识和。
```

