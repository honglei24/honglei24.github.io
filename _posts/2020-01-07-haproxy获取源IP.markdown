---
layout: post
title:  "openstack lbaas中获取客户端IP地址"
date:   2020-01-07 16:26:00 +0800
categories: kubernetes
---
## 背景
openstack lbaas默认driver为haproxy。使用haproxy之后的请求路径如下：
client ---> haproxy ---> real server
real server需要获取client IP。

## haproxy获取源IP的3中方式：

1. haproxy全透明代理.
    a. haproxy配置修改
    b. haproxy iptables规则修改
    c. backend默认路由修改
  a和b通过修改lbaas代码实现。c需要在应用服务器上配置。
  https://www.cnblogs.com/Bonker/p/6814183.html
  https://my.oschina.net/eddylinux/blog/535043

2. 利用haproxy的功能proxy_protocol 
    a. haproxy配置修改
    b. 后端也需要支持proxy_protocol 
    a通过修改lbaas代码实现，b需要backend支持。支持列表参考：https://www.haproxy.com/blog/haproxy/proxy-protocol/

3. 利用http头X-Forwarded-For来获取源IP，只在http模式下适用，tcp模式没用，但是可以在通过在客户端请求头显示添加X-Forwarded-For来实现。
