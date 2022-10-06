---
created: 2022-06-24 14:41:33.643
last_modified: 2022-06-24 14:41:33.643
tags: aws/storage/ebs  aws/container/eks 
---
```ad-attention
title: This is a github note

```
# ebs-for-eks

## install
```sh
export AWS_REGION=cn-northwest-1
export CLUSTER_NAME=eks0630
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
kubectl apply -k aws-ebs-csi-driver/deploy/kubernetes/overlays/stable

# verify pod running
kubectl get pods -n kube-system

```

## assign policy to node
```sh
# # (option) using customer managed policy
# aws iam create-policy \
#     --policy-name Amazon_EBS_CSI_Driver \
#     --policy-document file://./aws-ebs-csi-driver/docs/example-iam-policy.json \
#     --region ${AWS_REGION}
# POLICY_NAME=$(aws iam list-policies \
#   --query 'Policies[?PolicyName==`Amazon_EBS_CSI_Driver`].Arn' \
#   --output text --region ${AWS_REGION} )

# using aws managed policy
POLICY_ARN=$(aws iam list-policies \
  --query 'Policies[?PolicyName==`AmazonEBSCSIDriverPolicy`].Arn' \
  --output text --region ${AWS_REGION} )
# check detail permission in this policy
aws iam get-policy-version --policy-arn ${POLICY_ARN} --version-id v1

# get vpc id
VPC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} --region ${AWS_REGION} \
  --query "cluster.resourcesVpcConfig.vpcId" --output text )
# get nodegroups' instance profiles
TAG=tag:kubernetes.io/cluster/${CLUSTER_NAME}
INSTANCE_PROFILES=($(aws ec2 describe-instances \
  --filters "Name=${TAG},Values=owned" "Name=vpc-id,Values=${VPC_ID}"\
  |jq -r '.Reservations[].Instances[].IamInstanceProfile.Arn' ) )
# get role arns for instance profiles
ROLE_ARNS=($(for i in ${INSTANCE_PROFILES[@]}; do
  aws iam get-instance-profile \
    --instance-profile-name ${i##*/} |jq -r '.InstanceProfile.Roles[0].Arn'
done |sort -u ))
echo ${ROLE_ARNS[@]}

# attach policy to role
for i in ${ROLE_ARNS[@]}; do
  aws iam attach-role-policy --policy-arn ${POLICY_ARN} \
    --role-name ${i##*/} --region ${AWS_REGION}
done

```

## verify
```
kubectl apply -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/manifests/

```

1. create sc/pvc
2. pv will be created when pod run
3. write something in pod

### cross az pod definition
- check pv status
- check pod on which az
- delete pod and launch to another az
- check pod status pending
- kill pod and launch back to original az

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
#  nodeSelector:
#    topology.kubernetes.io/zone: cn-northwest-1b
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-claim
  terminationGracePeriodSeconds: 0

```


## check log
```sh
k logs -f deploy/ebs-csi-controller csi-provisioner -n kube-system 

```

###  ebs-csi-pod has 6 container
- https://www.velotio.com/engineering-blog/kubernetes-csi-in-action-explained-with-features-and-use-cases

- ebs-plugin 
- csi-provisioner 
- csi-attacher 
- csi-snapshotter 
- csi-resizer 
- liveness-probe


