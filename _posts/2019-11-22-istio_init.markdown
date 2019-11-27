---
layout: post
title:  "利用helm在k8s集群部署istio"
date:   2019-11-22 18:19:00 +0800
categories: kubernetes istio helm
---

**以下所有命令行操作都在master节点执行**

# istio部署
```
# helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.2.2/charts/
# helm install istio.io/istio-init --name istio-init --namespace istio-system
# helm install istio.io/istio --name istio --namespace istio-system --set gateways.istio-ingressgateway.type=NodePort

查看部署状态
# helm status istio-init
# helm status istio

# kubectl get crds | grep 'istio.io' | wc -l

```

部署bookinfo应用, 需要用到[bookinfo.yaml](https://github.com/honglei24/honglei24.github.io/appendix/kubernetes/istio/bookinfo.yaml)和[bookinfo-gateway.yaml](https://github.com/honglei24/honglei24.github.io/appendix/kubernetes/istio/bookinfo-gateway.yaml)
```
# kubectl label namespace default istio-injection=enabled
# kubectl get namespace -L istio-injection

# kubectl apply -f bookinfo.yaml
# kubectl get pods
# kubectl apply -f bookinfo-gateway.yaml
# export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
# export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
# export INGRESS_HOST=<k8s-node ip>
# export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

```
浏览器访问http://${GATEWAY_URL}/productpage，如果刷新几次应用的页面，就会看到页面中会随机展示 reviews 服务的不同版本的效果（红色、黑色的星形或者没有显示）。reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。
