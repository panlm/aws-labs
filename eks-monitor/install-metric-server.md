---
created: 2022-07-04 13:54:12.092
last_modified: 2022-07-04 13:54:12.092
tags: aws/container/eks kubernetes 
---
```ad-attention
title: This is a github note

```

# install-metric-server
https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/metrics-server.html

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl get deployment metrics-server -n kube-system

```





