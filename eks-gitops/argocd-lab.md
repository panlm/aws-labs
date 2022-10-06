---
created: 2022-08-07 19:24:34.481
last_modified: 2022-08-07 19:24:34.481
tags: gitops gitops/argo 
---
```ad-attention
title: This is a github note

```
# argocd-lab
## install
[link](https://github.com/argoproj/argo-cd)

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.8/manifests/install.yaml
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.8/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
echo $ARGOCD_SERVER
export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure

```

## add helm
```sh
argocd repo add https://aws.github.io/eks-charts --type helm --name aws-eks-charts --project default

```

## install aws-lb-controller
### with argocd UI
```sh
cluster_name=ekscluster1
export AWS_DEFAULT_REGION=us-east-2

curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
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

```

![[Pasted image 20220807201104.png]]
![[Pasted image 20220807201116.png]]
![[Pasted image 20220807201128.png]]
![[Pasted image 20220807201157.png]]
![[Pasted image 20220807201215.png]]
![[Pasted image 20220807201250.png]]

最高可选版本 chart version 1.4.3 

### with argocd cli
```sh
cat >aws-lb-controller.yaml <<-EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: aws-lb-controller
spec:
  destination:
    name: ''
    namespace: kube-system
    server: 'https://kubernetes.default.svc'
  source:
    path: ''
    repoURL: 'https://aws.github.io/eks-charts'
    targetRevision: 1.4.3
    chart: aws-load-balancer-controller
    helm:
      parameters:
        - name: clusterName
          value: ekscluster1
        - name: serviceAccount.create
          value: 'false'
        - name: serviceAccount.name
          value: aws-load-balancer-controller
  project: default
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
EOF

kubectl apply -f aws-lb-controller.yaml -n argocd

```

## install echoserver with argocd
```sh
# argocd app create echoserver --repo https://github.com/kubernetes-sigs/aws-load-balancer-controller.git --path docs/examples/echoservice --dest-server https://kubernetes.default.svc --dest-namespace echoserver

argocd app create echoserver --repo https://github.com/panlm/aws-eks-example.git --path echoserver --dest-server https://kubernetes.default.svc --dest-namespace echoserver
argocd app sync apps
```

![[Pasted image 20220807210217.png]]

clone to your github 
- modify ingress file to update hostname
- modify deployment file to increase replicas

## install 2048 game
```sh
argocd app create game2048 --repo https://github.com/panlm/aws-eks-example.git --path 2048 --dest-server https://kubernetes.default.svc --dest-namespace game2048
argocd app sync apps
```

