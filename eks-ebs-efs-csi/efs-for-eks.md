---
created: 2022-05-23 09:57:50.932
last_modified: 2022-05-23 09:57:50.932
tags: aws/storage/efs aws/container/eks 
---
```ad-attention
title: This is a github note

```
# efs-for-eks
- [efs workshop](https://www.eksworkshop.com/beginner/190_efs/launching-efs/)
- [[efs-on-eks-mini-priviledge]]
- [Introducing Amazon EFS CSI dynamic provisioning](https://aws.amazon.com/blogs/containers/introducing-efs-csi-dynamic-provisioning/)

## create efs
```sh
CLUSTER_NAME=eks0630
AWS_REGION=cn-northwest-1

VPC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} --region ${AWS_REGION} \
  --query "cluster.resourcesVpcConfig.vpcId" --output text )
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids ${VPC_ID} \
  --query "Vpcs[].CidrBlock"  --region ${AWS_REGION} --output text )

# create security group
SG_ID=$(aws ec2 create-security-group --description ${CLUSTER_NAME}-efs-eks-sg \
  --group-name efs-sg-$RANDOM --vpc-id ${VPC_ID} |jq -r '.GroupId' )
# allow tcp 2049 (nfs v4)
aws ec2 authorize-security-group-ingress --group-id ${SG_ID}  --protocol tcp --port 2049 --cidr ${VPC_CIDR}

# create efs
FILESYSTEM_ID=$(aws efs create-file-system \
  --creation-token ${CLUSTER_NAME} \
  --region ${AWS_REGION} |jq -r '.FileSystemId' )
echo ${FILESYSTEM_ID}

# create mount target
TAG=tag:kubernetes.io/role/internal-elb
SUBNETS=($(aws eks describe-cluster --name ${CLUSTER_NAME} \
  --region ${AWS_REGION} |jq -r '.cluster.resourcesVpcConfig.subnetIds[]'))
PRIV_SUBNETS=($(aws ec2 describe-subnets --filters "Name=${TAG},Values=1" \
  --subnet-ids ${SUBNETS[@]} |jq -r '.Subnets[].SubnetId' ) )
for i in ${PRIV_SUBNETS[@]} ; do
  echo "creating mount target in: " $i
  aws efs create-mount-target --file-system-id ${FILESYSTEM_ID} \
    --subnet-id ${i} --security-group ${SG_ID}
done

```

## install from github
直接安装不额外配置权限的话，只能验证静态制备
如果验证动态制备，会有权限不够的告警，因为需要动态创建access point，可以通过节点role方式加载权限，或者重新部署为irsa

```sh
git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
kubectl apply -k ./aws-efs-csi-driver/deploy/kubernetes/overlays/stable

# verify pod running
kubectl get pods -n kube-system

# verify version
pod1=$(kubectl get pod -n kube-system -l app=efs-csi-controller |tail -n 1 |awk '{print $1}')
kubectl exec -it ${pod1} -n kube-system -- mount.efs --version

```

### uninstall efs-csi
```sh
cd aws-efs-csi-driver/deploy/kubernetes/overlays/stable
kubectl kustomize |kubectl delete -f -
```


## install from helm
### node role (alternative)
```sh
wget -O iam-policy.json 'https://github.com/kubernetes-sigs/aws-efs-csi-driver/raw/master/docs/iam-policy-example.json'
# cp aws-efs-csi-driver/docs/iam-policy-example.json iam-policy.json

## Create an IAM policy 
POLICY_ARN=$(aws iam create-policy \
  --policy-name EFSCSIControllerIAMPolicy-$RANDOM \
  --policy-document file://iam-policy.json |jq -r '.Policy.Arn' )
echo ${POLICY_ARN}

## attach policy to node role

```

### sa role
[blog](https://aws.amazon.com/blogs/containers/introducing-efs-csi-dynamic-provisioning/)
```sh

# do steps in **node role** part

CLUSTER_NAME=eks0630
AWS_REGION=cn-northwest-1

eksctl utils associate-iam-oidc-provider \
  --cluster ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --approve
eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --namespace=kube-system \
  --name=efs-csi-controller-sa \
  --override-existing-serviceaccounts \
  --attach-policy-arn=${POLICY_ARN} \
  --approve

```

### install
```sh
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
helm repo update

# get url for your region 
# https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html
# add `.cn` postfix for china region
# REGISTRY=602401143452.dkr.ecr.us-east-1.amazonaws.com
REGISTRY=961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set image.repository=${REGISTRY}/eks/aws-efs-csi-driver \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa

```
find registry url from [[eks-container-image-registries-url-by-region]]

### uninstall
```sh
helm uninstall aws-efs-csi-driver -n kube-system
```


## static provisioning
[link](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/multiple_pods)
using static provision with mount, no access point needed, so no additional iam policy

```sh
cat > storageclass.yaml <<-EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc-mnt
provisioner: efs.csi.aws.com
EOF

# ensure volumeHandle has been replaced by your efs filesystem id
envsubst > pv.yaml <<-EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv-mnt
spec:
  capacity:
    storage: 6Gi             # equal or bigger pvc, but it's a soft limit
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc-mnt
  csi:
    driver: efs.csi.aws.com
    volumeHandle: ${FILESYSTEM_ID}
EOF

cat > pvc.yaml <<-EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim-mnt
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc-mnt
  resources:
    requests:
      storage: 5Gi           # it's a soft limit
EOF

echo '---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app1
spec:
  containers:
  - name: efs-app1
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out1.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: efs-claim-mnt
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app2
spec:
  containers:
  - name: efs-app2
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out2.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: efs-claim-mnt
' |tee pod.yaml


kubectl apply -f storageclass.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml

```

## dynamic provisioning with efs access point 
[link](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning)
need additional iam policy, execute `node role` part, or reinstall with irsa

```sh
# ensure volumeHandle has been replaced by your efs filesystem id
envsubst > ./dy-storageclass.yaml <<-EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${FILESYSTEM_ID}
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional
EOF

echo '---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi                      # soft limit
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app-dy1
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out1; sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app-dy2
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out2; sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim
' |tee ./dy-pod.yaml

kubectl apply -f dy-storageclass.yaml
kubectl apply -f dy-pod.yaml

```

## check log
```sh
kubectl logs deployment/efs-csi-controller -n kube-system -c efs-plugin
kubectl logs daemonset/efs-csi-node -n kube-system -c efs-plugin

```


## reference
- [Amazon EFS quotas and limits](https://docs.aws.amazon.com/efs/latest/ug/limits.html)

### redeploy deployment and daemonset
```sh
kubectl rollout restart deploy efs-csi-controller -n kube-system
kubectl rollout restart ds efs-csi-node -n kube-system

```


