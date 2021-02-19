# Go Instance 01

<!--more-->
## 使用Go爬取豆瓣电影排行榜
```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"regexp"
	"strings"
)

func main() {
	data, err := GetHtml("https://movie.douban.com/chart")
	if err != nil {
		fmt.Println("获取源代码失败:", err)
		return
	}
	//创建文件
	file, err := os.Create("movie.json")
	if err != nil {
		fmt.Println("创建文件失败:", err)
		return
	}

	encoder := json.NewEncoder(file)
	encoder.SetIndent(" ", "  ")
	encoder.Encode(GetItem(data))
}

func GetHtml(url string) ([]byte, error) {
	var clent http.Client
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		fmt.Println("创建请求失败:", err)
		return nil, err
	}
	//添加请求头，才能访问到需要的网页源码
	req.Header.Add("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36 Edg/85.0.564.51")

	resp, err := clent.Do(req)
	if err != nil {
		fmt.Println("请求失败:", err)
		return nil, err
	}

	data, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("读取失败:", err)
		return nil, err
	}

	defer resp.Body.Close()
	return data, nil
}

type Item struct {
	Link	string	`json:"link"`
	Name	string	`json:"name"`
	Info	string	`json:"info"`
	Rate	string	`json:"rate"`
	RateNum	string	`json:"rate_num"`
}

func GetItem(data []byte) []Item {
	//正则匹配需要的内容
	pattern := regexp.MustCompile(`(?s)<div.*?class="pl2".*?>.*?<a href="(.*?)".*?>(.*?)/.*?<span.*?</span>.*?<p class="pl">(.*?)</p>.*?<div.*?>.*?<span class="rating_nums">(.*?)</span>.*?<span class="pl">\((.*?)\)</span>.*?</div>`)
	//查找网页中所有的匹配项
	items := pattern.FindAllSubmatch(data, -1)
	var res []Item

	for _, item := range items {
		res = append(res, Item{
			Link:		string(item[1]),
			Name:		strings.TrimSpace(string(item[2])),
			Info:		string(item[3]),
			Rate:		string(item[4]),
			RateNum:	string(item[5]),
		})
	}
	return res
}
```
