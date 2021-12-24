---
layout: post
title:  "在k8s集群中安装kafka-exporter"
date:   2021-12-24 15:00:00 +0800
categories: kubernetes kafka prometheus
---

## 安装operator
```
# cat > kafka-exporter.yaml<<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kafka-exporter
  namespace: zookeeper
spec:
  selector:
    matchLabels:
      app: kafka-exporter
  template:
    metadata:
      labels:
        app: kafka-exporter
    spec:
      containers:
      - args:
        - -c
        - IPADDR=$(hostname -i); echo ${IPADDR}; /bin/kafka_exporter --kafka.server=${IPADDR}:9092
          --sasl.enabled --sasl.mechanism=scram-sha256 --sasl.username=admin --sasl.password=vss#JIANkong305
        command:
        - /bin/sh
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        image: danielqsj/kafka-exporter
        imagePullPolicy: IfNotPresent
        name: kafka-exporter
        resources: {}
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        vcn.ctyun.cn/kafka: "true"
EOF
# kubectl apply -f kafka-exporter.yaml
```
