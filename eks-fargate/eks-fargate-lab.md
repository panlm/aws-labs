---
created: 2022-06-25 09:17:05.835
last_modified: 2022-06-25 09:17:05.835
tags: aws/container/eks aws/container/fargate 
---
```ad-attention
title: This is a github note

```

# eks-fargate-lab

```toc
```

## 环境准备

- 登录你的实验环境 ([LINK](https://dashboard.eventengine.run/login))
- 进入 aws cloud9 ([LINK](https://console.aws.amazon.com/cloud9))，打开已经准备好的 `EKSLabIDE` 桌面
- 安装必要的软件
```sh
# install others
sudo yum -y install jq gettext bash-completion moreutils wget

# install kubectl with +/- 1 cluster version 1.23.12 / 1.22.15 / 1.24.6
# sudo curl --location -o /usr/local/bin/kubectl "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo curl --silent --location -o /usr/local/bin/kubectl "https://storage.googleapis.com/kubernetes-release/release/v1.24.6/bin/linux/amd64/kubectl"

# 1.22.x version of kubectl
# sudo curl --silent --location -o /usr/local/bin/kubectl "https://storage.googleapis.com/kubernetes-release/release/v1.22.11/bin/linux/amd64/kubectl"

sudo chmod +x /usr/local/bin/kubectl

kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
alias k=kubectl 
complete -F __start_kubectl k
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc

# install awscli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
echo A |unzip awscliv2.zip
sudo ./aws/install --update

# install eksctl
# consider install eksctl version 0.89.0
# if you have older version yaml 
# https://eksctl.io/announcements/nodegroup-override-announcement/
curl --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
# wget https://github.com/weaveworks/eksctl/releases/download/v0.89.0/eksctl_Linux_amd64.tar.gz

eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion

# helm 3.8.2 (helm 3.9.0 will have issue #10975)
#curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
tar xf helm-v3.8.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version --short

# install aws-iam-authenticator
wget -O aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64
chmod +x ./aws-iam-authenticator
sudo mv ./aws-iam-authenticator /usr/local/bin/

# option
# install jwt-cli
# https://github.com/mike-engel/jwt-cli/blob/main/README.md
# sudo yum -y install cargo
# cargo install jwt-cli

# install flux & fluxctl
curl -s https://fluxcd.io/install.sh | sudo bash
flux -v
. <(flux completion bash)

# sudo wget -O /usr/local/bin/fluxctl $(curl https://api.github.com/repos/fluxcd/flux/releases/latest | jq -r ".assets[] | select(.name | test(\"linux_amd64\")) | .browser_download_url")
# sudo chmod 755 /usr/local/bin/fluxctl
# fluxctl version
# fluxctl identity --k8s-fwd-ns flux

# install ssm session plugin
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm

```

- 配置 credential ，你的桌面已经被分配 `eksworkshop-admin` 角色，禁用本地 credential 即可以使用节点的 role
```sh
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials

```

- 更新 kubeconfig 文件
```sh
export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

eksctl utils  write-kubeconfig --cluster eksworkshop-eksctl

kubectl get no -o wide

```

- 如果可以正常显示节点信息，表示环境已经就绪。

## create fargate profile

- 在现有集群中添加 fargate 支持

```sh
CLUSTER_NAME=eksworkshop-eksctl
AWS_REGION=${AWS_DEFAULT_REGION}
NAMESPACE=game-2048

# pods in namespace called `game-2048` will be deployed to fargate profile
eksctl create fargateprofile \
  --cluster ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --name ${NAMESPACE} \
  --namespace ${NAMESPACE}

eksctl get fargateprofile \
  --cluster ${CLUSTER_NAME} \
  -o yaml

```

- 你可以登录 eks 管理界面确认 fargate profile 创建成功 
- 截个图

## install aws load balancer controller

- 接下来我们将安装一个应用，并且对外发布，这里需要用到 aws load balancer controller
    - refer [[aws-load-balancer-controller#install]]

```sh
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${CLUSTER_NAME} \
    --approve

# china region link
# wget -O iam_policy.json https://github.com/kubernetes-sigs/aws-load-balancer-controller/raw/main/docs/install/iam_policy_cn.json
# global region link
wget -O iam_policy.json https://github.com/kubernetes-sigs/aws-load-balancer-controller/raw/main/docs/install/iam_policy.json
POLICY_ARN=$(aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy-$RANDOM \
    --policy-document file://iam_policy.json |jq -r '.Policy.Arn' )
echo ${POLICY_ARN}

eksctl create iamserviceaccount \
  --cluster ${CLUSTER_NAME} \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn ${POLICY_ARN} \
  --override-existing-serviceaccounts \
  --approve

kubectl get sa aws-load-balancer-controller -n kube-system -o yaml

kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts

helm upgrade -i aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=${CLUSTER_NAME} \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller

kubectl -n kube-system rollout status deployment aws-load-balancer-controller

```

- 确认 `aws-load-balancer-controller` 部署成功
```sh
kubectl get deploy aws-load-balancer-controller -n kube-system

```

## deploy game 2048 

- 部署应用

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml

```

- 观察应用运行在fargate节点上
```sh
k get all -n $NAMESPACE -o wide
k get no  -o wide

```

- 尝试从公网访问应用
```sh
kubectl get ing -n ${NAMESPACE} -o=custom-columns="URL":.status.loadBalancer.ingress[*].hostname

```

- 打开浏览器访问应用URL
```sh
kubectl get ing -n ${NAMESPACE} -o=custom-columns="URL":.status.loadBalancer.ingress[*].hostname

```




