---
layout: post
title:  "kubeadm 部署kubernetes集群"
date:   2019-12-20 09:00:00 +0800
categories: kubernetes
---

简要记录kubeadm部署kubernetes的步骤

## 部署master
```
$ sudo swapoff -a


$ sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
$ curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add 
sudo apt update

$ sudo apt update && sudo apt install -y apt-transport-https curl docker.io

$ sudo cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
$ curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

$ sudo apt update && sudo apt install apt-transport-https ca-certificates curl $ software-properties-common -y

$ sudo apt install -y kubelet kubeadm kubectl
$ sudo systemctl daemon-reload && sudo systemctl restart kubelet

$ sudo systemctl enable docker

$ sudo kubeadm init   --image-repository registry.aliyuncs.com/google_containers   --pod-network-cidr=10.24.0.0/16   --ignore-preflight-errors=Swap --apiserver-cert-extra-sans 127.0.0.1

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

获取join token的命令
$ kubeadm token create --print-join-command
```

## 部署flannel
```
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ sed -i 's/quay.io/quay.azk8s.cn/g' kube-flannel.yml
$ kubectl create -f kube-flannel.yml
```

## 部署node节点
```

$ sudo swapoff -a
$ sudo vim /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
$ sudo apt update
$ sudo apt install docker kubeadm kubelet
$ sudo systemctl enable docker
$ sudo kubeadm join master:6443 --token 6dg13o.fyw6jf7ynt7gy40q --discovery-token-ca-cert-hash sha256:4bb3f1d889b0eb9799652c33464ebe747ef74b7199a41646a5f66cf068d2c3bb

```