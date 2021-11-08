# Go Docker


## 为什么需要Docker

```md
使用docker的主要目标是容器化。也就是为你的应用程序提供一致的环境，而不依赖它运行的主机
```

想象一下你是否也会遇到下面这个场景，你在本地开发了你的应用程序，它很可能有很多依赖的环境或者包，甚至对依赖的具体版本都有严格要求，当开发过程完成后，你希望将应用程序部署到Web服务器。这个时候你必须确保所有的依赖项都安装正确并且版本也相同，否则程序可能会崩溃并且无法运行。如果你想在另一个web服务器上也部署该应用程序。那么你必须从头开始重复这个过程。这种场景就是docker发回作用的地方。

对于运行我们程序的主机，不管是笔记本电脑还是服务器，我们唯一需要做的就是运行一个docker容器平台。从此以后，你就不需要担心你使用的MacOS，Ubuntu，Arch还是其他，你只需要定义一次应用，即可随时随地运行。

## Docker部署示例

### 准备代码

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", hello)
    server := &http.Server{
        Addr:   ":8888",
    }

    fmt.Println("server startup...")

    if err := server.ListenAndSave(); err != nil {
        fmt.Printf("server startup failed, err: %v\n", err)
    }
}

func hello(w http.ResponseWriter, _ *http.Request) {
    w.Write([]byte("hello cc.com"))
}
```

上面的代码通过`8888`端口对外提供服务，返回一个字段响应：`hello cc.com`

### 创建Docker镜像

```md
镜像(image)包含运行应用程序所需的所有东西--代码或二进制文件、运行时、依赖项以及所需的任何其他文件系统对象。
```

或者简单的说，镜像(image)是定义应用程序及其运行所需的一切

### 编写Dockerfile

要创建Docker镜像(image)必须在配置文件中指定步骤。这个文件我们同城称为`Dockerfile`。(虽然这个文件可以随意命名它，但是最好还是使用默认的`Dockerfile`。)

**注意：某些步骤不是唯一的，可以根据自己的需要修改诸如文件路径，最终可执行文件的名称等**

```md
FROM golang:alpine

<!-- 为我们镜像设置必要的环境变量 -->
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \

<!-- 移动到工作目录 -->
WORKDIR /build

<!-- 将代码复制到容器中 -->
COPY . .

<!-- 将我们的代码编译成二进制可执行文件app -->
RUN go build -o app .

<!-- 移动到用于存放生成二进制文件的 /dist 目录 -->
WORKDIR /dist

<!-- 将二进制文件从 /build 目录复制到这里 -->
RUN cp /build/app .

<!-- 声明服务端口 -->
EXPOSE 8888

<!-- 启动容器时运行的命令 -->
CMD ["/dist/app"]
```

**Dockerfile解析**

*From*

我们正在使用基础镜像`golang:alpine`来创建我们的镜像。这和我们要创建的镜像一样是一个我们能够访问的存储在Docker仓库的基础镜像。这个镜像运行的是alpine Linux发行版，该发行版的大小很小并且内置了Go，非常适合我们的用例。有大量公开可用的Docker镜像。

*Env*

用来设置我们编译阶段需要用的环境变量

*WORKDIR, COPY, RUN*

这几个命令做的事都写在注释里了，很好理解

*EXPORT, CMD*

最后我们声明服务端口，因为我们的应用程序监听的是这个端口并通过这个端口对外提供服务。并且我们还定义了在我们运行镜像的时候默认执行的命令`CMD ["/dist/app"]`

### 构建镜像

在项目目录下，执行下面的命令创建镜像，并指定镜像名称为`goweb_app`:

```bash
docker build . -t goweb_app
```

等待构建过程结束，输出如下提示：

```bash
...
Successfully built 90d9283286b7
Successfully tagged goweb_app:latest
```

现在我们已经准备好了镜像，但是目前它什么也没做。我们接下来要做的就是运行我们的镜像，以便处理我们的请求。运行中的镜像称为容器。

执行下面的命令来运行镜像：

```bash
docker run -p 8888:8888 goweb_app
```

标志位`-p`用来定义端口绑定。由于容器中的应用程序在端口8888上运行，我们将其绑定到主机端口也是8888。如果要绑定到另一个端口，则可以使用`-p $HOST_POST:8888`。例如`-p 5000:8888`

现在就可以测试下我们的web程序是否工作正常，打开浏览器输入`http://127.0.0.1:8888`就能看见实现定义的响应内容

## 分阶段构建示例

我们的Go程序编译之后会得到一个可执行的二进制文件，其实在最终的镜像是不需要go编译器的，也就是说我们只需要一个运行最终二进制文件的容器即可。

Docker的最佳实践之一是通过仅保留二进制文件来减小镜像大小，为此，我们将使用一种多阶段构建的技术，这意味着我们将通过多个步骤构建镜像

```bash
FROM golang:alpine AS builder

# 为我们的镜像设置必要的环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64

# 移动到工作目录：/build
WORKDIR /build

# 将代码复制到容器中
COPY . .

# 将我们的代码编译成二进制可执行文件 app
RUN go build -o app .

###################
# 接下来创建一个小镜像
###################
FROM scratch

# 从builder镜像中把/dist/app 拷贝到当前目录
COPY --from=builder /build/app /

# 需要运行的命令
ENTRYPOINT ["/app"]
```

使用这种技术，我们剥离了使用`golang:alpine`作为编译镜像来编译得到二进制可执行文件的过程，并基于`scratch`生成一个简单的非常小的新镜像。我们将二进制文件重命名为`builder`的第一个镜像中复制到新创建的`scratch`镜像中。

## 附带其他文件的部署示例

如果项目中带有静态文件或配置文件，需要将其拷贝到最终的镜像文件中。

例如bubble项目中用到了静态文件和配置文件，具体目录结构如下：

```go
bubble
├── README.md
├── bubble
├── conf
│   └── config.ini
├── controller
│   └── controller.go
├── dao
│   └── mysql.go
├── example.png
├── go.mod
├── go.sum
├── main.go
├── models
│   └── todo.go
├── routers
│   └── routers.go
├── setting
│   └── setting.go
├── static
│   ├── css
│   │   ├── app.8eeeaf31.css
│   │   └── chunk-vendors.57db8905.css
│   ├── fonts
│   │   ├── element-icons.535877f5.woff
│   │   └── element-icons.732389de.ttf
│   └── js
│       ├── app.007f9690.js
│       └── chunk-vendors.ddcb6f91.js
└── templates
    ├── favicon.ico
    └── index.html
```

我们需要将`templates`、`static`、`conf`三个文件夹中的内容拷贝到最终的镜像文件中，更新后的`Dockerfile`如下

```go
FROM golang:alpine AS builder

# 为我们的镜像设置必要的环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64

# 移动到工作目录：/build
WORKDIR /build

# 复制项目中的 go.mod 和 go.sum文件并下载依赖信息
COPY go.mod .
COPY go.sum .
RUN go mod download

# 将代码复制到容器中
COPY . .

# 将我们的代码编译成二进制可执行文件 bubble
RUN go build -o bubble .

###################
# 接下来创建一个小镜像
###################
FROM scratch

COPY ./templates /templates
COPY ./static /static
COPY ./conf /conf

# 从builder镜像中把/dist/app 拷贝到当前目录
COPY --from=builder /build/bubble /

# 需要运行的命令
ENTRYPOINT ["/bubble", "conf/config.ini"]
```

简单来说就是多了几步COPY的步骤

**TIPS：这里把COPY静态文件的步骤放在上层，把COPY二进制可执行文件放在下层，争取多使用缓存**

### 关联其他容器

因为项目中使用了MySQL，我们可以选择使用如下命令启动一个MySQL容器，它的别名为`mysql8019`；root用户的密码为`root1234`；挂载容器中的`/var/lib/mysql`到本地的`/Users/cc/docker/mysql`目录；内部服务端口为3306，映射到外部的13306端口。

```bash
docker run --name mysql8019 -p 13306:3306 -e MYSQL_ROOT_PASSWORD=root1234 -v /Users/cc/docker/mysql:/var/lib/mysql -d mysql:8.0.19
```

程序中配置的MySQL的host地址为容器的别名，使它们在内部通过的别名(mysql8019)联通

```go
[mysql]
user = root
password = root1234
host = mysql8019
port = 3306
db = bubble
```

修改后重新构建`bubble_app`镜像

```bash
docker build . -t bubble_app
```

我们这里运行`bubble_app`容器的时候需要使用`--link`的方式与上面的`mysql8019`容器关联起来，具体命令如下

```bash
docker run --link=mysql8019:mysql8019 -p 8888:8888 bubble_app
```

## Docker Compose模式

除了像上面一样使用`--link`的方式来关联两个容器之外，还可以使用`Docker Compose`来定义和运行多个容器。

`Compose`是用于定义和运行多容器Docker应用程序的工具。通过Compose，可以使用YML文件来配置应用程序需要的所有服务。然后，用一个命令，就可以从YML文件配置中创建并启动所有服务。

使用Compose基本上是一个三步过程：

- 使用`Dockerfile`定义你的应用环境以便可以在任何地方复制
- 定义组成应用程序的服务，`docker-compose.yml`以便它们可以在隔离的环境中一起运行
- 执行`docker-compose up`命令来启动并运行整个应用程序

我们的项目需要两个容器分别运行`mysql`和`bubble_app`,我们编写的`docker-compose.yml`文件内容如下：

```yml
# yaml 配置
version: "3.7"
services:
  mysql8019:
    image: "mysql:8.0.19"
    ports:
      - "33061:3306"
    command: "--default-authentication-plugin=mysql_native_password --init-file /data/application/init.sql"
    environment:
      MYSQL_ROOT_PASSWORD: "root1234"
      MYSQL_DATABASE: "bubble"
      MYSQL_PASSWORD: "root1234"
    volumes:
      - ./init.sql:/data/application/init.sql
  bubble_app:
    build: .
    command: sh -c "./wait-for.sh mysql8019:3306 -- ./bubble ./conf/config.ini"
    depends_on:
      - mysql8019
    ports:
      - "8888:8888"
```

这个Compose文件定义了两个服务：`bubble_app`和`mysql8019`。其中：

**bubble_app**

使用当前目录下的`Dockerfile`文件构建镜像，并通过`depends_on`指定依赖`mysql8019`服务，声明服务端口8888并绑定对外8888端口

**mysql8019**

mysql8019服务使用Docker Hub的公共mysql:8.0.19镜像，内部端口3306，外部端口33061。

这里需要注意的一个问题，`bubble_app`容器需要等待`mysql8019`容器正常启动之后在尝试启动，因为Web程序在启动的时候会初始化MySQL连接。这里共有两个地方需要更改，第一个就是`Dockerfile`中要把最后一句注释掉。

```bash
# Dockerfile
...
# 需要运行的命令（注释掉这一句，因为需要等MySQL启动之后再启动我们的Web程序）
# ENTRYPOINT ["/bubble", "conf/config.ini"]
```

第二个地方是在`bubble_app`下面添加如下命令，使用提前编写的`wait-for.sh`脚本检测`mysql8019:3306`正常后再执行后续启动Web应用程序的命令

```bash
command: sh -c "./wait-for.sh mysql8019:3306 -- ./bubble ./conf/config.ini"
```

当然，因为现在要在`bubble_app`镜像中执行sh命令，所以不能在使用`scratch`镜像构建了，这里改为使用`debian:stretch-slim`，同时还要安装`wait-for.sh`脚本用到的`netcat`，最后不要忘了把`wait-for.sh`脚本文件COPY到最终的镜像中，并赋予可执行权限哦。更新后的`Dockerfile`内容如下：

```bash
FROM golang:alpine AS builder

# 为我们的镜像设置必要的环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64

# 移动到工作目录：/build
WORKDIR /build

# 复制项目中的 go.mod 和 go.sum文件并下载依赖信息
COPY go.mod .
COPY go.sum .
RUN go mod download

# 将代码复制到容器中
COPY . .

# 将我们的代码编译成二进制可执行文件 bubble
RUN go build -o bubble .

###################
# 接下来创建一个小镜像
###################
FROM debian:stretch-slim

COPY ./wait-for.sh /
COPY ./templates /templates
COPY ./static /static
COPY ./conf /conf


# 从builder镜像中把/dist/app 拷贝到当前目录
COPY --from=builder /build/bubble /

RUN set -eux; \
	apt-get update; \
	apt-get install -y \
		--no-install-recommends \
		netcat; \
        chmod 755 wait-for.sh

# 需要运行的命令
# ENTRYPOINT ["/bubble", "conf/config.ini"]
```

所有条件准备就绪后，就可以执行下面的命令跑起来了

```bash
docker-compose up
```

## 总结

使用Docker容器能够极大简化我们在配置依赖环境方面的操作，但同时也对我们的技术储备提了更高的要求。对于Docker不管你是熟悉抑或是不熟悉，技术发展的车轮都滚滚向前。
