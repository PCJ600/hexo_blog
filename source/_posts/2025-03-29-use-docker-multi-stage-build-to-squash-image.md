---
layout: next
title: 使用Docker多阶段构建, 压缩镜像体积
date: 2025-03-29 20:41:06
categories: Docker
tags: Docker
---


# 什么是 Docker 多阶段构建
Docker 多阶段构建是一种通过在单个 Dockerfile 中定义多个构建阶段的技术。 每个阶段可以使用不同的基础镜像和依赖项，允许开发人员在构建过程中分离应用程序的编译环境与运行环境。 最终生成的镜像只包含运行应用程序所需的文件，而无需携带编译工具链或其他不必要的依赖
通过多阶段构建，可以有效减少镜像体积，提升部署效率

# 实例: 多阶段构建压缩 Docker 镜像

下面通过一个简单的 Go 程序，分别用传统方式和多阶段构建两种方法演示如何构建 Docker 镜像，并观察多阶段构建对镜像大小的显著优化效果

## 准备工作
* 安装 Docker
* 编写一个简单的 Go 程序 hello.go

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

<!-- more -->

## 传统构建方法
Dockerfile如下所示：
```
# 第一阶段：构建环境
FROM rockylinux:9.3

# 安装Go环境
RUN dnf install -y golang git

WORKDIR /app

# 将本地代码复制到容器中
COPY . .

# 初始化Go模块
RUN go mod init example.com/hello

# 编译Go程序，静态链接
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o hello .

# 指定启动命令
CMD ["./hello"]
```
缺点：包含了完整的编译工具链（如 Go 编译器）以及操作系统的基础包，导致镜像体积庞大

构建与运行
```
docker build -t hello:1.0 .
docker run -it --rm hello:1.0
Hello, World!
```
查看镜像大小
```
# docker images
REPOSITORY        TAG        IMAGE ID       CREATED          SIZE
hello             1.0        3a1caaa45859   10 minutes ago   671MB
```

## 多阶段构建方法
为了减小镜像体积，我们采用多阶段构建技术，将编译环境与运行环境分离
```
# 第一阶段：构建环境
FROM rockylinux:9.3 AS builder

# 安装Go环境
RUN dnf install -y golang git

WORKDIR /app

# 将本地代码复制到容器中
COPY . .

# 初始化Go模块
RUN go mod init example.com/hello

# 编译Go程序，静态链接
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o hello .

# 第二阶段：运行环境
FROM rockylinux:9.3 AS runner

# 从构建阶段复制可执行文件
COPY --from=builder /app/hello .

# 指定启动命令
CMD ["./hello"]
```

构建与运行
```
docker build -t hello-multi-stage:1.0 .
docker run -it --rm hello-multi-stage:1.0
```
查看镜像大小
```
# docker images
REPOSITORY                                                          TAG        IMAGE ID       CREATED          SIZE
hello                                                               1.0        3a1caaa45859   12 minutes ago   671MB
hello-multi-stage                                                   1.0        1579543c9b16   15 minutes ago   177MB
```
效果：镜像体积从 671MB 压缩到 177MB


## 进一步优化：使用 scratch 镜像

scratch 是 Docker 提供的一个空镜像，没有任何操作系统层或预装软件。它非常适合用于以下场景：
* 静态编译的应用程序：如 Go 或 C/C++ 编写的程序，编译后生成的可执行文件不需要动态链接库支持
* 最小化攻击面：由于没有操作系统层，几乎不存在因系统漏洞被攻击的风险
* 快速部署：极小的镜像体积意味着更快的上传、下载和启动速度

Dockerfile如下:
```
FROM rockylinux:9.3 AS builder
RUN dnf install -y golang git
WORKDIR /app
COPY . .
RUN go mod init example.com/hello
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o hello .

# 第二阶段：运行环境, 基础镜像改为 scratch
FROM scratch
COPY --from=builder /app/hello .
CMD ["./hello"]
```
构建与运行
```
docker build -t hello-tiny:1.0 .
docker run -it --rm hello-tiny:1.0
```
查看镜像大小
```
docker images
REPOSITORY              TAG        IMAGE ID       CREATED          SIZE
hello-tiny              1.0        40fd75a4f0ee   10 minutes ago   1.89MB
hello                   1.0        3a1caaa45859   12 minutes ago   671MB
hello-multi-stage       1.0        1579543c9b16   15 minutes ago   177MB
```
效果: 镜像从177MB压缩到1.89M


构建镜像
```
docker build -t hello-tiny:1.0 .
```
运行
```
# docker run -it --rm hello-tiny:1.0
Hello, World!
```
查看镜像大小
```
# docker images
REPOSITORY                                                          TAG        IMAGE ID       CREATED          SIZE
hello-tiny                                                          1.0        40fd75a4f0ee   10 minutes ago   1.89MB
hello                                                               1.0        3a1caaa45859   12 minutes ago   671MB
hello-multi-stage                                                   1.0        1579543c9b16   15 minutes ago   177MB
```


# 其他几种常见的压缩 Docker 镜像的方法

1.选择合适的基础镜像
* Alpine Linux：一个轻量级的Linux发行版，通常比其他基础镜像（如Ubuntu或Debian）要小得多。
* Scratch：一个完全空白的基础镜像，适合静态编译的应用程序，如Go语言编写的应用程序。
* 选择更小版本的基础镜像：例如，使用 python:3.8-slim 而不是 python:3.8。

2.删除不必要的文件和依赖
* 在 Dockerfile 中执行清理操作，如移除安装包缓存或不需要的开发工具, 例如`RUN apt-get clean && rm -rf /var/lib/apt/lists/*

3.合并命令以减少层数
* 每个 RUN 命令都会创建一个新的层。通过合并相关的命令到一个 RUN 指令中，可以减少最终镜像的层数，从而缩小镜像大小。
```
RUN apt-get update && apt-get install -y \
    package-one \
    package-two \
    && rm -rf /var/lib/apt/lists/*
```

4.使用 .dockerignore 文件
* 类似于 .gitignore，.dockerignore 文件可以指定哪些文件和目录不应包含在构建上下文中，从而避免不必要的文件被添加到镜像中。

5.利用 `Docker Squash`等外部工具优化, 将镜像中的多层合并为一层，减少最终镜像的大小


# 参考
[Dockerfile 多阶段构建](https://yeasy.gitbook.io/docker_practice/image/multistage-builds)