---
layout: post
title:  "手动替换kube-apiserver证书"
date:   2019-12-19 16:59:00 +0800
categories: kubernetes apiserver
---
## 背景
用kubespray部署kubernetes集群之后，需要在apiserver前加一个负载均衡器。
但是在初始化集群的时候，负载均衡器的主机名/IP地址没有加到apiserver的证书里，
所以如果用负载均衡器的IP:6443来访问apiserver的话，回报如下的错误：
```
Unable to connect to the server: x509: certificate is valid for 10.42.0.1, 172.19.2.1, 172.19.2.1, 10.42.0.1, 127.0.0.1, 47.101.29.129, 172.19.2.1, 172.19.2.2, 172.19.2.3, not 172.19.2.100
```

其实在kubespray里定义了一个apiserver_loadbalancer_domain_name变量来指定负载均衡的hostname，其默认值是lb-apiserver.kubernetes.local。
该参数定义在roles/kubespray-defaults/defaults/main.yaml。有如下一些解决方法：
* 通过查看apiserver.crt信息。发现其中定义了lb-apiserver.kubernetes.local，所以最简单的做法是在DNS服务器绑定负载均衡器的IP和lb-apiserver.kubernetes.local，
这样可以通过lb-apiserver.kubernetes.local:6443来访问集群。工具可能需要下载，可以参照下面的工具下载章节。

```
$ cfssl-certinfo -cert apiserver.crt 
{
  ......
  "sans": [
    "master01",
    "master02",
    "master03",
    "lb-apiserver.kubernetes.local",
    ......
  ],
  ......
}

```
* 如果提前规划了负载均衡的IP，可以修改apiserver_loadbalancer_domain_name为负载均衡器的IP，这样部署出来的集群也是可以通过负载均衡器的IP来访问。

* 如果是用kubeadm部署单节点的话，可以通过指定apiserver-cert-extra-sans来实现。
```
sudo kubeadm init   --image-repository registry.aliyuncs.com/google_containers   --pod-network-cidr=10.24.0.0/16   --ignore-preflight-errors=Swap --apiserver-cert-extra-sans 127.0.0.1
```

* 在现有集群下手动更新证书，以下介绍手动更新的步骤。


## 下载工具
```
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ sudo chmod +x cfssl_linux-amd64
$ sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
 
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ sudo chmod +x cfssljson_linux-amd64
$ sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
 
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ sudo chmod +x cfssl-certinfo_linux-amd64
$ sudo mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
 
$ sudo export PATH=/usr/local/bin:$PATH
```

## 备份证书
```
$ sudo cp -r /etc/kubernetes/ssl /etc/kubernetes/ssl.org
```

## 生成新证书
```
$ cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF

$ cat > kubernetes-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
	    "localhost",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.k8s-prod.sh",
        "master01",
        "master02",
        "master03",
        "lb-apiserver.kubernetes.local",
        "172.19.2.1",
        "10.42.0.1",
        "127.0.0.1",
        "47.101.29.129",
        "172.19.2.1",
        "172.19.2.2",
        "172.19.2.3",
        "172.19.2.100"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
    ]
}
EOF

$ sudo cfssl gencert -ca=/etc/kubernetes/ssl/ca.crt -ca-key=/etc/kubernetes/ssl/ca.key -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```

## 替换证书重启api-server
```
$ sudo cp kubernetes-key.pem /etc/kubernetes/pki/apiserver.key 
$ sudo cp kubernetes.pem /etc/kubernetes/pki/apiserver.crt 
$ sudo docker restart $(sudo docker ps | grep k8s_kube-apiserver_kube-apiserver | awk '{print $1}')
```

## 确认
```
$ sed -i 's/172.19.2.1:6443/172.19.2.100:6443/g' ~/.kube/config
$ kubectl cluster-info 
Kubernetes master is running at https://172.19.2.100:6443
coredns is running at https://172.19.2.100:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://172.19.2.100:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```