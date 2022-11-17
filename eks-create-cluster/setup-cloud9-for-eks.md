---
created: 2022-05-21 12:46:05.435
last_modified: 2022-10-17 08:17:09.959
tags: aws/container/eks aws/cloud9 
---
```ad-attention
title: This is a github note

```
# setup-cloud9-for-eks
```toc
```

## need a cloud9 instance in your region
click [here](https://us-east-2.console.aws.amazon.com/cloudshell) to run cloud shell and execute code block, and go to your region and open cloud9

```sh
export AWS_DEFAULT_REGION=us-east-2 # need put each command

DEFAULT_VPC=$(aws ec2 describe-vpcs \
  --filter Name=is-default,Values=true \
  --query 'Vpcs[0].VpcId' --output text \
  --region ${AWS_DEFAULT_REGION})

if [[ ! -z ${DEFAULT_VPC} ]]; then
  FIRST_SUBNET=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=${DEFAULT_VPC}" \
    --query "Subnets[?AvailabilityZone=='"${AWS_DEFAULT_REGION}a"'].SubnetId" \
    --output text \
    --region ${AWS_DEFAULT_REGION})
  aws cloud9 create-environment-ec2 \
    --name cloud9-$RANDOM \
    --instance-type t3.small \
    --subnet-id ${FIRST_SUBNET} \
    --automatic-stop-time-minutes 10080 \
    --region ${AWS_DEFAULT_REGION}
else
  echo "you have no default vpc in $AWS_DEFAULT_REGION"
fi

```

## install in cloud9 
1. resize disk - [[cloud9-resize-instance-volume-script]]
2. disable temporary credential from settings and delete `aws_session_token=` line in `~/.aws/credentials`
3. install dependencies
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
sudo yum -y install cargo
cargo install jwt-cli
sudo ln -sf ~/.cargo/bin/jwt /usr/local/bin/jwt

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

4. resize cloud9 disk
5. disable cloud9 aws credential management
```sh
if [[ -c /dev/nvme0 ]]; then
  wget -qO- https://github.com/amazonlinux/amazon-ec2-utils/raw/main/ebsnvme-id >/tmp/ebsnvme-id
  VOLUME_ID=$(sudo python3 /tmp/ebsnvme-id -v /dev/nvme0 |awk '{print $NF}')
  DEVICE_NAME=/dev/nvme0n1
else
  C9_INST_ID=$(curl 169.254.169.254/latest/meta-data/instance-id)
  VOLUME_ID=$(aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=${C9_INST_ID} --query "Volumes[0].VolumeId" --output text)
  DEVICE_NAME=/dev/xvda
fi

aws ec2 modify-volume --volume-id ${VOLUME_ID} --size 60
sleep 10
sudo growpart ${DEVICE_NAME} 1
sudo xfs_growfs -d /

# ---

aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials

```

6. 分配管理员role到instance。（直接执行下列步骤可能遇到权限不够的告警）。
- 如果你有workshop的Credentials，直接先复制粘贴到命令行，再执行下列步骤
- 或者如果自己账号的cloud9，先用 `aws configure` 配置aksk

```sh
AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
C9_INST_ID=$(curl 169.254.169.254/latest/meta-data/instance-id)
ROLE_NAME=adminrole-$RANDOM
cat > trust.json <<-EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF
aws iam create-role --role-name ${ROLE_NAME} \
  --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name ${ROLE_NAME} \
  --policy-arn "arn:aws:iam::aws:policy/AdministratorAccess"

instance_profile_arn=$(aws ec2 describe-iam-instance-profile-associations \
  --filter Name=instance-id,Values=$C9_INST_ID \
  --query IamInstanceProfileAssociations[0].IamInstanceProfile.Arn \
  --output text)
if [[ ${instance_profile_arn} == "None" ]]; then
  # create one
  aws iam create-instance-profile \
    --instance-profile-name ${ROLE_NAME}
  sleep 10
  # attach role to it
  aws iam add-role-to-instance-profile \
    --instance-profile-name ${ROLE_NAME} \
    --role-name ${ROLE_NAME}
  sleep 10
  # attach instance profile to ec2
  aws ec2 associate-iam-instance-profile \
    --iam-instance-profile Name=${ROLE_NAME} \
    --instance-id ${C9_INST_ID}
else
  existed_role_name=$(aws iam get-instance-profile \
    --instance-profile-name ${instance_profile_arn##*/} \
    --query 'InstanceProfile.Roles[0].RoleName' \
    --output text)
  aws iam attach-role-policy --role-name ${existed_role_name} \
    --policy-arn "arn:aws:iam::aws:policy/AdministratorAccess"
fi

```

7. 在 cloud9 中，重新打开一个 terminal 窗口，并验证权限符合预期。如果权限又问题，请手工移除 instance profile ，再 assign 。
```sh
aws sts get-caller-identity

```



## reference
- https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-aws-iam-authenticator.html
- [[switch-role-to-create-dedicate-cloud9]]

## Turn off AWS managed temporary credentials 
[LINK](https://docs.aws.amazon.com/cloud9/latest/user-guide/security-iam.html#auth-and-access-control-temporary-managed-credentials)

If you turn off AWS managed temporary credentials, by default the environment cannot access any AWS services, regardless of the AWS entity who makes the request. If you can't or don't want to turn on AWS managed temporary credentials for an environment, but you still need the environment to access AWS services, consider the following alternatives:

- Attach an instance profile to the Amazon EC2 instance that connects to the environment. For instructions, see [Create and Use an Instance Profile to Manage Temporary Credentials](https://docs.aws.amazon.com/cloud9/latest/user-guide/credentials.html#credentials-temporary).
- Store your permanent AWS access credentials in the environment, for example, by setting special environment variables or by running the `aws configure` command. For instructions, see [Create and store permanent access credentials in an Environment](https://docs.aws.amazon.com/cloud9/latest/user-guide/credentials.html#credentials-permanent-create).



