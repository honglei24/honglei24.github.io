---
layout: post
title:  "openstack修改ubuntu默认密码"
date:   2019-12-17 15:00:00 +0800

---

* 镜像里安装clout-init

* 创建虚拟机的时候添加自定义脚本
```
#cloud-config
ssh_pwauth: yes
chpasswd:
  list: |
      root:123456
      ubuntu:123456
  expire: false
```