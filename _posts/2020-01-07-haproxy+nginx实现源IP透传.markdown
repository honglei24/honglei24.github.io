---
layout: post
title:  "openstack lbaas + k8s nginx-ingress实现源IP透传"
date:   2020-01-07 16:26:00 +0800
categories: kubernetes openstack
---
## 背景
openstack lbaas默认driver为haproxy。使用haproxy之后的请求路径如下：
client ---> haproxy ---> real server
real server需要获取client IP。

我们部署部署方式是k8s on openstack, 利用openstack的lbaas来做ingress的负载均衡器。大致的流程如下:
client -> haproxy(openstack) -> ingress(k8s) -> service(k8s)
ingress controller用的nginx来实现的。所以问题就归结为haproxy+nginx如何保持源IP。

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
 　　
   haproxy的配置里的加上配置选项option forwardfor即可。现在的lbaas里使用的haproxy模板已经有了该配置，不需要额外配置。

以上3种方式在下面简称为haproxy模式1，haproxy模式2，haproxy模式3。

## 利用nginx realip模块获取用户真实IP
1. 确认当前nginx是否带有realip模块，如果没有的话需要下载合适的nginx版本，或者重新编译nginx。
 
```
nginx -V 2>&1 | grep realip
```

2. 修改nginx配置，在nginx配置中增加以下配置（可以在http,server或location段中增加）
 
```
set_real_ip_from 0.0.0.0/; #真实服务器上一级代理的IP地址或者IP段,可以写多行。
real_ip_header X-Forwarded-For;  #从哪个header头检索出要的IP地址。
real_ip_recursive on; #递归的去除所配置中的可信IP。
```

3. 利用上述配置本地验证了haproxy模式3+步骤2的配置可以在nginx上获取源IP。

## 修改ingress congroller配置
1. 只需要关注compute-full-forwarded-for和use-forwarded-headers这两行。

```
# cat > values.yaml <<EOF
controller:
  config:
    worker-processes: "2"
    client-body-buffer-size: "50m"
    proxy-body-size: "50m"
    compute-full-forwarded-for: 'true'
    use-forwarded-headers: 'true'
  tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Equal"
      value: ""
      effect: "NoSchedule"
  kind: DaemonSet
  service:
    externalTrafficPolicy: Local
    type: NodePort
    nodePorts:
      http: 32080
      https: 32443
      tcp:
        8080: 32808
EOF

# helm upgrade nginx-ingress nginx-ingress-1.25.0.tgz -f values.yaml 
```
　
如果想修改nginx.conf里的set_real_ip_from，可以通过在config段下指定proxy-real-ip-cidr来实现。

2. 进入容器确认nginx配置已经修改。
 
```
www-data@nginx-ingress-controller-hswks:/etc/nginx$ grep real_ip nginx.conf 
	real_ip_header      X-Forwarded-For;
	real_ip_recursive   on;
	set_real_ip_from    0.0.0.0/0;

```

## 总结
1. haproxy模式3+nginx的realip模块可以解决源IP获取的问题。
2. 目前只支持haproxy的http模式，要使用tcp模式的话，可以通过客户端显示指定X-Forwarded-For头来实现。
3. 这里利用http头的X-Forwarded-For来替换nginx的内置变量remote_addr，客户端可以通过显示的制定X-Forwarded-For头来达到篡改的目的。
比如ingress的白名单为192.168.2.0/24,我在不是该网段的机器上可以通过指定X-Forwarded-For头绕过白名单。
```
curl -H 'X-Forwarded-For: 192.168.2.8' http://nginx-test.172.20.0.172.xip.io/
```
X-Forwarded-For是一个HTTP的扩展头，要解决这个问题，可以通过由用户制定特殊的HTTP来头来实现。但是涉及到代码修改。openstack lbaas的代码修改量较小。k8s的ingress-controller修改还有待进一步确认。
4. 根据官方文档来看，haproxy模式2+nginx也是可以实现的。关联的配置是use-proxy-protocol，这个没有验证。因为这个模式也需要修改lbaas代码。
5. haproxy模式1的全透明代理模式配置复杂，而且还需要在real server端配置静态路由。如果代码改造的话，改动比较大。如果用lvs的DR模式来做负载均衡器的话，可以更容易的实现IP投传，但是openstack社区只有设计图，没有具体的代码实现。需要自己来实现lvs driver.

## 参考
https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
https://www.cnblogs.com/dkblog/archive/2012/03/13/2393321.html
https://www.hi-linux.com/posts/53006.html
https://cloud.tencent.com/developer/article/1452511
https://wiki.openstack.org/wiki/Neutron/LBaaS/LVSDriver