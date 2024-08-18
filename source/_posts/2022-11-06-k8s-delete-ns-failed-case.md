---
layout: next
title: k8s删除namespace失败, 陷入Terminating状态解决方法
date: 2022-11-16 20:58:28
categories: k8s
tags: k8s
---

## 解决方法
强制删除deployment和pod后，问题解决：
```bash
kubectl -n ${ns} delete deployment --all --force
kubectl -n ${ns} delete pod --all --force
```
判断namespace的状态是否为Terminating：
```bash
function check_if_namespace_is_terminating() {
    status=$(kubectl get ns ${ns} -o json | jq .status.phase -r)
    if [ "$status" = "Terminating" ]; then
        return 1
    fi
    return 0
}
```
<!-- more -->

其他解决方法参考：
* [A namespace is stuck in the Terminating state](https://www.ibm.com/docs/en/cloud-private/3.2.0?topic=console-namespace-is-stuck-in-terminating-state)
* [移除該死的Terminating Namespace](https://medium.com/%E8%BC%95%E9%AC%86%E5%B0%8F%E5%93%81-pks%E8%88%87k8s%E7%9A%84%E9%BB%9E%E6%BB%B4/%E7%A7%BB%E9%99%A4%E8%A9%B2%E6%AD%BB%E7%9A%84terminating-namespace-c6594ebe351)

原因参考：
* [删除namespace为什么会Terminating](https://cloud.tencent.com/developer/article/1802531)
