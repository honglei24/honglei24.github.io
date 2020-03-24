---
layout: post
title:  "etcd v3常用命令"
date:   2020-03-23 22：00:00 +0800
categories: kubernetes etcd
---
## 背景
kubernetes使用etcd来存储数据，kubernetes的网络插件也利用etcd来存储数据，我们来看看etcd里的数据结构。

## etcd API版本：
etcd2和etcd3是不兼容的，两者的api参数也不一样，详细请查看 etcdctl -h 。
可以使用api2 和 api3 写入 etcd 数据，但是需要注意，使用不同的api版本写入数据需要使用相应的api版本读取数据。
kubernetes v1.13.0开始不支持etcd2作为后端。

## 常见操作
以下以kubeadm部署的但节点为例说明，根据世纪情况需要调整
```
sudo su -
cd /etc/kubernetes/pki
etcdctl --cacert="etcd/ca.crt" --key=apiserver-etcd-client.key --cert=apiserver-etcd-client.crt get / --prefix --keys-only
 ```
