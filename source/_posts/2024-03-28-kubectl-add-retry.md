---
layout: next
title: kubectl执行失败后等待一段时间重试(Shell实现)
date: 2024-03-28 20:16:59
categories: Kubernetes
tags: Kubernetes
---

使用Shell脚本实现功能： `kubectl`执行失败后，等待30秒后再重试，一共重试3次，代码如下：
<!-- more -->
```bash
#!/bin/bash

KUBECTL_BIN=/var/lib/snapd/snap/bin/kubectl

ERR_MSG_K8S_NOTRUNNING="microk8s is not running"
ERR_MSG_CONNECTION_TIMEOUT="connection timed out"
ERR_MSG_CONNECTION_REFUSED="refused - did you specify the right host or port"

function debuglog() {
    local log_content=${@:1}
    local caller=$(basename "$0")
    echo "`date +"%Y-%m-%d %H:%M:%S"` [$caller] INFO $log_content" 2>&1
}

function api_kubectlX() {
    local retry=$1
    shift
    local args=$@
    local ret=0
    local cmdstr="${args[@]}"
    local tmpfile=$(mktemp)
    for ((i=0;i<$retry;i++)); do
        ${KUBECTL_BIN} $@ 2>"$tmpfile"
        ret=$?
        if [ $ret -eq 0 ]; then
            debuglog "KUBECTL $cmdstr ret: 0"
            break
        fi

        local errinfo=$(< "$tmpfile")
        local need_retry="no"
        if [ -n "$(echo "$errinfo" | grep "$ERR_MSG_K8S_NOTRUNNING")" ]; then
            need_retry="yes"
        elif [ -n "$(echo "$errinfo" | grep "$ERR_MSG_CONNECTION_TIMEOUT")" ]; then
            need_retry="yes"
        elif [ -n "$(echo "$errinfo" | grep "$ERR_MSG_CONNECTION_REFUSED")" ]; then
            need_retry="yes"
        elif [ $ret -eq 143 ]; then # 143 means SIGTERM, just retry
            need_retry="yes"
        else
            need_retry="no"
        fi

        if [ $need_retry != "yes" ]; then
            debuglog "KUBECTL ret: $ret/($errinfo) $cmdstr"
            break
        fi
        debuglog "KUBECTL ret: $ret/($errinfo) (retry:$i) $cmdstr"
        sleep 30
    done
    rm -- ${tmpfile}
    return $ret
}

function api_kubectl() {
    local args=(3 "$@")
    api_kubectlX "${args[@]}"
    return $?
}

# 使用方法：
api_kubectl get ns
```
