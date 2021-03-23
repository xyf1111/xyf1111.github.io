# Go Instance 17

<!--more-->
## 时间
Go对时间和时间段提供了大量的支持
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	p := fmt.Println

	//获得当前时间
	now := time.Now()
	p(now)

	//提供年月日等信息，可以构建一个time。时间总是关联位置信息，例如时区
	then := time.Date(2009, 11, 17, 20, 34, 58, 651387237, time.UTC)
	p(then)

	//提取出时间的各个组成部分
	p(then.Year())
	p(then.Month())
	p(then.Day())
	p(then.Hour())
	p(then.Minute())
	p(then.Second())
	p(then.Nanosecond())
	p(then.Location())

	//输出时间的星期
	p(then.Weekday())

	//使用方法比较时间，精确到秒
	p(then.Before(now))
	p(then.After(now))
	p(then.Equal(now))

	//Sub返回一个duration来表示两个时间点的间隔时间
	diff := now.Sub(then)
	p(diff)

	//计算不同单位下的时间长度值
	p(diff.Hours())
	p(diff.Minutes())
	p(diff.Seconds())
	p(diff.Nanoseconds())

	//可以使用Add将时间后移一个时间间隔，或在时间间隔前加-，将时间前移一个时间间隔
	p(then.Add(diff))
	p(then.Add(-diff))
}
```
## 时间戳
一般程序都有获取Unix时间的秒数，毫秒数或微秒数的需要
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	//分别使用带Unix或者UnixNano的time.Now来获取从自协调世界时起到现在的秒数或纳秒数
	now := time.Now()
	secs := now.Unix()
	nanos := now.UnixNano()
	fmt.Println(now)

	//UnixMillis是不存在的，所以要得到毫秒数，得自己转化
	millis := nanos/1000000
	fmt.Println(secs)
	fmt.Println(millis)
	fmt.Println(nanos)

	//也可以将秒数或毫秒数转化为时间
	fmt.Println(time.Unix(secs, 0))
	fmt.Println(time.Unix(0, nanos))
}
```
## 时间格式化和解析
Go支持通过基于描述模板的时间格式化和解析
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	p := fmt.Println
	now := time.Now()
	//按照RFC3339格式化
	p(now.Format(time.RFC3339))

	//时间解析使用Format相同的形式值
	t1, _ := time.Parse(time.RFC3339, "2021-03-23T11:01:37+08:00")
	p(t1)

	//Format和Parse使用基于例子的形式来决定日期格式
	//一般只需要使用time包内的格式，但是可以实现自定义模式
	// 模式必须使用时间 Mon Jan 2 15:04:05 MST 2006来指定给定时间/字符串的格式化/解析方式。
	// 时间一定要按照如下所示：2006为年，15 为小时，Monday 代表星期几，等等。
	p(now.Format("03:04pm"))
	p(now.Format("Mon Jan _2 15:04:05 2006"))
	p(now.Format("2006-01-02T15:04:05.999999-07:00"))
	format := "3 04pm"
	t2, _ := time.Parse(format, "8 41pm")
	p(t2)

	//对于纯数字表示的时间，可以使用标准的格式化字符串来提取时间值的组成
	fmt.Printf("%d-%02d-%02dT%02d:%02d:%02d-00:00\n", now.Year(), now.Month(), now.Day(), now.Hour(), now.Minute(), now.Second())

	//Parse函数在输入时间格式不正确会返回错误
	ansic := "Mon Jan _2 15:04:05 2006"
	_, e := time.Parse(ansic, "8:41PM")
	fmt.Println(e)
}
```
