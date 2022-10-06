---
created: 2022-05-21 13:18:53.303
last_modified: 2022-06-15 08:48:49.763
tags: aws/container/eks kubernetes/ingress 
---
```ad-attention
title: This is a github note

```
# aws-load-balancer-controller
## github
- https://github.com/kubernetes-sigs/aws-load-balancer-controller
- https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/how-it-works/

![[Pasted image 20220531150851.png|500]]

## workshop
- [[awslbc-ingress-lab-echoserver]]
- https://www.eksworkshop.com/beginner/180_fargate/prerequisites-for-alb/
- 常用ingress的相关配置 ([[awslbc-ingress-settings]])
- 使用已有ingress的相关配置 ([[awslbc-ingress-settings-ingress-group]])
- pod rediness gate ([link](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/pod_readiness_gate/))

## install
- [refer](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/installation/)

```sh
cluster_name=ekscluster1
export AWS_DEFAULT_REGION=us-east-2
eksctl utils associate-iam-oidc-provider \
  --cluster ${cluster_name} \
  --approve

#curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.1/docs/install/iam_policy.json

git clone https://github.com/kubernetes-sigs/aws-load-balancer-controller.git
cp aws-load-balancer-controller/docs/install/iam_policy.json .

policy_name=AWSLoadBalancerControllerIAMPolicy-$RANDOM
policy_arn=$(aws iam create-policy \
  --policy-name ${policy_name}  \
  --policy-document file://iam_policy.json \
  --query 'Policy.Arn' \
  --output text)

eksctl create iamserviceaccount \
  --cluster=${cluster_name} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name=${policy_name} \
  --attach-policy-arn=${policy_arn} \
  --override-existing-serviceaccounts \
  --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update

# following helm cmd will fail if you use 3.9.0 version
# downgrade to helm 3.8.2
# and another solved issue is here: [[ingress-controller-lab-issue]]
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${cluster_name} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 

kubectl get deployment -n kube-system aws-load-balancer-controller
```

[[awslbc-ingress-controller-lab-issue]]

### in china region
```sh
# using china region ecr url
helm upgrade -i aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=${cluster_name} \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set image.repository=961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/amazon/aws-load-balancer-controller \
    # --set region=${AWS_DEFAULT_REGION} \
    # --set vpcId=${VPC_ID} 

```

find registry url from [[eks-container-image-registries-url-by-region]]
using parameter `image.repository`  (refer [LINK](https://github.com/kubernetes-sigs/aws-load-balancer-controller/tree/main/helm/aws-load-balancer-controller))

## blog
- [[How To Expose Multiple Applications on Amazon EKS Using a Single Application Load Balancer]]


## slide
![[Pasted image 20220630150510.png]]


