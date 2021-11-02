# Go测试从零到溜2-网络测试


## httptest

在Web开发场景下的单元测试，如果涉及到HTTP请求推荐大家使用Go标准库`net/http/httptest`进行测试，能够显著提升效率。

在这一小节，我们以常见的gin框架为例，演示如何为http server编写单元测试

假设我们的业务逻辑是搭建一个http server端，对外提供http服务。我们编写一个`helloHandle`函数，用来处理用户请求。

```go
// gin.go
package httptest_demo

import (
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"
)

// Param 请求参数
type Param struct {
    Name    string  `json:"name"`
}

// helloHandle /hello请求处理函数
func helloHandler(c *gin.Context) {
    var p Param

    if err := c.ShouldBindJSON(&p); err != nil {
        c.JSON(http.StatusOK, gin.H{
            "msg": "we need a name",
        })
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "msg": fmt.Sprintf("hello %s", p.Name)
    })
}

// 路由
func SetupRouter() *gin.Engine {
    router := gin.Default()
    router.POST("/hello", helloHandler)
    return router
}
```

现在我们需要为`helloHander`函数编写单元测试，这种情况下我们就可以使用`httptest`这个工具mock一个HTTP请求和响应记录器，让我们的server端接收并处理我们mock的HTTP请求，同时使用响应记录器来记录server端返回的响应内容。

单元测试的示例代码如下：

```go
// gin_test.go
package httptest_demo

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/stretchr/testify/assert"
)

func Test_helloHandler(t *testing.T) {
    // 定义两个测试用例
    tests := []struct{
        name    string
        param   string
        expect  string
    } {
        {"base case", `{"name": "cc"}`, "hello cc"},
        {"bad case", "", "we need a name"},
    }

    r := SetupRouter()

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // mock一个http请求
            req := httptest.NewRequest(
                // 请求方法
                "POST",
                // 请求URL
                "/hello",
                // 请求参数
                strings.NewReader(tt.Param),
            )

            // mock一个响应记录器
            w := httptest.NewRecorder()

            // 让server端处理mock请求并记录返回的响应内容
            r.ServeHttp(w, req)

            // 校验状态码是否符合预期
            assert.Equal(t, http.StatusOK, w.Code)

            // 解析并检验响应内容是否符合预期
            var resp map[string]string
            err := json.Unmarshal([]byte(w.Body.String()), &resp)
            assert.Nil(t, err)
            assert.Equal(t, tt.expect, resp["msg"])
        })
    }
}
```

通过这个示例我们就掌握了如何使用httptest在HTTP Server服务中为请求处理函数编写单元测试了。

## gock

上面的示例介绍了如何在HTTP Server服务类场景下为请求处理函数编写单元测试，那么如果我们是在代码中请求外部API的场景(比如通过API调用其他服务获取返回值)又该怎么编写单元测试呢？

例如，我们有以下业务逻辑代码，依赖外部API：`http://your-api.com/post`提供的数据。

```go
// api.go

// ReqParam API请求参数
type ReqParam struct {
    X   int `json:"x"`
}

// Result API返回结果
type Result struct {
    Value   int `json:"value"`
}

func GetResultByAPI(x, y int) int {
    p := &ReqParam{X: x}
    b, _ := json.Marshal(p)

    // 调用其他服务的API
    resp, err := http.Post(
        "http://your-api.com/post",
        "application/json",
        bytes.NewBuffer(b),
    )
    if err != nil {
        return -1
    }

    body, _ := ioutil.ReadAll(resp.Body)
    var ret Result
    if err := json.Unmarshal(body, &ret); err != nil {
        return -1
    }

    // 这里是对API返回的数据做一些逻辑处理
    return ret.Value + y
}
```

在对类似上述这类业务代码编写单元测试的时候，如果不想在测试过程中真正去发送请求或者依赖的外部接口还没开发完成时，我们可以在单元测试中对依赖的API进行mock。

### 安装

```bash
go get -u gopkg.in/h2non/gock.v1
```

### 使用示例

使用`gock`对外部API进行mock，即mock指定参数返回约定好的响应内容。下面的代码中mock了两组数据，组成了两个测试用例

```go
// api_test.go
package gock_demo

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "gopkg.in/h2non/gock.v1"
)

func TestGetResultByAPI(t *testing.T) {
    // 测试执行结束后刷新挂起的mock
    defer gock.Off()

    // mock请求外部api时传参x=1返回100
    gock.New("http://your-api.com").
        Post("/post").
        MatchType("json").
        JSON(map[string]int{"x": 1}).
        Reply(200).
        JSON(map[string]int{"value": 100})

    // 调用我们的业务函数
    res := GetResultByAPI(1, 1)
    // 校验返回结果是否符合预期
    assert.Equal(t, res, 101)

    // mock请求外部API时传参x=2返回200
    gock.New("http://your-api.com").
        Post("/post").
        MatchType("json").
        JSON(map[string]int{"x": 2}).
        Reply(200).
        JSON(map[string]int{"value": 200})

    // 调用我们的业务函数
    res = GetResultByAPI(2, 2)
    // 校验返回结果是否符合预期
    assert.Equal(t, res, 202)

    // 断言mock被触发
    assert.True(t, gock.IsDone())
}
```

测试结果和预期完全一致

## 总结

作为日常开发中为代码编写单元测试时如何处理外部依赖是最常见的问题，本文介绍了如何使用`httptest`和`gock`工具mock相关依赖。
