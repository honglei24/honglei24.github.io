---
layout: post
title:  "kubernetes日志"
date:   2020-03-24 17:40:00 +0800
categories: kubernetes 日志
---
# Kubernetes日志

本文从以下三个维度要解读kubernetes的日志
1. [kubernetes主要组件的日志](!#Kubernetes主要组件日志)
2. [kubernetes提供的event资源对象](!Event)
3. [审计日志](!审计日志)

## Kubernetes主要组件日志
1. kubernetes日志等级设置为5，具体配置是在启动组件的时候加上--v=5,否则很多日志查看不到。
2. 以下以创建deployment为例说明。使用到的yaml文件和命令如下
```
$ cat nginx-deployment.yaml 
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: nginx-deploy
spec: 
  replicas: 2
  selector:
    matchLabels:
      name: nginx
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http

$ kubectl create -f nginx-deployment.yaml 

```

### 日志解读

#### kube-apiserver
kube-apiserver日志示例：
```
{"log":"I0324 03:42:44.573965       1 httplog.go:90] POST /apis/apps/v1/namespaces/default/deployments: (13.177203ms) 201 [kubectl/v1.17.4 (linux/amd64) kubernetes/8d8aa39 192.168.3.2:42876]\n","stream":"stderr","time":"2020-03-24T03:42:44.574561651Z"}
```
* 主要字段说明：
请求方法：POST
请求路径：/apis/apps/v1/namespaces/default/deployments
请求客户端：kubectl
返回状态码：201

#### kube-controllermanager
以下截取了DeploymentController会watch和ReplicaSetController的相关日志
```
{"log":"I0324 03:42:44.570025       1 deployment_controller.go:168] Adding deployment nginx-deploy\n","stream":"stderr","time":"2020-03-24T03:42:44.57203038Z"}
{"log":"I0324 03:42:44.570056       1 deployment_controller.go:564] Started syncing deployment \"default/nginx-deploy\" (2020-03-24 03:42:44.570046661 +0000 UTC m=+2047.567174118)\n","stream":"stderr","time":"2020-03-24T03:42:44.572077585Z"}
{"log":"I0324 03:42:44.570379       1 deployment_util.go:259] Updating replica set \"nginx-deploy-78bc8876dd\" revision to 1\n","stream":"stderr","time":"2020-03-24T03:42:44.572092026Z"}
{"log":"I0324 03:42:44.608345       1 replica_set.go:288] Adding ReplicaSet default/nginx-deploy-78bc8876dd\n","stream":"stderr","time":"2020-03-24T03:42:44.60887584Z"}
{"log":"I0324 03:42:44.608407       1 deployment_controller.go:214] ReplicaSet nginx-deploy-78bc8876dd added.\n","stream":"stderr","time":"2020-03-24T03:42:44.608933679Z"}
{"log":"I0324 03:42:44.608526       1 replica_set.go:561] Too few replicas for ReplicaSet default/nginx-deploy-78bc8876dd, need 2, creating 2\n","stream":"stderr","time":"2020-03-24T03:42:44.609128056Z"}
```
控制其主要工作：
1. DeploymentController会watch和ReplicaSetController通过kube-apiserver watch资源对象的变化。
2. 根据实际状态与期望状态的差异来执行对应的逻辑,将集群状态调整到期望状态。
    * 实际状态：kubernetes集群本身的现状。
    * 期望状态：一般来自于用户提交的yaml文件

各种控制器执行逻辑基本相似，大致流程如下所示：
```
for {
  实际状态 := 获取集群中对象X的实际状态
  期望状态 := 获取集群中对象X的期望状态
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

各种Controller watch的对象不同，比如：
* DeploymentController会watch deployment,replicaset，pod这3种资源对象
* ReplicaSetController会watch replicaset和pod这两种种资源对象

kube-apiserver中可以查看到controllermanager调用apiserver创建资源对象的日志：
```
{"log":"I0324 03:42:44.607936       1 httplog.go:90] POST /apis/apps/v1/namespaces/default/replicasets: (34.612933ms) 201 [kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:deployment-controller 192.168.3.2:54368]\n","stream":"stderr","time":"2020-03-24T03:42:44.610565146Z"}
{"log":"I0324 03:42:44.735468       1 httplog.go:90] POST /api/v1/namespaces/default/pods: (126.045527ms) 201 [kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:replicaset-controller 192.168.3.2:54368]\n","stream":"stderr","time":"2020-03-24T03:42:44.7356929Z"}
{"log":"I0324 03:42:44.753459       1 httplog.go:90] POST /api/v1/namespaces/default/pods: (13.952351ms) 201 [kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:replicaset-controller 192.168.3.2:54368]\n","stream":"stderr","time":"2020-03-24T03:42:44.753602336Z"}
{"log":"I0324 03:42:44.753667       1 httplog.go:90] POST /api/v1/namespaces/default/events: (14.077901ms) 201 [kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:replicaset-controller 192.168.3.2:54368]\n","stream":"stderr","time":"2020-03-24T03:42:44.753803869Z"}
{"log":"I0324 03:42:44.910393       1 httplog.go:90] POST /api/v1/namespaces/default/events: (154.704266ms) 201 [kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:replicaset-controller 192.168.3.2:54368]\n","stream":"stderr","time":"2020-03-24T03:42:44.911370764Z"}
```
第一行是创建replicaset，第二行是创建pod。

#### kube-scheduler
以下截取了kube-scheduler中绑定pod到合适的节点的日志。
```
{"log":"I0324 03:42:44.741782       1 scheduling_queue.go:841] About to try and schedule pod default/nginx-deploy-78bc8876dd-8j29k\n","stream":"stderr","time":"2020-03-24T03:42:44.741977433Z"}
{"log":"I0324 03:42:44.742049       1 scheduler.go:605] Attempting to schedule pod: default/nginx-deploy-78bc8876dd-8j29k\n","stream":"stderr","time":"2020-03-24T03:42:44.742227013Z"}
{"log":"I0324 03:42:44.743326       1 factory.go:519] Attempting to bind nginx-deploy-78bc8876dd-8j29k to hl\n","stream":"stderr","time":"2020-03-24T03:42:44.743480703Z"}
{"log":"I0324 03:42:44.754842       1 scheduling_queue.go:841] About to try and schedule pod default/nginx-deploy-78bc8876dd-rthvs\n","stream":"stderr","time":"2020-03-24T03:42:44.755069502Z"}
{"log":"I0324 03:42:44.755026       1 scheduler.go:605] Attempting to schedule pod: default/nginx-deploy-78bc8876dd-rthvs\n","stream":"stderr","time":"2020-03-24T03:42:44.75512981Z"}
{"log":"I0324 03:42:44.755903       1 factory.go:519] Attempting to bind nginx-deploy-78bc8876dd-rthvs to hl\n","stream":"stderr","time":"2020-03-24T03:42:44.756021915Z"}
{"log":"I0324 03:42:44.776169       1 scheduler.go:751] pod default/nginx-deploy-78bc8876dd-8j29k is bound successfully on node \"hl\", 1 nodes evaluated, 1 nodes were found feasible.\n","stream":"stderr","time":"2020-03-24T03:42:44.77738641Z"}
{"log":"I0324 03:42:44.915173       1 scheduler.go:751] pod default/nginx-deploy-78bc8876dd-rthvs is bound successfully on node \"hl\", 1 nodes evaluated, 1 nodes were found feasible.\n","stream":"stderr","time":"2020-03-24T03:42:44.915431504Z"}
```
scheduler通过watch pod资源对象，发现pod需要调度。通过预选和优选两阶段，选出合适的node，将pod绑定到node，通过apiserver将绑定关系写入etcd。kube-scheduler调用kube-apiserver的日志可以再kube-apiserver日志里查看
```
{"log":"I0324 03:42:44.774434       1 httplog.go:90] POST /api/v1/namespaces/default/pods/nginx-deploy-78bc8876dd-8j29k/binding: (29.573108ms) 201 [kube-scheduler/v1.17.0 (linux/amd64) kubernetes/70132b0/scheduler 192.168.3.2:53930]\n","stream":"stderr","time":"2020-03-24T03:42:44.777531341Z"}
{"log":"I0324 03:42:44.904052       1 httplog.go:90] POST /api/v1/namespaces/default/pods/nginx-deploy-78bc8876dd-rthvs/binding: (146.407817ms) 201 [kube-scheduler/v1.17.0 (linux/amd64) kubernetes/70132b0/scheduler 192.168.3.2:53930]\n","stream":"stderr","time":"2020-03-24T03:42:44.90420613Z"}
```

#### kubelet
```
3月 24 15:50:32 hl kubelet[3752]: I0324 15:50:32.074030    3752 config.go:412] Receiving a new pod "nginx-deploy-78bc8876dd-v92lq_default(62972b12-c5f4-4736-b906-0ff931a55a80)"
3月 24 15:50:32 hl kubelet[3752]: I0324 15:50:32.288308    3752 status_manager.go:544] Patch status for pod "nginx-deploy-78bc8876dd-v92lq_default(62972b12-c5f4-4736-b906-0ff931a55a80)" with "{\"metadata\":{\"uid\":\"62972b12-c5f4-4736-b906-0ff931a55a80\"},\"status\":{\"$setElementOrder/conditions\":[{\"type\":\"Initialized\"},{\"type\":\"Ready\"},{\"type\":\"ContainersReady\"},{\"type\":\"PodScheduled\"}],\"conditions\":[{\"lastProbeTime\":null,\"lastTransitionTime\":\"2020-03-24T07:50:32Z\",\"status\":\"True\",\"type\":\"Initialized\"},{\"lastProbeTime\":null,\"lastTransitionTime\":\"2020-03-24T07:50:32Z\",\"message\":\"containers with unready status: [nginx]\",\"reason\":\"ContainersNotReady\",\"status\":\"False\",\"type\":\"Ready\"},{\"lastProbeTime\":null,\"lastTransitionTime\":\"2020-03-24T07:50:32Z\",\"message\":\"containers with unready status: [nginx]\",\"reason\":\"ContainersNotReady\",\"status\":\"False\",\"type\":\"ContainersReady\"}],\"containerStatuses\":[{\"image\":\"nginx\",\"imageID\":\"\",\"lastState\":{},\"name\":\"nginx\",\"ready\":false,\"restartCount\":0,\"started\":false,\"state\":{\"waiting\":{\"reason\":\"ContainerCreating\"}}}],\"hostIP\":\"192.168.3.2\",\"startTime\":\"2020-03-24T07:50:32Z\"}}"
```
kubelet通过watch机制，发现当前节点绑定了一个pod，创建网络，卷等信息，并把kubernetes附带的config信息转化为容器运行时参数，利用创建的网络+卷+配置信息再当前节点启动pod。创建完成之后还会将信息上报给kube-apiserver. kube-apiserver对应的日志如下： 
```
{"log":"I0324 07:50:32.253502       1 httplog.go:90] GET /api/v1/namespaces/default/pods/nginx-deploy-78bc8876dd-v92lq: (66.583365ms) 200 [kubelet/v1.17.4 (linux/amd64) kubernetes/8d8aa39 192.168.3.2:41958]\n","stream":"stderr","time":"2020-03-24T07:50:32.25424078Z"}
{"log":"I0324 07:50:32.287478       1 httplog.go:90] PATCH /api/v1/namespaces/default/pods/nginx-deploy-78bc8876dd-v92lq/status: (32.692723ms) 200 [kubelet/v1.17.4 (linux/amd64) kubernetes/8d8aa39 192.168.3.2:41958]\n","stream":"stderr","time":"2020-03-24T07:50:32.288126136Z"}
```
PATCH方法就是更新pod状态的接口。


# Event
除了通过kubernetes组件来分析资源创建流程之外，还可以利用kubernetes的event资源对象来查看资源创建与删除的日志，相较于组件日志，event日志更加直观。
```
$ kubectl get event
LAST SEEN   TYPE     REASON                    OBJECT                               MESSAGE
47m         Normal   Starting                  node/hl                              Starting kubelet.
47m         Normal   NodeHasSufficientMemory   node/hl                              Node hl status is now: NodeHasSufficientMemory
47m         Normal   NodeHasNoDiskPressure     node/hl                              Node hl status is now: NodeHasNoDiskPressure
47m         Normal   NodeHasSufficientPID      node/hl                              Node hl status is now: NodeHasSufficientPID
47m         Normal   NodeNotReady              node/hl                              Node hl status is now: NodeNotReady
47m         Normal   NodeAllocatableEnforced   node/hl                              Updated Node Allocatable limit across pods
47m         Normal   Starting                  node/hl                              Starting kubelet.
47m         Normal   NodeHasSufficientMemory   node/hl                              Node hl status is now: NodeHasSufficientMemory
47m         Normal   NodeHasNoDiskPressure     node/hl                              Node hl status is now: NodeHasNoDiskPressure
47m         Normal   NodeHasSufficientPID      node/hl                              Node hl status is now: NodeHasSufficientPID
47m         Normal   NodeAllocatableEnforced   node/hl                              Updated Node Allocatable limit across pods
47m         Normal   NodeReady                 node/hl                              Node hl status is now: NodeReady
46m         Normal   Killing                   pod/nginx-deploy-78bc8876dd-8j29k    Stopping container nginx
45m         Normal   Scheduled                 pod/nginx-deploy-78bc8876dd-h7nhp    Successfully assigned default/nginx-deploy-78bc8876dd-h7nhp to hl
45m         Normal   Pulled                    pod/nginx-deploy-78bc8876dd-h7nhp    Container image "nginx" already present on machine
45m         Normal   Created                   pod/nginx-deploy-78bc8876dd-h7nhp    Created container nginx
45m         Normal   Started                   pod/nginx-deploy-78bc8876dd-h7nhp    Started container nginx
46m         Normal   Killing                   pod/nginx-deploy-78bc8876dd-rthvs    Stopping container nginx
45m         Normal   Scheduled                 pod/nginx-deploy-78bc8876dd-v92lq    Successfully assigned default/nginx-deploy-78bc8876dd-v92lq to hl
45m         Normal   Pulled                    pod/nginx-deploy-78bc8876dd-v92lq    Container image "nginx" already present on machine
45m         Normal   Created                   pod/nginx-deploy-78bc8876dd-v92lq    Created container nginx
45m         Normal   Started                   pod/nginx-deploy-78bc8876dd-v92lq    Started container nginx
45m         Normal   SuccessfulCreate          replicaset/nginx-deploy-78bc8876dd   Created pod: nginx-deploy-78bc8876dd-h7nhp
45m         Normal   SuccessfulCreate          replicaset/nginx-deploy-78bc8876dd   Created pod: nginx-deploy-78bc8876dd-v92lq
45m         Normal   ScalingReplicaSet         deployment/nginx-deploy              Scaled up replica set nginx-deploy-78bc8876dd to 2
```

Event由Kubernetes的核心组件Kubelet和ControllerManager等产生，用来记录系统一些重要的状态变更。
与其他资源对象一样，Event对象也通过调用apiserver，将Event存储在etcd里。
event默认保留事件是1小时，可以通过apiserver的event-ttl参数来配置。
* 以下是apiserver日志中关于event调用的例子
```
{"log":"I0324 03:11:55.353939       1 httplog.go:90] POST /api/v1/namespaces/harbor/events: (666.620288ms) 201 [kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:persistent-volume-binder 192.168.3.2:54368]\n","stream":"stderr","time":"2020-03-24T03:11:55.354083534Z"}

```

* 以下是etcd里的Event存储示例：
```
# etcdctl get /registry/event  --prefix --keys-only
......
/registry/events/default/hl.15ff2df222ff1329
/registry/events/default/nginx-deploy-78bc8876dd-8j29k.15ff2e0ade740f86
/registry/events/default/nginx-deploy-78bc8876dd-h7nhp.15ff2e0e177a9d25
/registry/events/default/nginx-deploy-78bc8876dd-h7nhp.15ff2e10c9a173d6
/registry/events/default/nginx-deploy-78bc8876dd-h7nhp.15ff2e12c7d1a4b3
/registry/events/default/nginx-deploy-78bc8876dd-h7nhp.15ff2e12ff2d374f
/registry/events/default/nginx-deploy-78bc8876dd-rthvs.15ff2e0adf8a52ee
/registry/events/default/nginx-deploy-78bc8876dd-v92lq.15ff2e0e20e95d0c
/registry/events/default/nginx-deploy-78bc8876dd-v92lq.15ff2e10d421fa99
/registry/events/default/nginx-deploy-78bc8876dd-v92lq.15ff2e12cac4d6a9
/registry/events/default/nginx-deploy-78bc8876dd-v92lq.15ff2e1304bbadf6
/registry/events/default/nginx-deploy-78bc8876dd.15ff2e0dd96ff6e5
/registry/events/default/nginx-deploy-78bc8876dd.15ff2e0df6802d37
/registry/events/default/nginx-deploy.15ff2e0d9012cf82
......

# etcdctl get /registry/events/default/nginx-deploy-78bc8876dd-v92lq.15ff2e0e20e95d0c  --prefix --keys-only=false -w json | python -m json.tool
{
    "count": 1,
    "header": {
        "cluster_id": 4264270518481295231,
        "member_id": 1902898853476784267,
        "raft_term": 45,
        "revision": 1891902
    },
    "kvs": [
        {
            "create_revision": 1885298,
            "key": "L3JlZ2lzdHJ5L2V2ZW50cy9kZWZhdWx0L25naW54LWRlcGxveS03OGJjODg3NmRkLXY5MmxxLjE1ZmYyZTBlMjBlOTVkMGM=",
            "lease": 6380317590371777085,
            "mod_revision": 1885298,
            "value": "azhzAAoLCgJ2MRIFRXZlbnQS5AIKcwoubmdpbngtZGVwbG95LTc4YmM4ODc2ZGQtdjkybHEuMTVmZjJlMGUyMGU5NWQwYxIAGgdkZWZhdWx0IgAqJGZhMWIxNDhmLTlhMjgtNGVkOC1iN2NmLTVlYmY3ZjVhM2Q5ZjIAOABCCAjI9+bzBRAAegASYgoDUG9kEgdkZWZhdWx0Gh1uZ2lueC1kZXBsb3ktNzhiYzg4NzZkZC12OTJscSIkNjI5NzJiMTItYzVmNC00NzM2LWI5MDYtMGZmOTMxYTU1YTgwKgJ2MTIHMTg4NTI4OToAGglTY2hlZHVsZWQiQVN1Y2Nlc3NmdWxseSBhc3NpZ25lZCBkZWZhdWx0L25naW54LWRlcGxveS03OGJjODg3NmRkLXY5MmxxIHRvIGhsKhUKEWRlZmF1bHQtc2NoZWR1bGVyEgAyCAjI9+bzBRAAOggIyPfm8wUQAEABSgZOb3JtYWxSAGIAcgB6ABoAIgA=",
            "version": 1
        }
    ]
}
```

# 审计日志
Kubernetes 审计（Audit）提供了安全相关的时序操作记录，支持日志和 webhook 两种格式，并可以通过审计策略自定义事件类型。
如果使用日志记录，需要在kube-apiserver的启动参数里添加如下配置项，
```
audit-log-maxbackup=10	审计日志最大分片存储10个日志文件
audit-log-maxsize=100	单个审计日志最大size为100MB
audit-log-path=/var/log/kubernetes/audit.log	审计日志输出路径为/var/log/kubernetes/audit.log
audit-log-maxage=7	审计日志最多保存期为7天
audit-policy-file=/etc/kubernetes/audit-policy/audit-policy.yml	审计日志配置策略文件，文件路径为：/etc/kubernetes/audit-policy/audit-policy.yml
```

配置好之后重启kube-apiserver,可以在/var/log/kubernetes/audit.log文件里查看具体的审计日志，典型的audit日志如下所示：
```
{
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"system:controller:deployment-controller\" of ClusterRole \"system:controller:deployment-controller\" to ServiceAccount \"deployment-controller/kube-system\""
    },
    "apiVersion": "audit.k8s.io/v1",
    "auditID": "f0fed96b-11c6-4066-a7a3-249cd39bb83f",
    "kind": "Event",
    "level": "RequestResponse",
    "objectRef": {
        "apiGroup": "apps",
        "apiVersion": "v1",
        "name": "nginx-deploy",
        "namespace": "default",
        "resource": "deployments",
        "resourceVersion": "1885351",
        "subresource": "status",
        "uid": "8a676d6d-2d41-49f2-8aac-e0030a2a28a8"
    },
    "requestObject": {
        "apiVersion": "apps/v1",
        "kind": "Deployment",
        "metadata": {
            "annotations": {
                "deployment.kubernetes.io/revision": "1"
            },
            "creationTimestamp": "2020-03-24T07:50:28Z",
            "generation": 1,
            "name": "nginx-deploy",
            "namespace": "default",
            "resourceVersion": "1885351",
            "selfLink": "/apis/apps/v1/namespaces/default/deployments/nginx-deploy",
            "uid": "8a676d6d-2d41-49f2-8aac-e0030a2a28a8"
        },
        "spec": {
            "progressDeadlineSeconds": 600,
            "replicas": 2,
            "revisionHistoryLimit": 10,
            "selector": {
                "matchLabels": {
                    "name": "nginx"
                }
            },
            "strategy": {
                "rollingUpdate": {
                    "maxSurge": "25%",
                    "maxUnavailable": "25%"
                },
                "type": "RollingUpdate"
            },
            "template": {
                "metadata": {
                    "creationTimestamp": null,
                    "labels": {
                        "name": "nginx"
                    }
                },
                "spec": {
                    "containers": [
                        {
                            "image": "nginx",
                            "imagePullPolicy": "IfNotPresent",
                            "name": "nginx",
                            "ports": [
                                {
                                    "containerPort": 80,
                                    "name": "http",
                                    "protocol": "TCP"
                                }
                            ],
                            "resources": {},
                            "terminationMessagePath": "/dev/termination-log",
                            "terminationMessagePolicy": "File"
                        }
                    ],
                    "dnsPolicy": "ClusterFirst",
                    "restartPolicy": "Always",
                    "schedulerName": "default-scheduler",
                    "securityContext": {},
                    "terminationGracePeriodSeconds": 30
                }
            }
        },
        "status": {
            "availableReplicas": 2,
            "conditions": [
                {
                    "lastTransitionTime": "2020-03-24T07:50:54Z",
                    "lastUpdateTime": "2020-03-24T07:50:54Z",
                    "message": "Deployment has minimum availability.",
                    "reason": "MinimumReplicasAvailable",
                    "status": "True",
                    "type": "Available"
                },
                {
                    "lastTransitionTime": "2020-03-24T07:50:29Z",
                    "lastUpdateTime": "2020-03-24T07:50:54Z",
                    "message": "ReplicaSet \"nginx-deploy-78bc8876dd\" has successfully progressed.",
                    "reason": "NewReplicaSetAvailable",
                    "status": "True",
                    "type": "Progressing"
                }
            ],
            "observedGeneration": 1,
            "readyReplicas": 2,
            "replicas": 2,
            "updatedReplicas": 2
        }
    },
    "requestReceivedTimestamp": "2020-03-24T07:50:54.147551Z",
    "requestURI": "/apis/apps/v1/namespaces/default/deployments/nginx-deploy/status",
    "responseObject": {
        "apiVersion": "apps/v1",
        "kind": "Deployment",
        "metadata": {
            "annotations": {
                "deployment.kubernetes.io/revision": "1"
            },
            "creationTimestamp": "2020-03-24T07:50:28Z",
            "generation": 1,
            "name": "nginx-deploy",
            "namespace": "default",
            "resourceVersion": "1885354",
            "selfLink": "/apis/apps/v1/namespaces/default/deployments/nginx-deploy/status",
            "uid": "8a676d6d-2d41-49f2-8aac-e0030a2a28a8"
        },
        "spec": {
            "progressDeadlineSeconds": 600,
            "replicas": 2,
            "revisionHistoryLimit": 10,
            "selector": {
                "matchLabels": {
                    "name": "nginx"
                }
            },
            "strategy": {
                "rollingUpdate": {
                    "maxSurge": "25%",
                    "maxUnavailable": "25%"
                },
                "type": "RollingUpdate"
            },
            "template": {
                "metadata": {
                    "creationTimestamp": null,
                    "labels": {
                        "name": "nginx"
                    }
                },
                "spec": {
                    "containers": [
                        {
                            "image": "nginx",
                            "imagePullPolicy": "IfNotPresent",
                            "name": "nginx",
                            "ports": [
                                {
                                    "containerPort": 80,
                                    "name": "http",
                                    "protocol": "TCP"
                                }
                            ],
                            "resources": {},
                            "terminationMessagePath": "/dev/termination-log",
                            "terminationMessagePolicy": "File"
                        }
                    ],
                    "dnsPolicy": "ClusterFirst",
                    "restartPolicy": "Always",
                    "schedulerName": "default-scheduler",
                    "securityContext": {},
                    "terminationGracePeriodSeconds": 30
                }
            }
        },
        "status": {
            "availableReplicas": 2,
            "conditions": [
                {
                    "lastTransitionTime": "2020-03-24T07:50:54Z",
                    "lastUpdateTime": "2020-03-24T07:50:54Z",
                    "message": "Deployment has minimum availability.",
                    "reason": "MinimumReplicasAvailable",
                    "status": "True",
                    "type": "Available"
                },
                {
                    "lastTransitionTime": "2020-03-24T07:50:29Z",
                    "lastUpdateTime": "2020-03-24T07:50:54Z",
                    "message": "ReplicaSet \"nginx-deploy-78bc8876dd\" has successfully progressed.",
                    "reason": "NewReplicaSetAvailable",
                    "status": "True",
                    "type": "Progressing"
                }
            ],
            "observedGeneration": 1,
            "readyReplicas": 2,
            "replicas": 2,
            "updatedReplicas": 2
        }
    },
    "responseStatus": {
        "code": 200,
        "metadata": {}
    },
    "sourceIPs": [
        "192.168.3.2"
    ],
    "stage": "ResponseComplete",
    "stageTimestamp": "2020-03-24T07:50:54.149350Z",
    "user": {
        "groups": [
            "system:serviceaccounts",
            "system:serviceaccounts:kube-system",
            "system:authenticated"
        ],
        "uid": "02b810ff-860a-4803-ac63-43964c3656fa",
        "username": "system:serviceaccount:kube-system:deployment-controller"
    },
    "userAgent": "kube-controller-manager/v1.17.0 (linux/amd64) kubernetes/70132b0/system:serviceaccount:kube-system:deployment-controller",
    "verb": "update"
}
```
根据审计日志，看查看是谁(user)在什么时候(requestReceivedTimestamp)针对什么资源(objectRef)做了什么操作(update)。


官方审计相关的文档：
https://kubernetes.io/zh/docs/tasks/debug-application-cluster/audit/