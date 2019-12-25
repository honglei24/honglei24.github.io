---
layout: post
title:  "kubespray从零部署kubernetes集群"
date:   2019-12-17 17:46:00 +0800
categories: kubernetes
---

## 从零部署kubernetes集群
集群所用的虚拟机在openstack集群里。
集群环境

|-------+----------|----------|
|   IP  | 操作系统  |角色  |
|-------+----------|----------|
|  ----  | ----  | ----  |
| 172.19.3.1,172.20.3.20  | CentOS7 | nfs-server, 部署节点 |
| 172.19.2.1,172.20.3.142  | Ubuntu18.04 | master01  |
| 172.19.2.2  | Ubuntu18.04 | master02  |
| 172.19.2.3  | Ubuntu18.04 | master03  |
| 172.19.2.4  | Ubuntu18.04 | node01  |
| 172.19.2.5  | Ubuntu18.04 | node02  |
| 172.19.2.6  | Ubuntu18.04 | node03  |
| 172.19.2.7  | Ubuntu18.04 | node04  |
| 172.19.2.8  | Ubuntu18.04 | node05  |

## kubespray部署kubernetes集群
```
# yum -y install epel-release
# yum install python-pip

# mkdir -p /root/.pip/
# cat > /root/.pip/pip.conf <<EOF 
[global]
index-url = http://mirrors.aliyun.com/pypi/simple
trusted-host = mirrors.aliyun.com
EOF

# pip install --upgrade pip

# pip install ansible==2.7.8

# yum install git
# git clone https://gitlab.oneitfarm.com/idg-public/kubespray.git
# cd kubespray
# git checkout release-2.10
# cp kubespray/aliyun-prod-sh kubespray/test -r
# for i in `seq 1 8`; do ssh-copy-id ubuntu@172.19.2.${i}; done
# cat > inventory/test/hosts.ini <<EOF
[all]
master01    ansible_host=172.19.2.1 ip=172.19.2.1
master02    ansible_host=172.19.2.2 ip=172.19.2.2
master03    ansible_host=172.19.2.3 ip=172.19.2.3
node01      ansible_host=172.19.2.4 ip=172.19.2.4
node02      ansible_host=172.19.2.5 ip=172.19.2.5
node03      ansible_host=172.19.2.6 ip=172.19.2.6
node04      ansible_host=172.19.2.7 ip=172.19.2.7
node05      ansible_host=172.19.2.8 ip=172.19.2.8

[kube-master]
master01
master02
master03

[etcd]
master01
master02
master03

[kube-node]
node01
node02
node03
node04
node05

[k8s-cluster:children]
kube-master
kube-node

[all:vars]
ansible_user=ubuntu
ansible_become=yes
ansible_become_user=root
EOF

# yum install python-netaddr

```

中间碰到的坑
1. TASK [etcd : Check_certs | Set 'sync_certs' to true]报错，错误信息如下：
```
fatal: [master02]: FAILED! => {
    "msg": "The conditional check 'gen_node_certs[inventory_hostname] or (not etcdcert_node.results[0].stat.exists|default(false)) or (not etcdcert_node.results[1].stat.exists|default(false)) or (etcdcert_node.results[1].stat.checksum|default('') != etcdcert_master.files|selectattr(\"path\", \"equalto\", etcdcert_node.results[1].stat.path)|map(attribute=\"checksum\")|first|default(''))' failed. The error was: no test named 'equalto'\n\nThe error appears to have been in '/root/kubespray/roles/etcd/tasks/check_certs.yml': line 57, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: \"Check_certs | Set 'sync_certs' to true\"\n  ^ here\n"
}
```
解决方法：
需要在master阶段上安装Jinja2 2.8
```
$ sudo apt install python-pip
$ pip install Jinja2==2.8
```

## 确认集群状态
```
# mkdir ~/.kube
# scp ubuntu@master01:/etc/kubernetes/admin.conf ~/.kube/config
# scp ubuntu@master01:/usr/bin/kubectl /usr/bin/kubectl
# kubectl get  no
NAME       STATUS   ROLES    AGE     VERSION
master01   Ready    master   10m     v1.14.6
master02   Ready    master   9m22s   v1.14.6
master03   Ready    master   9m23s   v1.14.6
node01     Ready    <none>   8m16s   v1.14.6
node02     Ready    <none>   8m15s   v1.14.6
node03     Ready    <none>   8m15s   v1.14.6
node04     Ready    <none>   8m14s   v1.14.6
node05     Ready    <none>   8m15s   v1.14.6

# kubectl get nodes | grep none | awk '{print $1}' | xargs -i kubectl label node {} node-role.kubernetes.io/node=
node/node01 labeled
node/node02 labeled
node/node03 labeled
node/node04 labeled
node/node05 labeled

# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   92m   v1.14.6
master02   Ready    master   92m   v1.14.6
master03   Ready    master   92m   v1.14.6
node01     Ready    node     90m   v1.14.6
node02     Ready    node     90m   v1.14.6
node03     Ready    node     90m   v1.14.6
node04     Ready    node     90m   v1.14.6
node05     Ready    node     90m   v1.14.6

# yum install -y bash-completion
# echo "source <(kubectl completion bash)" >> ~/.bashrc
```

创建vg,lv并挂载到对应目录。
```
# yum install lvm2 -y
# vgcreate vg_k8s /dev/vdb
# lvcreate -l 100%VG -n lv_k8s vg_k8s
# mkfs.ext4 /dev/vg_k8s/lv_k8s
# echo -e "/dev/vg_k8s/lv_k8s   /data\t          ext4       defaults              0 0" >>/etc/fstab
# mount -a
# df -h /data
Filesystem                 Size  Used Avail Use% Mounted on
/dev/mapper/vg_k8s-lv_k8s 1008G   77M  957G   1% /data

# for i in `seq 1 8`; do ssh ubuntu@172.19.2.${i} "sudo apt install -y nfs-common"; done
```

## nfs-server部署
```
# sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
# reboot

# yum install nfs-utils wget -y

# mkdir -p /data/k8s/
# cat > /etc/exports <<EOF
/data/k8s/        172.19.0.0/16(rw,insecure,sync,no_subtree_check,no_root_squash)
EOF

# chmod 777 /data/ -R

# systemctl enable rpcbind
# systemctl enable nfs

# systemctl start rpcbind
# systemctl start nfs
```

如有必要，防火墙需要打开 rpc-bind 和 nfs 的服务
```
# firewall-cmd --zone=public --permanent --add-service={rpc-bind,mountd,nfs}
# firewall-cmd --reload
```

接下来helm, prometheus, nfs-provide参照https://honglei24.github.io/kubernetes/prometheus/helm/2019/11/19/prometheus/