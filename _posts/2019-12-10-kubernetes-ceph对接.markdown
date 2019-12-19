---
layout: post
title:  "kubernetes ceph对接"
date:   2019-12-10 18:18:00 +0800

---

本文介绍了kubernetes集群中使用使用Ceph RBD的方法。

# 单节点ceph环境部署
首先部署一个但节点的ceph环境用来测试
```
# echo "
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-hammer/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-hammer/el7/noarch/
gpgcheck=0
" > /etc/yum.repos.d/ceph.repo
# yum install ceph ceph-radosgw ceph-deploy
# mkdir -p  /root/cluster
# cd /root/cluster
# systemctl stop firewalld.service 
# ceph-deploy new $HOSTNAME
# echo osd pool default size = 1 >> ceph.conf
# echo osd crush chooseleaf type = 0 >> ceph.conf
# echo osd max object name len = 256 >> ceph.conf
# echo osd journal size = 128 >> ceph.conf
# mkdir -p /var/run/ceph/
# chown ceph:ceph /var/run/ceph/
# useradd ceph
# chown ceph:ceph /var/run/ceph/
# mkdir -p /osd
# chown ceph:ceph /osd
# ceph-deploy mon create-initial
# ceph-deploy osd prepare $HOSTNAME:/osd
# ceph-deploy osd activate  $HOSTNAME:/osd
# ceph -s

```

# kubernetes环境配置

## 每个节点上安装ceph-common
```
# yum install -y ceph-common
```
安装完成之后把ceph.conf拷贝到每个节点的/etc/ceph/目录下。

## 创建 admin secret
先找ceph管理员要到key。可以通过下面的命令在ceph节点获取。
```
grep key /etc/ceph/ceph.client.admin.keyring |awk '{printf "%s", $NF}'|base64
```

利用上面的key创建kubernetes里的secret
```
# cat > 01_ceph-secret.yaml <EOF
apiVersion: v1
kind: Secret
metadata:
   name: ceph-secret
   namespace: kube-system
data:
   key: QVFCYWNPOWRIMFg1SlJBQXBZUEl6N2tJZFdtNXV0eFB5VDlTcnc9PQ==
type: kubernetes.io/rbd
EOF

# kubectl create -f 01_ceph-secret.yaml
```

## 创建storage-rbd-provisioner
<B>由于使用动态存储时 controller-manager 需要使用 rbd 命令创建 image。
所以 controller-manager 需要使用 rbd 命令，
由于官方controller-manager镜像里没有rbd命令，
如果没使用如下方式会报错无法成功创建pvc，
相关 issue: https://github.com/kubernetes/kubernetes/issues/38923。</B>

```
# cat > 02_external-storage-rbd-provisioner.yaml <EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["kube-dns"]
    verbs: ["list", "get"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
subjects:
  - kind: ServiceAccount
    name: rbd-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: rbd-provisioner
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rbd-provisioner
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rbd-provisioner
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rbd-provisioner
subjects:
- kind: ServiceAccount
  name: rbd-provisioner
  namespace: kube-system

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rbd-provisioner
  namespace: kube-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "quay.azk8s.cn/external_storage/rbd-provisioner:latest"
        imagePullPolicy: IfNotPresent
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccount: rbd-provisioner
EOF

# kubectl create -f 02_external-storage-rbd-provisioner.yaml
```
此处创建完成之后还需要把ceph.conf和ceph.client.admin.keyring拷贝到rbd-provisioner容器的/etc/ceph/目录下。


## 创建storageclass
```
# cat > 03_ceph-storageclass.yaml <EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 192.168.3.166:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system 
  pool: rbd 
  userId: admin
  userSecretName: ceph-secret
  userSecretNamespace: kube-system
  fsType: ext4
  imageFormat: "2"
EOF

# kubectl create -f 03_ceph-storageclass.yaml
```

## 创建PVC
```
# cat > 04_ceph-pvc.yaml <EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: ceph-claim
spec:
   accessModes:
     - ReadWriteOnce
   storageClassName: ceph-rbd
   resources:
     requests:
       storage: 1Gi
EOF

# kubectl create -f 04_ceph-pvc.yaml
```


## 创建pod,并挂在上面的pvc
```
# cat > 05_nginx-pod.yaml <EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod1
  labels:
    name: nginx-pod1
spec:
  containers:
  - name: nginx-pod1
    image: nginx
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: ceph-rdb
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ceph-rdb
    persistentVolumeClaim:
      claimName: ceph-claim

EOF

# kubectl create -f 05_nginx-pod.yaml
```