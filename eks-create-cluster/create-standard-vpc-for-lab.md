---
created: 2022-04-10 22:12:29.404
last_modified: 2022-04-10 22:12:29.404
tags: aws/network/vpc aws/mgmt/cloudformation 
---
```ad-attention
title: This is a github note

```

## using cloudformation template 

```ad-note
title: using cloudshell

```

```sh
AWS_REGION=us-east-1
BUCKET_NAME=$(aws s3 mb s3://panlm-$RANDOM-$RANDOM |awk '{print $2}')

wget -O aws-vpc.template.yaml https://github.com/panlm/quickstart-aws-vpc/raw/main/templates/aws-vpc.template.yaml
aws s3 cp aws-vpc.template.yaml s3://${BUCKET_NAME}/

STACK_NAME=stack1-$RANDOM
CIDR="10.1"
aws cloudformation create-stack --stack-name ${STACK_NAME} \
  --parameters ParameterKey=AvailabilityZones,ParameterValue="${AWS_REGION}a\,${AWS_REGION}b" \
  ParameterKey=VPCCIDR,ParameterValue="${CIDR}.0.0/16" \
  ParameterKey=NumberOfAZs,ParameterValue=2 \
  ParameterKey=PublicSubnet1CIDR,ParameterValue="${CIDR}.1.0/24" \
  ParameterKey=PublicSubnet2CIDR,ParameterValue="${CIDR}.2.0/24" \
  ParameterKey=PrivateSubnet1ACIDR,ParameterValue="${CIDR}.3.0/24" \
  ParameterKey=PrivateSubnet1BCIDR,ParameterValue="${CIDR}.4.0/24" \
  ParameterKey=PrivateSubnet2ACIDR,ParameterValue="${CIDR}.5.0/24" \
  ParameterKey=PrivateSubnet2BCIDR,ParameterValue="${CIDR}.6.0/24" \
  --template-url https://${BUCKET_NAME}.s3.amazonaws.com/aws-vpc.template.yaml \
  --region ${AWS_REGION}

# until get CREATE_COMPLETE
while true ; do
  status=$(aws cloudformation --region ${AWS_REGION} describe-stacks --stack-name ${STACK_NAME} --query 'Stacks[0].StackStatus' --output text)
  echo ${status}
  if [[ ${status} == 'CREATE_IN_PROGRESS' ]]; then
    sleep 10
  else
    break
  fi
done

PublicSubnet1ID=$(aws cloudformation --region ${AWS_REGION} describe-stacks --stack-name ${STACK_NAME} --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet1ID`].OutputValue' --output text)

OWNER_ARN=$(aws sts get-caller-identity  --query 'Arn'  --output text)
ENV_ID=$(aws cloud9 create-environment-ec2 \
--name ${STACK_NAME} \
--instance-type t3.small \
--subnet-id ${PublicSubnet1ID} \
--automatic-stop-time-minutes 10080 \
--owner-arn ${OWNER_ARN} \
--query 'environmentId' --output text )

(C9_URL=https://${AWS_REGION}.console.aws.amazon.com/cloud9/ide/${ENV_ID}
echo "open cloud9 url:"
echo "${C9_URL}")

```

- no s3 endpoint
- security group named eks-shared-sg
- tag subnet 
    - `kubernetes.io/role/internal-elb` = `1`
    - `kubernetes.io/role/elb` = `1`
    - (option) `kubernetes.io/cluster/<vpc_name>` = `shared`
- verified in china region

## refer
- [[cloudformation-cli]]



