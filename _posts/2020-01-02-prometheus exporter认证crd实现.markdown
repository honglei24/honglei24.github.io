---
layout: post
title:  "prometheus exporter认证crd实现"
date:   2020-01-02 15:16:00 +0800
categories: kubernetes
---
## 背景
给业务pod注入nginx作为反向代理来实现拉取metrics的认证。

## 利用kubuilder开发CRD
```
# cd $(go env PATH)/src/
# mkdir exporter-controller
# kubebuilder init --domain ci.com --owner "honglei"
# kubebuilder create api --group app --version v1 --kind Exporter

```

在完善了 AppSpec 和 Controller 的 Reconcile 函数后，使 Kubebuilder 重新生成代码，并将 config/crd 下的 CRD yaml 应用到当前集群：
```
# make && make install && make run
```
具体代码参考[exproter-controller]([http](https://github.com/honglei24/exporter-controller))

## 注意点
不使用 Finalizer 时，资源被删除无法获取任何信息；
对象的 Status 字段变化也会触发 Reconcile 方法；
Reconcile 逻辑需要幂等；

## 待完善
该CRD只实现了nginx注入，但是nginx注入之后需要自动修改配置，这一块还需要完善。
同时prometheus需要加上对应的认证配置。

## 参考
https://blog.hdls.me/15708754600835.html
https://www.jianshu.com/p/edd9c17d8c8b
https://juejin.im/post/5d8acac2e51d4577f54a0fd2