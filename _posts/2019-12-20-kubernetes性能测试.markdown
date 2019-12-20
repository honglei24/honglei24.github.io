---
layout: post
title:  "kubernetes性能测试"
date:   2019-12-19 16:59:00 +0800
categories: kubernetes
---
## 背景
https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md
关于集群的各种阈值，社区有相关的讨论，但是并没有给出很详细的参考值。
社区本身是提供了一些工具来做测试的。
https://github.com/kubernetes/perf-tests/。
下面主要说明clusterloader2的用法。

## 编译
前提需要预先安装go,这里省略。本次测试用的go-1.12.12
```
$ go version
go version go1.12.12 linux/amd64

$ cd $(go env GOPATH)/src/k8s.io/
$ git clone https://github.com/kubernetes/perf-tests.git

$ cd perf-tests/clusterloader2
$ go build -o clusterloader './cmd/'
```

创建测试配置文件
```
$ cat > clusterload.conf <<EOF
CLUSTERLOADER_BIN="/home/hl/go/src/k8s.io/perf-tests/clusterloader2/clusterloader"

# kube config for kubernetes api
KUBE_CONFIG=${HOME}/.kube/config

# Provider setting
# Supported provider for xiaomi: local, kubemark, lvm-local, lvm-kubemark
#PROVIDER='kubemark'
PROVIDER='local'

# SSH config for metrics' collection
KUBE_SSH_KEY_PATH=$HOME/.ssh/id_rsa
KUBEMARK_SSH_KEY=$HOME/.ssh/id_rsa
MASTER_SSH_IP=192.168.122.169
MASTER_SSH_USER_NAME=master

# Clusterloader2 testing strategy config paths
# It supports setting up multiple test strategy. Each testing strategy is individual and serial.
TEST_CONFIG='/home/hl/go/src/k8s.io/perf-tests/clusterloader2/testing/density/config.yaml'

# Clusterloader2 testing override config paths
# It supports setting up multiple override config files. All of override config files will be applied to each testing strategy.
# OVERRIDE_CONFIG='testing/density/override/200-nodes.yaml'

# Log config
REPORT_DIR='./reports'
LOG_FILE='logs/tmp.log'
EOF

$ mkdir -p logs 
$ touch logs/tmp.log
$ $CLUSTERLOADER_BIN --kubeconfig=$KUBE_CONFIG     --provider=$PROVIDER     --masterip=$MASTER_SSH_IP --mastername=$MASTER_SSH_USER_NAME     --testconfig=$TEST_CONFIG     --report-dir=$REPORT_DIR     --alsologtostderr 2>&1 | tee $LOG_FILE
```

执行完成后就在logs/tmp.log可一看到报告。内容如下：
```
......
I1220 18:14:09.806828   25424 phase_latency.go:127] PodStartupLatency: perc50: 1.461648398s, perc90: 2.286786988s, perc99: 2.286786988s
I1220 18:14:09.806842   25424 phase_latency.go:122] PodStartupLatency: 2 worst pod_startup latencies: [{test-teclb6-1/latency-deployment-0-6f797dbfc7-lqz7n 1.628902073s} {test-teclb6-1/latency-deployment-1-784c74c8b5-2b2lz 2.639642901s}]
I1220 18:14:09.806859   25424 phase_latency.go:127] PodStartupLatency: perc50: 1.628902073s, perc90: 2.639642901s, perc99: 2.639642901s; threshold 5s
......
I1220 18:14:35.569745   25424 simple_test_executor.go:361] Resources cleanup time: 10.009821438s
I1220 18:14:35.569780   25424 clusterloader.go:187] --------------------------------------------------------------------------------
I1220 18:14:35.569790   25424 clusterloader.go:188] Test Finished
I1220 18:14:35.569798   25424 clusterloader.go:189]   Test: /home/hl/go/src/k8s.io/perf-tests/clusterloader2/testing/density/config.yaml
I1220 18:14:35.569807   25424 clusterloader.go:190]   Status: Success
I1220 18:14:35.569815   25424 clusterloader.go:194] --------------------------------------------------------------------------------

```

## 问题
* 执行工程中报错："unexpected error (code: 52) in ssh connection to master: <nil>", 在代码里搜索了一下家了两行打印日志，发现是代码里获取etcd的metrics的端口是硬编码，用的2379端口，而测试集群的etcd的metrics端口用的是2381,所以修改了一下代码，重新编译得到新的二进制之后，再次测试。成功执行玩测试流，当然也可以修改集群配置来适应代码。
```
diff --git a/clusterloader2/pkg/measurement/common/etcd_metrics.go b/clusterloader2/pkg/measurement/common/etcd_metrics.go
index ada4169d..1e0c770b 100644
--- a/clusterloader2/pkg/measurement/common/etcd_metrics.go
+++ b/clusterloader2/pkg/measurement/common/etcd_metrics.go
@@ -195,13 +195,14 @@ func (e *etcdMetricsMeasurement) getEtcdMetrics(host, provider string) ([]*model
        }
 
        // Use old endpoint if new one fails.
-       return e.sshEtcdMetrics("curl http://localhost:2379/metrics", host, provider)
+       return e.sshEtcdMetrics("curl http://localhost:2381/metrics", host, provider)
 }
```

## 思考
* clusterloader2的测试可选配置还是很多的，具体可以参考clusterloader2/testing目录下定义的各种yaml文件。
* 根据实际的环境，很多测试参数需要调整，可以根据实际的节点性能来调整测试的策略。比如我的测试环境是在本地搭建的虚拟机，在做饱和度测试的时候，我就需要调小一些参数。
* 测试环境的机器有限，如果想模拟大规模集群环境，可以使用kubemark来模拟，kubemark的使用还需要再完善。

## 参考
https://www.jianshu.com/p/eb8a58281f95