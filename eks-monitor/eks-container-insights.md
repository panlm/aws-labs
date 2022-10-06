---
created: 2022-02-22 08:08:35.714
last_modified: 2022-06-23 15:43:53.895
tags: aws/container/eks aws/mgmt/cloudwatch 
---
```ad-attention
title: This is a github note

```

# eks-container-insights
## enable 
1. replace 2 service accounts with [CloudWatchAgentServerPolicy](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-prerequisites.htm)
```sh
CLUSTER_NAME=ekscluster1
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve

```

```sh
CLUSTER_NAME=ekscluster1
eksctl create iamserviceaccount \
    --name cloudwatch-agent \
    --namespace amazon-cloudwatch \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    --approve \
    --override-existing-serviceaccounts

eksctl create iamserviceaccount \
    --name fluent-bit \
    --namespace amazon-cloudwatch \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    --approve \
    --override-existing-serviceaccounts

```

2. [enable](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-quickstart.html) 
```sh
CLUSTER_NAME=ekscluster1
AWS_REGION=us-east-2
FluentBitHttpPort='2020'
FluentBitReadFromHead='On'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'

output=cwqs-1.yaml
curl -o $output https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml 

sed -i 's;amazon/cloudwatch-agent;public.ecr.aws/cloudwatch-agent/cloudwatch-agent;' $output
#sed -i 's;amazon/aws-for-fluent-bit:2.10.0;public.ecr.aws/aws-observability/aws-for-fluent-bit:2.28.0;' $output

cat $output | sed 's/{{cluster_name}}/'${CLUSTER_NAME}'/;s/{{region_name}}/'${AWS_REGION}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f - 

k get po -n amazon-cloudwatch

```

if you do 2 before 1, than need
- delete pods which use these service account
- check cloudtrail for "AccessDeny" events

## check pod / deployment log
- [workshop](https://www.eksworkshop.com/intermediate/250_cloudwatch_container_insights/viewlogs/)

