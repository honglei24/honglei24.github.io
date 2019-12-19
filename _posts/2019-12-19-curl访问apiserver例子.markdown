---
layout: post
title:  "curl访问kube-apiserver"
date:   2019-12-19 09:08:00 +0800
categories: kubernetes apiserver
---

```
kubectl create serviceaccount api-explorer

cat <<EOF | kubectl create -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: log-reader
rules:
- apiGroups: [""] 
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
EOF

SERVICE_ACCOUNT=api-explorer
 
# Get the ServiceAccount's token Secret's name
SECRET=$(kubectl get serviceaccount ${SERVICE_ACCOUNT} -o json | jq -Mr '.secrets[].name | select(contains("token"))')
 
# Extract the Bearer token from the Secret and decode
TOKEN=$(kubectl get secret ${SECRET} -o json | jq -Mr '.data.token' | base64 -d)
 
# Extract, decode and write the ca.crt to a temporary location
kubectl get secret ${SECRET} -o json | jq -Mr '.data["ca.crt"]' | base64 -d > /tmp/ca.crt
 
# Get the API Server location
APISERVER=https://$(kubectl -n default get endpoints kubernetes --no-headers | awk '{ print $2 }')
```