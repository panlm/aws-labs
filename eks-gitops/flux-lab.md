---
created: 2022-04-06 19:28:05.118
last_modified: 2022-09-14 08:40:30.375
tags: aws/container/eks gitops gitops/weaveworks/flux
---
```ad-attention
title: This is a github note

```
# flux-lab
## bootstrap v2
```sh
export GITHUB_TOKEN=ghp_xxxxxx
flux check --pre
flux bootstrap github \
    --owner=panlm \
    --repository=aws-eks-config \
    --branch=main \
    --personal \
    --path=clusters/ekscluster2

```

## lab
```sh
flux get all

```

### generate yaml
`cat value.yaml`
```yaml
clusterName: ekscluster3
serviceAccount:
  create: false
  name: aws-load-balancer-controller
```

```sh
flux create source helm ww-gitops \
  --url=https://aws.github.io/eks-charts \
  --export > a.yaml

flux create helmrelease aws-load-balancer-controller \
  --source=HelmRepository/ww-gitops \
  --chart=aws-load-balancer-controller \
  --chart-version 1.4.4 --values value.yaml --export > c.yaml
```

## helm v1 (alternative)
```sh
CLUSTER_NAME=ekscluster1
kubectl create ns flux
helm repo add fluxcd https://charts.fluxcd.io 

helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:panlm/aws-eks-config \
--set git.branch=main \
--set git.path=clusters/${CLUSTER_NAME} \
--namespace flux

helm upgrade -i helm-operator fluxcd/helm-operator \
--set helm.versions=v3 \
--set git.ssh.secretName=flux-git-deploy \
--set git.branch=main \
--namespace flux

kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
# put public key to repo's `Deploy keys`

```

## dependencies
```
```

## others
https://www.eksworkshop.com/intermediate/260_weave_flux/

got error:
```
[Container] 2022/04/06 07:19:56 Running command curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -  

/codebuild/output/tmp/script.sh: 4: /codebuild/output/tmp/script.sh: sudo: not found
```

try to find `build details` in `image-codepipeline` `build project`
modify `buildspec`
remove `sudo` before `apt-key add`



