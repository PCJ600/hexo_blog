---
layout: next
title: RESUME(draft)
date: 2025-02-10 23:07:09
categories: interview
tags: interview
---

# 潘闯

## 个人信息
* 联 系 方 式: 13951722618 | nju_pc@163.com | 南京市浦口区
* 个 人 博 客: https://pcj600.github.io (170篇，20万浏览)
* 期 望 职 位: XXX工程师

## 工作及教育经历
* 趋势科技(中国)有限公司&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2022.4-至今&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Vision One Spark Team 
* 华为技术有限公司&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2019.8-2022.4&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;设备管理开发部            
* 南京大学&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2015.9-2019.6&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;计算机科学与技术专业-本科 

<!-- more -->

## 专业技能
* 熟悉C和Python, 了解C++, Shell等编程语言
* 熟悉Docker和Kubernetes, 有相关开发经验
* 了解并使用过MySQL, 了解Redis单机数据库的实现
* 了解并使用过RabbitMQ, AWS Iot Core等消息队列
* 了解RESTful API, 用Django开发经验
* 熟练使用GIT, GDB, Makefile等开发工具

## 项目经验
趋势科技(中国)有限公司 - Trend Vision One™ Service Gateway - 开发人员 - 2022.4-至今 <br/>
* 技术: Linux启动流程, Python, kubernetes, MQ, Nginx, rest API, MySQL, Squid 
* [项目简介] A Service Gateway consists of a cloud-based inventory list in the console app and a locally-based virtual appliance. A Service Gateway installed in the local network acts as a relay between Trend Vision One and other products, such as on-premises Trend Micro or third-party products. This allows use of Trend Micro cloud services while reducing internet traffic and sharing threat intelligence.
* 设计了一个双系统分区的固件在线升级方案，支持系统故障后回滚到上个可用版本, 显著改善了客户使用体验
* 设计了一个微服务的在线升级方案, 将微服务与固件解耦，提高了开发效率
* 引入消息队列AWS Iot Core, 实现VA和后端的解耦, 提高了后端响应速度和可靠性
* 基于CLISH开发了一个命令行, 允许用户配置和管理VA。设计了一组系统诊断的命令, 用于提高问题定位效率
* 基于Squid实现了一个正向代理的微服务, 实现认证，访问控制, 缓存功能。设计了stunnel加密方案，解决客户防火墙异常问题

华为技术有限公司 - Matrix Simulation Platform - 开发人员 - 2019.08-2022.04 <br/>
* 技术：C, Docker, Redis
* [项目简介] 使用Docker和Redis实现了一个交换机的仿真平台，让开发人员在没有物理设备的情况下进行代码测试
* 基于Hiredis库开发了一组读写Redis键的API，用于驱动接口的仿真。 实现不修改驱动代码的情况下进行故障注入，提高测试效率。
* 利用Redis的键空间通知机制，实现了一组发布订阅的API，实现单板插拔场景的仿真
* 把Redis短连接优化为长连接, 使驱动接口的读写性能提升了15倍
* 通过XML建模，将新增一个仿真单板的工作量从1人天缩短到1人时，提升测试效率