---
title: "Golang代理"
date: 2019-11-17T21:59:10+08:00
tags: ["go", "proxy"]
categories: ["环境"]
---

> 下载加速, 同时解决国内访问官方仓库被屏蔽的问题

<!--more-->

# 使用七牛代理
```shell
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

# 使用阿里代理
```shell
go env -w GO111MODULE=on
go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
```

# 临时修改
``` shell
export GO111MODULE=on
export GOPROXY=https://goproxy.cn,direct
```
