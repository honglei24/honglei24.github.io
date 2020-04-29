---
layout: post
title:  "kubernetes删除namespace卡在terminating状态"
date:   2020-04-27 10:30:00 +0800
categories: kubernetes namespace
---
# 删除namespace
```
kubectl delete ns cattle-system
```

发现namespace一直处于terminating状态.
```
kubectl get ns
NAME              STATUS        AGE
argocd            Active        37d
cattle-system     Terminating   45d
default           Active        158d
kube-node-lease   Active        158d
kube-public       Active        158d
kube-system       Active        158d
```

google搜索了一下,发现一个处理方法:
    
    https://medium.com/@clouddev.guru/how-to-fix-kubernetes-namespace-deleting-stuck-in-terminating-state-5ed75792647e

# 获取namespace的详细信息
```
kubectl get namespace cattle-system -o yaml > cattle-system.yaml
```

# 编辑cattle-system.yaml
```
$ vi cattle-system.yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    cattle.io/status: '{"Conditions":[{"Type":"ResourceQuotaInit","Status":"True","Message":"","LastUpdateTime":"2020-03-12T10:29:06Z"},{"Type":"InitialRolesPopulated","Status":"True","Message":"","LastUpdateTime":"2020-03-12T10:29:11Z"}]}'
    field.cattle.io/projectId: c-qhn7r:p-2pj46
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Namespace","metadata":{"annotations":{},"name":"cattle-system"}}
    lifecycle.cattle.io/create.namespace-auth: "true"
  creationTimestamp: "2020-03-12T10:28:26Z"
  deletionGracePeriodSeconds: 0
  deletionTimestamp: "2020-04-27T02:44:31Z"
  finalizers:
  - controller.cattle.io/namespace-auth
  labels:
    field.cattle.io/projectId: p-2pj46
  name: cattle-system
  resourceVersion: "8401653"
  selfLink: /api/v1/namespaces/cattle-system
  uid: 3f127819-a100-4330-a3ca-e84241d9da0c
spec: {}
status:
  conditions:
  - lastTransitionTime: "2020-04-27T02:44:36Z"
    message: All resources successfully discovered
    reason: ResourcesDiscovered
    status: "False"
    type: NamespaceDeletionDiscoveryFailure
  - lastTransitionTime: "2020-04-27T02:44:36Z"
    message: All legacy kube types successfully parsed
    reason: ParsedGroupVersions
    status: "False"
    type: NamespaceDeletionGroupVersionParsingFailure
  - lastTransitionTime: "2020-04-27T02:44:36Z"
    message: All content successfully deleted
    reason: ContentDeleted
    status: "False"
    type: NamespaceDeletionContentFailure
  phase: Terminating

```
删除finalizers定义

# 更新namespace
```
kubectl replace -f cattle-system.yaml
``` 

确认namespace被删除.
```
kubectl get ns
NAME              STATUS   AGE
argocd            Active   37d
default           Active   158d
kube-node-lease   Active   158d
kube-public       Active   158d
kube-system       Active   158d
```