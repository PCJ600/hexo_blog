---
layout: next
title: Nginx中的location配置
date: 2024-11-17 14:18:29
categories: Nginx
tags: Nginx
---

Nginx的location配置，用于定义请求的URI和服务器响应之间的对应关系。

## location语法
Nginx的location规则匹配的是URI, 不需要考虑后面的query_string。语法如下:
```
location [ = | ~ | ~* | ^~ | 空格 ] /URI { 
	... 
}
```

## location modifier的解释
| 参数 | 解释 |
| -- | -- |
| = | 精确匹配, 如匹配成功就立即停止搜索 |
| ^~ | 前缀匹配, 不使用正则表达式。如果匹配成功, 并且匹配字符串是最长的，就不再匹配其他location |
| ~ | 正则匹配，区分大小写|
| ~* | 正则匹配，不区分大小写 |
| 空格 | 前缀匹配, 不使用正则表达式 |

<!-- more -->

## location modifier的匹配顺序 

1. 先精确匹配`=`，如精确匹配成功会立刻停止搜索
2. 再前缀匹配(`^~`和空格), 如果匹配最长的结果是`^~`, 立刻停止搜索; 否则暂存匹配最长的结果(空格)，继续往下走
3. 查找正则匹配(`'~'`和`'~*'`), 如果同时有多个正则匹配，按配置文件中定义的先后顺序，优先取配置文件中最上面的，立刻停止搜索；否则往下走
4. 返回第2步保存的结果(匹配最长的空格匹配)

官方文档的描述: [http://nginx.org/en/docs/http/ngx_http_core_module.html#location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)

以下通过几个示例加深理解

## 示例

### 例1(Nginx官方文档的例子):
```
location = / {
    return 200 '1';
}

location / {
    return 200 '2';
}

location /documents/ {
    return 200 '3';

location ^~ /images/ {
    return 200 '4';
}

location ~* \.(gif|jpg|jpeg)$ {
    return 200 '5';
}
```

测试结果:
```
curl localhost/
'1'
curl localhost/index.html
'2'
curl localhost/documents/document.html
'3'
curl localhost/images/1.gif
'4'
curl localhost/documents/1.jpg
'5'
```
解释: 对于`localhost/documents/1.jpg`, 前缀匹配最长结果为空格匹配`/documents/`, 所以继续正则匹配, 正则匹配命中`location ~* \.(gif|jpg|jpeg)$`，所以返回5

### 例2:
```
location /document {
    return 200 '1';
}
location ~* ^/document$ {
    return 200 '2';
}
```
测试结果
```
curl localhost/document
2
```
解释: 前缀匹配的结果为空格匹配, 所以继续执行正则匹配, 正则匹配可以命中，所以返回2

### 例3
```
location ^~ /doc {
	return 200 '1';
}
location ~ ^/document$ {
    return 200 '2';
}
```
测试结果
```
curl localhost/document
'1'
curl localhost/doc
'1'
```
解释: `^~`前缀匹配命中后，立刻停止搜索，不会进行正则匹配, 所以返回1

### 例4
```
location ~ ^/document {
    return 200 '1';
}
location ~ ^/document2 {
    return 200 '2';
}
```
测试结果
```
curl localhost/document2
1
```
解释: 正则匹配是按照配置文件中定义的顺序，先匹配成功的返回，所以返回1

### 例5
```
location /images/1.jpg {
  return 200  '1';     
}        

location ^~ /images/ { 
  return 200  '2';      
}    

location ~ /images/ {        
  return 200  '3';   
} 

location ~ /images/1.jpg {        
  return 200  '4';   
}
```
测试:
```
curl localhost/images/1.png 
'3'
curl localhost/images/1 
'2'
```
解释: 
* 对于第1个`/images/1.png`，前缀匹配最长的是空格(第1个location), 所以继续正则匹配; 正则匹配可以命中第3个和第4个，取最上面那个，所以返回3
* 对于第2个`/images/1`，最长的前缀匹配为`^~`(第2个location)，所以直接返回2


## location 实际使用
实际使用中，要避免出现上述示例中混乱的配置；可以先放置精确匹配，再前缀匹配，最后是正则匹配
```
server {
    # 精确匹配
    location = / {
        # 配置...
    }

    # 前缀匹配
    location /static/ {
        # 配置...
    }

    # 正则匹配
    location ~* \.(jpg|jpeg|png)$ {
        # 配置...
    }
}
```

## 参考
【1】[https://juejin.cn/post/6908623305129852942](https://juejin.cn/post/6908623305129852942)
【2】[https://juejin.cn/post/7048952689601806366](https://juejin.cn/post/7048952689601806366)
