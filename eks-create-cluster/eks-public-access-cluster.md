---
created: 2022-05-21 12:43:38.021
last_modified: 2022-05-21 12:43:38.021
tags: aws/container/eks 
---
```ad-attention
title: This is a github note

```
# eks-public-access-cluster
## prep
- do not need to create vpc in advance
- [[setup-cloud9-for-eks]] or using your local environment

## cluster yaml
- dont put `subnets`/`sharedNodeSecurityGroup` in your `vpc` section. eksctl will create a clean vpc for you
- dont use `privateCluster` section, you could make cluster api server endpoint `public` or `public and private`
- you still could put your group node in private subnet for security consideration
- recommend for most of poc environment

```sh
# get optimized eks ami id for your version & region
EKS_VERSION=1.23
AWS_REGION=us-east-2
aws ssm get-parameter --name /aws/service/eks/optimized-ami/${EKS_VERSION}/amazon-linux-2/recommended/image_id --region ${AWS_REGION} --query "Parameter.Value" --output text

```

`c1.yaml`
```yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ekscluster1 # MODIFY cluster name, have another one in nodeGroup section
  region: "us-east-2" # MODIFY region
  version: "1.23" # MODIFY version

availabilityZones: ["us-east-2a", "us-east-2b", "us-east-2c"]

# REPLACE THIS CODE BLOCK
# vpc:
#   subnets:
#     private:
#       us-east-2a:
#         id: subnet-xxxxxxxx
#       us-east-2b:
#         id: subnet-xxxxxxxx
#     public:
#       us-east-2a:
#         id: subnet-xxxxxxxx
#       us-east-2b:
#         id: subnet-xxxxxxxx
#   sharedNodeSecurityGroup: sg-xxxxxxxx
vpc:
  cidr: "10.251.0.0/16"
  clusterEndpoints:
    privateAccess: true
    publicAccess: true

cloudWatch:
  clusterLogging:
    enableTypes: ["*"]

# secretsEncryption:
#   keyARN: ${MASTER_ARN}

managedNodeGroups:
- name: managed-ng
  minSize: 1
  maxSize: 5
  desiredCapacity: 1
  instanceType: m5.large
  ssh:
    enableSsm: true
  privateNetworking: true

nodeGroups:
- name: ng1
  minSize: 1
  maxSize: 5
  desiredCapacity: 1
  instanceType: m5.large
  ssh:
    enableSsm: true
  privateNetworking: true
  ami: "ami-06acc29adf84c8f3c"
  amiFamily: AmazonLinux2
  overrideBootstrapCommand: |
    #!/bin/bash
    source /var/lib/cloud/scripts/eksctl/bootstrap.helper.sh
    /etc/eks/bootstrap.sh ${CLUSTER_NAME} --container-runtime containerd --kubelet-extra-args "--node-labels=${NODE_LABELS}"

addons:
- name: vpc-cni 
  version: latest
- name: coredns
  version: latest # auto discovers the latest available
- name: kube-proxy
  version: latest

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: ebs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      ebsCSIController: true
  - metadata:
      name: efs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      efsCSIController: true
  - metadata:
      name: cloudwatch-agent
      namespace: amazon-cloudwatch
    attachPolicyARNs:
    - "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
  - metadata:
      name: fluent-bit
      namespace: amazon-cloudwatch
    attachPolicyARNs:
    - "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"

```

```sh
eksctl create cluster -f c1.yaml
```

## default tags on subnet
- [[eksctl-default-tags-on-subnet]]

## network topo preview
- [[security-group-for-eks-deepdive]]

## refer
- [[eks-private-access-cluster]]
- [[eks-nodegroup]]


