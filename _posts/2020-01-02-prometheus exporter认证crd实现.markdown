---
layout: post
title:  "prometheus exporter认证crd实现"
date:   2020-01-02 15:16:00 +0800
categories: kubernetes
---

```
cd /home/hl/go/src/github.com
$ mkdir -p honglei24/exporter-controller
$ mkdir -p honglei24/exporter-controller/pkg/client
$ mkdir -p honglei24/exporter-controller/pkg/apis/myexport/v1
$ cd honglei24/exporter-controller/pkg/apis/myexport/v1
$ touch doc.go types.go regsiter.go
$ for name in *.go ; do echo 'package v1' >$name; done


$ cd ~/go/src/k8s.io/code-generator/
./generate-groups.sh all github.com/honglei24/exporter-controller/pkg/client github.com/honglei24/exporter-cokubebuilder init --domain my.domainntroller/pkg/apis "myexporter:v1"


./generate-groups.sh all github.com/trstringer/k8s-controller-custom-resource/pkg/client github.com/trstringer/k8s-controller-custom-resource/pkg/apis "myresource:v1"
trstringer/k8s-controller-custom-resource

./generate-groups.sh all gitlab.oneitfarm.com/hl/exporter-controller/pkg/client gitlab.oneitfarm.com/hl/exporter-controller/pkg/apis "myexport:v1"



gitlab.oneitfarm.com/hl/exporter-controller
kubebuilder create api --group base_auth --version v1 --kind Exporter


```