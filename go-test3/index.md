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
