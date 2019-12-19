---
layout: post
title:  "利用helm在k8s集群部署prometheus+grafana"
date:   2019-11-19 15.24:18 +0800
categories: kubernetes prometheus helm
---

**以下所有命令行操作都在master节点执行**
# Helm部署
```
# export VERSION="v2.14.0"
# mkdir -p /root/helm && cd /root/helm
# wget https://get.helm.sh/helm-${VERSION}-linux-amd64.tar.gz
# tar xf helm-${VERSION}-linux-amd64.tar.gz
# mv linux-amd64/helm /usr/local/bin
# rm -rf linux-amd64

# helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:${VERSION} --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

```
配置rbac
```
# cat > tiller-rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

# kubectl create -f tiller-rbac.yaml
# kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
# helm version
```

# nfs-client-provisioner部署
```
# helm repo add apphub https://apphub.aliyuncs.com
# helm install apphub/nfs-client-provisioner --name nfs --set nfs.server=192.168.3.153 --set nfs.path=/data/k8s13-test/ --set storageClass.defaultClass=true

测试
# cat > nfs_test.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
EOF

# kubectl create -f nfs_test.yaml
# kubectl get pvc --field-selector='metadata.name==test'
NAME   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test   Bound    pvc-1f251204-0113-11ea-b72a-005056a63d64   1Gi        RWO            nfs-client     7h17m
```

# prometheus+grafana部署
```
 #  helm fetch --untar apphub/prometheus
 #  helm install prometheus --name prometheus

 #  helm fetch --untar apphub/grafana
 #  helm install grafana --name grafana

```

# grafana配置
```
# export POD_NAME=$(kubectl get pods --namespace default -l "app=grafana,release=grafana" -o jsonpath="{.items[0].metadata.name}")
# kubectl --namespace default --address localhost,192.168.0.1 port-forward $POD_NAME 3000
```
grafana admin用户密码获取
```
# kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
浏览器登录 192.168.0.1:3000,
配置grafana的默认数据源为http://prometheus-server。

导入kubernetes集群监控dashboard。导入json内容参考附件。
[kubernetes-cluster-monitoring-via-prometheus_rev1.json](https://github.com/honglei24/honglei24.github.io/tree/master/appendix/kubernetes/prometheus/kubernetes-cluster-monitoring-via-prometheus_rev1.json)

prometheus容量规划。[prometheus_helm.xlsx](https://github.com/honglei24/honglei24.github.io/tree/master/appendix/kubernetes/prometheus/prometheus_helm.xlsx)


# nginx-ingress配置
```
# helm fetch apphub/nginx-ingress
# cat > values.yaml <<EOF
controller:
  service:
    type: NodePort
    nodePorts:
      http: 32080
      https: 32443
      tcp:
        8080: 32808
EOF 

# helm install nginx-ingress-1.25.0.tgz -f values.yaml --name nginx-ingress
```

创建grafana ingress
```
# cat > grafana_ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: grafana
spec:
  rules:
   - host: grafana.test.com
     http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 80
EOF

# kubectl create -f grafana_ingress.yaml
```

在本地hosts文件里追加一行
```
echo "172.20.3.142 grafana.test.com" >>/etc/hosts
```
在浏览器里访问http://grafana.test.com:32080/，即可访问grafana，然后添加数据源prometheus-server，导入对应的dashboard(kubernetes-cluster-monitoring-via-prometheus_rev1.json),就可以查看集群的资源使用状况了。
