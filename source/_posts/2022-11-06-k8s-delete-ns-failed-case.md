---
layout: next
title: k8s删除namespace失败, 陷入Terminating状态解决方法
date: 2022-11-16 20:58:28
categories: k8s
tags: k8s
---

## 先尝试强制删除deployment和pod等资源
```
kubectl -n ${ns} delete deployment --all --force
kubectl -n ${ns} delete pod --all --force
```

## 如果还是失败，通过API强制删除namespace

### 获取需要强制删除的namespace信息
```
kubectl get ns ${ns} -o json > old_ns.json
```
<!-- more -->

### 删除finalizers
```
jq 'del(.spec.finalizers)' old_ns.json > new_ns.json
```

### 运行kube-proxy
```
# kubectl proxy &
Starting to serve on 127.0.0.1:8001
```

### 通过API强制删除namespace
```
curl -k -H "Content-Type: application/json" -X PUT --data-binary @new_ns.json http://127.0.0.1:8001/api/v1/namespaces/${ns}/finalize
```

### 最后关闭kube-proxy, 确认namespace被成功删除

最后给出一个Shell脚本，强制删除Terminating状态的namespace
```bash
#!/bin/bash

# 变量ns表示需要强制删除的namespace
kubectl get ns ${ns} -o json > old_ns.json
jq 'del(.spec.finalizers)' old_ns.json > new_ns.json

kubectl proxy &
KUBECTL_PROXY_PID=$!
curl -k -H "Content-Type: application/json" -X PUT --data-binary @new_ns.json http://127.0.0.1:8001/api/v1/namespaces/${ns}/finalize
kill ${KUBECTL_PROXY_PID}

# 检查namespace是否陷入Terminating状态
function check_if_namespace_is_terminating() {
    status=$(kubectl get ns ${ns} -o json | jq .status.phase -r)
    if [ "$status" = "Terminating" ]; then
        return 1
    fi
    return 0
}
```

## 参考
https://www.iszy.cc/posts/force-delete-k8s-namespace/
