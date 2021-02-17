# Go Study 09

<!--more-->
## GOPATH环境变量
- 默认在~/go(unix，linux)，%USERPOFILE%\go(windows)
- 官方推荐：所有项目和第三方库都放在同一个GOPATH下
- 也可以将每个项目放在不同的GOPATH
## go get 获取第三方库
- go get + 包(github可以，golang不行)
- 使用gopm来获取无法下载的包
    - go get -v github.com/gpmgo/gopm
## GOPATH下目录结构
- go build来编译
- go install 产生pkg文件和可执行文件
- go run 直接编译运行
## GOPATH下目录结构
- src
    - git repository 1
    - git repository 2
- pkg
    - git repository 1
    - git repository 2
- bin
    - 执行文件1,2,3...
