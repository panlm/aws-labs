---
created: 2022-03-22 15:56:53.271
last_modified: 2022-03-22 15:56:53.271
tags: 
- aws/container/eks 
- aws/mgmt/cloudwatch 
- prometheus 
---
```ad-attention
title: This is a github note

```

# enable-prometheus-in-cloudwatch
## enable
1. service account
```sh
cluster_name=ekscluster1
eksctl create iamserviceaccount \
  --cluster $cluster_name \
  --namespace amazon-cloudwatch \
  --name cwagent-prometheus \
  --attach-policy-arn  arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --override-existing-serviceaccounts \
  --approve

output=prom.yaml
curl -o $output 'https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/service/cwagent-prometheus/prometheus-eks.yaml'
sed -i 's;amazon/cloudwatch-agent;public.ecr.aws/cloudwatch-agent/cloudwatch-agent;' $output
kubectl apply -f $output

```

## reference
[lab](https://www.eksworkshop.com/advanced/330_servicemesh_using_appmesh/add_nodegroup_fargate/cloudwatch_setup/#enable-prometheus-metrics-in-cloudwatch)


