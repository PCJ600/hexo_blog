---
layout: next
title: 使用 Docker Compose 在单节点部署多容器
date: 2025-01-04 22:54:37
categories: Docker
tags: Docker
---

# Docker Compose 是什么
Docker Compose 是一个用于运行多容器应用的工具, 通过一个`docker-compose.yml`文件, 配置应用的服务、网络和卷，然后使用简单的命令启动或停止所有服务

# 为什么需要 Docker Compose 
当你有一个包含多个相互依赖的容器应用时，手动管理每个服务的启动、停止以及配置会比较复杂且容易出错
Docker Compose 提供了一种更简便的方法, 在单节点部署多个容器, 使得开发测试环境的一致性得到保证，同时简化了部署过程

# 示例: 使用Docker Compose 部署一个WordPress站点

下面演示如何使用 Docker Compose 在单个节点上部署一个包含 WordPress 和 MySQL 的简单网站。

## 1. 安装Docker Compose
官网上查找匹配版本安装, 我直接取最新版本安装
```
curl -L "https://github.com/docker/compose/releases/download/v2.34.0/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose version
```

<!-- more -->

## 2. 创建`docker-compose.yml`文件
在你的项目目录下创建一个名为 docker-compose.yml 的文件，并添加如下内容
```
version: '3.8'

services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: example
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    volumes:
      - db_data:/var/lib/mysql
    restart: always

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    restart: always

volumes:
  db_data:
  wordpress_data:
```
这个yaml文件定义了两个服务：一个是 MySQL 数据库，另一个是 WordPress 应用程序, 同时我们定义了两个存储卷来持久化数据

## 3. 启动服务
在 docker-compose.yml 文件所在的目录中执行以下命令以启动服务
```
docker-compose up -d
[+] Running 3/3
 ✔ Network docker-compose_default        Created                                                                                                                                                                             0.1s
 ✔ Container docker-compose-db-1         Started                                                                                                                                                                             0.3s
 ✔ Container docker-compose-wordpress-1  Started
```
访问 `http://localhost:8080` 查看 WordPress 网站


## 4. 停止服务
使用`docker-compose down`命令, 即可一键停止服务
```
docker-compose down
[+] Running 3/3
 ✔ Container docker-compose-wordpress-1  Removed                                                                                                                                                                             1.2s
 ✔ Container docker-compose-db-1         Removed                                                                                                                                                                             1.7s
 ✔ Network docker-compose_default        Removed
```

# Docker Compose 的优点

通过上述示例可以看出，Docker Compose 极大地简化了多容器应用的部署流程, 优点如下:
| 特性 | 直接使用Docker | 使用Docker Compose |
| :--- | :-------------- | :----------------- |
| 配置管理 | 每个容器的配置分散在多个命令中，容易出错且难以维护 | 配置集中在docker-compose.yml文件，便于维护和分享 |
| 一键操作 | 需要分别运行多条命令来启动、停止容器，效率低 | 使用一条命令(如 `docker-compose up` 或 `docker-compose down`) 即可完成整个应用的管理 |
| 依赖管理 | 需要手动指定容器之间的依赖关系(如--link)，容易出错 | 使用depends_on处理依赖关系 |
| 存储卷操作 | 需要手动创建和挂载存储卷 | 在 docker-compose.yml 中定义存储卷，自动管理持久化数据 |

# 参考
[https://juejin.cn/post/7042663735156015140](https://juejin.cn/post/7042663735156015140)