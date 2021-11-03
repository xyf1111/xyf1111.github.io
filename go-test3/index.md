# Go测试从零到溜3-MySQL和Redis测试


## go-sqlmock

sqlmock是一个实现`sql/driver`的mock库。它不需要建立真正的数据库连接就可以在测试中模拟任何sql驱动程序的行为。使用它可以很方便的在编写单元测试的时候mock sql语句的执行结果。

### 安装

```bash
go get github.com/DATA-DOG/go-sqlmock
```

### 使用示例

这里使用的是`go-sqlmock`官方文档中提供的基础示例代码。在下面的代码中，我们实现一个`recordStats`函数用来记录用户浏览商品时产生的相关数据。具体实现的功能是在一个事务中进行以下两个SQL操作：

- 在`products`表中将当前商品的浏览次数+1
- 在`product_viewers`表中记录浏览当前商品的用户id

```go
// app.go
package main

import "database/sql"

// recordStats 记录用户浏览产品信息
func recordStatus(db *sql.DB, userID, productID int64) (err error) {
    // 开启事务
    // 操作views和priduct_viewers两张表
    tx, err := db.Begin()
    if err != nil {
        return
    }

    defer func() {
        switch err {
        case nil :
            err = tx.Commit()
        default:
            tx.Rollback()
        }
    }()

    // 更新products表
    if _, err = tx.Exec("UPDATE products SET views = views + 1"); err != nil {
        return
    }

    // product_viewers表中插入一条数据
    if _, err = tx.Exec("INSERT INTO product_viewers(user_id, product_id) VALUES (?, ?)", userID, productID); err != nil {
        return
    }

    return
}

func main() {
    // 注意：测试中并不需要真正的连接
    db, err := sql.Open("mysql", "root@/blog")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    // UserID为1的用户浏览了productID为5的产品
    if err = recordStats(db, 1 /*some user id*/, 5 /*some product id*/); err != nil {
        panic(err)
    }
}
```

现在我们需要为代码中的`recordStats`函数编写单元测试，但是又不想在测试过程中连接真实数据库进行测试。这个时候我们就可以像下面示例代码中那样，使用`sqlmock`工具去mock数据库操作。

```go
package main

import (
    "fmt"
    "testing"

    "github.com/DATA-DOG/go-sqlmock"
)

// TestShouldUpdateStats sql执行成功的测试用例
func TestShouldUpdateStats(t *testing.T) {
    // mock一个*sql.DB对象，不需要连接真实的数据库
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatalf("an error '%s' was not expected when opening a stub database connection", err)
    }
    defer db.Close()

    // mock执行指定SQL语句时的返回结果
    mock.ExpectBegin()
    mock.ExpectExec("UPDATE products").WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectExec("INSERT INTO product_viewers").WillArgs(2, 3).WithReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectCommit()

    // 将mock的DB对象传入我们的函数中
    if err = recordStats(db, 2, 3); err != nil {
        t.Errorf("error was not expected while updating stats: %s", err)
    }

    // 确保期望的结果都满足
    if err = mock.ExpectationsWereMet(); err != nil {
        t.Errorf("there are unfulfilled expectations: %s", err)
    }
}

// TestShouldRollbackStatUpdateOnFailure sql执行失败回滚的测试用例
func TestShouldRollbackStatUpdateOnFailure(t *testing.T) {
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatalf("an error '%s' was not expected when opening a stub database connection", err)
    }
    defer db.Close()

    mock.ExpectBegin()
    mock.ExpectExec("UPDATE products").WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectExec("INSERT INTO product_viewers").WillArgs(2, 3).WillReturnError(fmt.Errorf("some error"))
    mock.ExpectRollback()

    // now we execute our method
    if err = recordStats(db, 2, 3); err != nil {
        t.Errorf("was expecting an error, but there was none")
    }

    // we make sure that all expectations were met(我们确保所有的期望 都满足)
    if err = mock.ExpectationsWereMet(); err != nil {
        t.Errorf("there were unfulfilled expectations: %s", err)
    }
}
```

上面的代码中，定义了一个执行成功的测试用例和一个执行失败回滚的测试用例，确保我们代码中的每个逻辑分支都能被测试到，提高单元测试覆盖率的同时也保证了代码的健壮性。

## miniredis

除了经常用到MySQL外，Redis在日常开发中也会经常用到。

miniredis是一个纯go实现的用于单元测试的redis server。它是一个简单易用的，基于内存的redis替代品，它具有真正的TCP接口，你可以把它当成redis版本的`net/http/httptest`

当我们为一些包含Redis操作的代码编写单元测试时就可以用它来mock Redis操作

### 安装

```bash
go get github.com/alicebob/miniredis/v2
```

### 使用示例

这里以`github.com/go-redis/redis`库为例，编写了一个包含若干个Redis操作的`DoSomethingWithRedis`函数

```go
// redis_op.go
package miniredis_demo

import (
    "context"
    "strings"
    "time"

    "github.com/go-redis/redis/v8"
)

const (
    KeyValidWebsite = "app:valid:website:list"
)

func DoSomethingWithRedis(rdb *redis.Client, key string) bool {
    // 这里可以是对Redis操作的一些逻辑
    ctx := context.TODO()
    if !rdb.SisMember(ctx, KeyValidWebsite, key).Val() {
        return false
    }

    val, err := rdb.Get(ctx, key).Result()
    if err != nil {
        return false
    }

    if !strings.HasPrefix(val, "https://") {
        val = "https://" + val
    }

    // 设置blog key五秒过期
    if err = rdb.Set(ctx, "blog", val, 5 * time.Second).Err(); err != nil {
        return false
    }

    return true
}
```

下面代码使用`miniredis`库为`DoSomethingWithRedis`函数编写单元测试代码，其中`miniredis`不仅支持mock常用的Redis操作，还提供了很多实用的帮助函数，例如检查key的值是否与预期相等的`s.CheckGet()`和帮助检查key过期时间的`s.FastForward()`

```go
// redis_op_test.go

package miniredis_demo

import (
    "testing"
    "time"

    "github.com/alicebob/miniredis/v2"
    "github.com/go-redis/redis/v8"
)

func TestDoSomethingWithRedis(t *testing.T) {
    // mock 一个redis server
    s, err := minireids.Run()
    if err != nil {
        panic(err)
    }
    defer s.Close()

    // 准备数据
    s.Set("cc", "cc.com")
    s.SAdd(KeyValidWebsite, "cc")

    // 连接mock的redis server
    rdb := redis.NewClient(&redis.Options{
        // mock redis server 地址
        Addr: s.Addr(),
    })

    // 调用函数
    ok := DoSomethingWithRedis(rdb, "cc")
    if !ok {
        t.Fatal()
    }

    // 可以手动检查redis中的值是否符合预期
    if got, err := s.Get("blog"); err != nil || got != "https://cc.com" {
        t.Fatalf("'blog' has the wrong value")
    }

    // 也可以使用帮助工具检查
    s.CheckGet(t, "blog", "https://cc.com")

    // 过期检查
    s.FastForward(5 * time.Second)  // 快进5秒
    if s.Exists("blog") {
        t.Fatal("'blog' should not have existed anymore")
    }
}
```

## 总结

在日常工作开发中为代码编写单元测试时如何处理数据库依赖是最常见的问题，本文介绍了如何使用`go-sqlmock`和`miniredis`工具mock相关依赖。
