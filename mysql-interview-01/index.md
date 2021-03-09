# Mysql Interview 01

<!--more-->
## drop、delete和truncate分别在什么场景下使用？对比一下区别
### drop table
- 属于DDL
- 不可回滚
- 不可带where
- 表内容结构删除
- 删除速度快
### truncate table
- 属于DDL
- 不可回滚
- 不可带where
- 表内容删除
- 速度快
### delete from
- 属于DML
- 可回滚
- 可带where
- 表结构在，表内容看具体where
- 删除速度慢
### 使用场景
- 不再需要一张表的时候，使用drop
- 想删除部分数据行时候，使用delete，并且带上where子句
- 想保留表结构而删除所有数据时，使用truncate
##



























