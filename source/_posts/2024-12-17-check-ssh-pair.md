---
layout: next
title: 判断SSH的公钥和私钥是否为一对的方法
date: 2024-12-17 23:30:35
categories: SSH
tags: SSH
---

## 使用sshkey-gen命令判断
使用ssh-keygen -y命令可以从私钥文件中提取公钥。例如: 你的私钥文件是id_rsa，执行如下命令:
```
ssh-keygen -y -f id_rsa > extracted_pubkey.pub
```
将提取的公钥保存到extracted_pubkey.pub文件。再和你的公钥文件比较，看文件内容是否相同
```
diff extracted_pubkey.pub id_rsa.pub
```
