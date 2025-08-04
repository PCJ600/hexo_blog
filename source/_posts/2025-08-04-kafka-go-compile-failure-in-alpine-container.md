---
layout: next
title: 解决 Alpine 容器中编译 confluent-kafka-go 报错的问题
date: 2025-08-04 18:40:30
categories: Kafka
tags: Kafka
---

## 问题描述
在使用 Kafka 官方的 Golang SDK (confluent-kafka-go) 时，在 Alpine 容器中编译会报错，而在宿主机上编译没有这个问题
详细问题描述参考：[官方 Github issue](https://github.com/confluentinc/confluent-kafka-go/issues/479)

## 问题原因
* confluent-kafka-go 依赖C语言库 (librdkafka), 编译时需指定`CGO_ENABLED=1`
* Alpine Linux使用musl libc而不是glibc, `go build`编译时需指定`-tags musl`

## 解决方案
Dockerfile里指定编译依赖, 编译时指定`CGO_ENABLED=1`和`-tags musl`, Dockerfile示例如下：
<!-- more -->
```
FROM swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/golang:1.24.5-alpine3.22 AS builder

# 指定编译依赖
RUN apk add --no-cache git build-base

ENV GOPROXY=https://goproxy.cn,direct \
    GO111MODULE=on

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# 编译时指定CGO_ENABLED=1和-tags musl
RUN CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -tags musl -ldflags="-s -w" -o /app/your-app ./cmd/main.go

FROM swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/alpine:3.22.0

COPY --from=builder /app/cloud-agent /usr/local/bin/

EXPOSE 8080

CMD ["your-app"]

```

## 参考
[https://github.com/confluentinc/confluent-kafka-go/issues/479](https://github.com/confluentinc/confluent-kafka-go/issues/479)