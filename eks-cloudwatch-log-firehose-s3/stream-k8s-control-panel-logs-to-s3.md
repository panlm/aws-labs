---
created: 2022-10-02 08:37:27.387
last_modified: 2022-10-02 19:50:39.960
tags: aws/container/eks aws/analytics/kinesis/firehose aws/storage/s3 aws/analytics/databrew 
---
```ad-attention
title: This is a github note

```
# stream-k8s-control-panel-logs-to-s3
```toc
style: bullet
min_depth: 2
max_depth: 4
```

## background
目前eks控制平面日志只支持发送到cloudwatch，且在同一个log group中有5种类型6种前缀的log stream的日志，不利于统一查询。且只有audit日志是json格式其他均是单行日志，且字段各不相同。

- kube-apiserver-audit
- kube-apiserver
- kube-scheduler
- authenticator
- kube-controller-manager
- cloud-controller-manager

## requirement
客户需求：
1. 简单 - 已有splunk日志平台，不希望使用opensearch等其他日志平台，保证运维简化
2. 实时 - 需要有方法将日志近实时地发送到S3，可以通过splunk进行查询和实时告警。export cloudwatch 日志的方式，实时性无法满足，且同样需要额外实现export端点续导的问题
3. 二次处理 - 未来可以实现对日志进行查询及关键字段提取方便进行分析和告警
4. 成本和安全 - 成本控制，高安全性，支持多账号

## architecture
![](stream-k8s-control-panel-logs-to-s3.png)

## lab
### eks cluster
- need an  eks cluster and enable log to cloudwatch

### s3
- 创建s3桶

```sh
bucket_name=centrallog-$RANDOM
s3_prefix="CentralizedAccountLogs"
aws s3 mb s3://${bucket_name}

athena_bucket_name=athena-$RANDOM
aws s3 mb s3://${athena_bucket_name}

```

### lambda
- 创建函数所需角色
- 下载定制代码
    - [[cloudwatch-to-firehose-python]]
    - [download](https://github.com/panlm/aws-labs/raw/main/eks-cloudwatch-log-firehose-s3/lambda_function.py)
- 创建函数并获取arn

```sh
lambda_name=${bucket_name}

# create lambda role
lambda_role_name=lambda-ex-$RANDOM
aws iam create-role --role-name ${lambda_role_name} --assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ${lambda_role_name} --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name ${lambda_role_name} --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaKinesisExecutionRole
lambda_role_arn=$(aws iam get-role --role-name ${lambda_role_name} |jq -r '.Role.Arn')

# download code and create lambda
wget -O lambda_function.py https://github.com/panlm/aws-labs/raw/main/eks-cloudwatch-log-firehose-s3/lambda_function.py
zip lambda_function.zip ./lambda_function.py

sleep 10
aws lambda create-function \
    --function-name ${lambda_name} \
    --runtime python3.8 \
    --timeout 60 \
    --zip-file fileb://lambda_function.zip \
    --handler lambda_function.lambda_handler \
    --role ${lambda_role_arn}

lambda_arn=$(aws lambda get-function \
--function-name ${lambda_name} \
--query 'Configuration.FunctionArn' --output text)

# download package and create lambda layer
wget -O package.zip https://github.com/panlm/aws-labs/raw/main/eks-cloudwatch-log-firehose-s3/package.zip

aws lambda publish-layer-version --layer-name layer_flatten_json --description "flatten_json" --zip-file fileb://package.zip --compatible-runtimes python3.8
layer_arn=$(aws lambda list-layer-versions --layer-name layer_flatten_json \
--query 'LayerVersions[0].LayerVersionArn' --output text)

# add layer to lambda
aws lambda update-function-configuration --function-name ${lambda_name} \
--layers ${layer_arn}

```

### firehose
- 创建firehose所需角色
- 创建firehose并且关联 transform 函数
- 获取arn

```sh
aws_region=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
firehose_name=${bucket_name}
role_name=${bucket_name}-$RANDOM
aws_account_id=$(aws sts get-caller-identity --query "Account" --output text)
firehose_name=${bucket_name}
lambda_name=${bucket_name}

echo '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "firehose.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}' |tee role-trust-policy.json
aws iam create-role --role-name ${role_name} \
  --assume-role-policy-document file://role-trust-policy.json

envsubst >role-policy.json <<-EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "glue:GetTable",
                "glue:GetTableVersion",
                "glue:GetTableVersions"
            ],
            "Resource": [
                "arn:aws:glue:${aws_region}:${aws_account_id}:catalog",
                "arn:aws:glue:${aws_region}:${aws_account_id}:database/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%",
                "arn:aws:glue:${aws_region}:${aws_account_id}:table/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:ListBucketMultipartUploads",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::${bucket_name}",
                "arn:aws:s3:::${bucket_name}/*"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction",
                "lambda:GetFunctionConfiguration"
            ],
            "Resource": "arn:aws:lambda:${aws_region}:${aws_account_id}:function:%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:GenerateDataKey",
                "kms:Decrypt"
            ],
            "Resource": [
                "arn:aws:kms:${aws_region}:${aws_account_id}:key/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
            ],
            "Condition": {
                "StringEquals": {
                    "kms:ViaService": "s3.${aws_region}.amazonaws.com"
                },
                "StringLike": {
                    "kms:EncryptionContext:aws:s3:arn": [
                        "arn:aws:s3:::%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%/*",
                        "arn:aws:s3:::%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
                    ]
                }
            }
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:${aws_region}:${aws_account_id}:log-group:/aws/kinesisfirehose/${firehose_name}:log-stream:*",
                "arn:aws:logs:${aws_region}:${aws_account_id}:log-group:%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%:log-stream:*"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "kinesis:DescribeStream",
                "kinesis:GetShardIterator",
                "kinesis:GetRecords",
                "kinesis:ListShards"
            ],
            "Resource": "arn:aws:kinesis:${aws_region}:${aws_account_id}:stream/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt"
            ],
            "Resource": [
                "arn:aws:kms:${aws_region}:${aws_account_id}:key/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
            ],
            "Condition": {
                "StringEquals": {
                    "kms:ViaService": "kinesis.${aws_region}.amazonaws.com"
                },
                "StringLike": {
                    "kms:EncryptionContext:aws:kinesis:arn": "arn:aws:kinesis:${aws_region}:${aws_account_id}:stream/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%"
                }
            }
        }
    ]
}
EOF

sed -i '/:function:/s/%FIREHOSE_POLICY_TEMPLATE_PLACEHOLDER%/'"${lambda_name}"':$LATEST/' role-policy.json

policy_arn=$(aws iam create-policy \
--policy-name ${role_name} \
--policy-document file://role-policy.json |jq -r '.Policy.Arn')

aws iam attach-role-policy --role-name ${role_name} \
  --policy-arn ${policy_arn}
aws iam list-attached-role-policies --role-name ${role_name}

role_arn=$(aws iam get-role --role-name ${role_name} |jq -r '.Role.Arn')

sleep 10
aws firehose create-delivery-stream \
--delivery-stream-name ${firehose_name} \
--delivery-stream-type "DirectPut" \
--extended-s3-destination-configuration "RoleARN=${role_arn},BucketARN=arn:aws:s3:::${bucket_name},Prefix=${s3_prefix}/,ErrorOutputPrefix=${s3_prefix}_failed/,BufferingHints={SizeInMBs=2,IntervalInSeconds=120},CompressionFormat=GZIP,EncryptionConfiguration={NoEncryptionConfig=NoEncryption},CloudWatchLoggingOptions={Enabled=true,LogGroupName=${role_name},LogStreamName=${role_name}},ProcessingConfiguration={Enabled=true,Processors=[{Type=Lambda,Parameters=[{ParameterName=LambdaArn,ParameterValue=${lambda_arn}:\$LATEST},{ParameterName=BufferSizeInMBs,ParameterValue=1},{ParameterName=BufferIntervalInSeconds,ParameterValue=60}]}]}"

# no data transform
# --s3-destination-configuration "RoleARN=${role_arn},BucketARN=arn:aws:s3:::${bucket_name},Prefix=CentralizedAccountLogs/,ErrorOutputPrefix=CentralizedAccountLogs_failed/,BufferingHints={SizeInMBs=1,IntervalInSeconds=60},CompressionFormat=UNCOMPRESSED,EncryptionConfiguration={NoEncryptionConfig=NoEncryption},CloudWatchLoggingOptions={Enabled=true,LogGroupName=${role_name},LogStreamName=${role_name}}"

firehose_arn=$(aws firehose describe-delivery-stream --delivery-stream-name ${firehose_name} --query "DeliveryStreamDescription.DeliveryStreamARN" --output text)

while true ; do
  firehose_status=$(aws firehose describe-delivery-stream --delivery-stream-name ${firehose_name} --query "DeliveryStreamDescription.DeliveryStreamStatus" --output text)
  echo ${firehose_status}
  if [[ ${firehose_status} == "ACTIVE" ]]; then
    break
  fi
  sleep 10
done

```


### cloudwatch
- 创建cwl所需角色来访问firehose

```sh
cwl_role_name=cwl-firehose-role-$RANDOM
aws_region=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
aws_account_id=$(aws sts get-caller-identity --query "Account" --output text)

envsubst > cwl-role-trust-policy.json <<-EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "logs.${aws_region}.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
aws iam create-role --role-name ${cwl_role_name} \
  --assume-role-policy-document file://cwl-role-trust-policy.json

envsubst > cwl-role-policy.json <<-EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "firehose:PutRecord",
                "firehose:PutRecordBatch"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:firehose:${aws_region}:${aws_account_id}:deliverystream/*"
        }
    ]
}
EOF
cwl_policy_arn=$(aws iam create-policy \
--policy-name ${cwl_role_name} \
--policy-document file://cwl-role-policy.json |jq -r '.Policy.Arn')
aws iam attach-role-policy --role-name ${cwl_role_name} \
  --policy-arn ${cwl_policy_arn}
aws iam list-attached-role-policies --role-name ${cwl_role_name}

cwl_role_arn=$(aws iam get-role --role-name ${cwl_role_name} |jq -r '.Role.Arn')

```

- 注册firehose到eks集群到log group

```sh
log_group_name=/aws/eks/ekscluster1/cluster

aws logs create-log-group \
--log-group-name ${log_group_name}

aws logs put-subscription-filter \
--log-group-name ${log_group_name} \
--filter-name "other" \
--filter-pattern "" \
--destination-arn ${firehose_arn} \
--role-arn ${cwl_role_arn}

```

### glue
- create database in glue catalog `testdb`
- create crawler

#### using cli
```sh
database_name=testdb
s3_uri="${bucket_name}/${s3_prefix}/"

crawler_role_name=AWSGlueServiceRole-$RANDOM
aws iam create-role --role-name ${crawler_role_name} --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"glue.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ${crawler_role_name} --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

envsubst > crawler-role-policy.json <<-EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::${s3_uri}*"
            ]
        }
    ]
}
EOF
crawler_policy_arn=$(aws iam create-policy \
--policy-name ${crawler_role_name} \
--policy-document file://crawler-role-policy.json |jq -r '.Policy.Arn')
aws iam attach-role-policy --role-name ${crawler_role_name} \
  --policy-arn ${crawler_policy_arn}
crawler_role_arn=$(aws iam get-role --role-name ${crawler_role_name} |jq -r '.Role.Arn')

aws glue create-database \
    --database-input '{"Name":"'"${database_name}"'"}'

sleep 10
crawler_name=c1-$RANDOM
aws glue create-crawler --name ${crawler_name} \
--role ${crawler_role_arn} \
--database-name ${database_name} \
--targets '{
  "S3Targets": [
    {
      "Path": "s3://'"${s3_uri}"'"
    }
  ]
}'

# run crawler later
# aws glue start-crawler --name ${crawler_name}

```

#### using ui
![[Pasted image 20221002145225.png]]

![[Pasted image 20221002145319.png]]

![[Pasted image 20221002145338.png]]

![[Pasted image 20221002145404.png]]


### databrew
#### using cli
```sh
databrew_name=cwl-$RANDOM
databrew_output=parquet-$RANDOM

# "Json={MultiLine=false}": json line, not json document
aws databrew create-dataset \
--name ${databrew_name} \
--format JSON \
--format-options "Json={MultiLine=false}" \
--input "S3InputDefinition={Bucket=${bucket_name},Key=${s3_prefix}/}"

wget -O cwl.json 'https://github.com/panlm/aws-labs/raw/main/eks-cloudwatch-log-firehose-s3/cwl-recipe.json'
aws databrew create-recipe \
--name ${databrew_name} \
--steps file://cwl.json

aws databrew publish-recipe \
--name ${databrew_name}

databrew_role_name=AWSGlueDataBrewServiceRole-$RANDOM
aws iam create-role --role-name ${databrew_role_name} --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"databrew.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ${databrew_role_name} --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueDataBrewServiceRole
aws iam attach-role-policy --role-name ${databrew_role_name} --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
databrew_role_arn=$(aws iam get-role --role-name ${databrew_role_name} |jq -r '.Role.Arn')

sleep 10
# this command will failed when bucket is empty
# aws s3 ls --recursive s3://${bucket_name}/${s3_prefix}/
aws databrew create-recipe-job \
--dataset-name ${databrew_name} \
--name ${databrew_name} \
--recipe-reference 'Name='"${databrew_name}"',RecipeVersion=1.0' \
--outputs "Format=PARQUET,Location={Bucket=${bucket_name},Key=${databrew_output}/},Overwrite=true" \
--role-arn ${databrew_role_arn}

aws databrew create-schedule \
--cron-expression 'Cron(*/15 * * * ? *)' \
--name per15min-${databrew_name} \
--job-names ${databrew_name}

```

create another crawler and query with athena
```sh
s3_uri_2="${bucket_name}/${databrew_output}/"

crawler_role_name_2=AWSGlueServiceRole-$RANDOM
aws iam create-role --role-name ${crawler_role_name_2} --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"glue.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ${crawler_role_name_2} --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

envsubst > crawler-role-policy.json <<-EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::${s3_uri_2}*"
            ]
        }
    ]
}
EOF
crawler_policy_arn_2=$(aws iam create-policy \
--policy-name ${crawler_role_name_2} \
--policy-document file://crawler-role-policy.json |jq -r '.Policy.Arn')
aws iam attach-role-policy --role-name ${crawler_role_name_2} \
  --policy-arn ${crawler_policy_arn_2}
crawler_role_arn_2=$(aws iam get-role --role-name ${crawler_role_name_2} |jq -r '.Role.Arn')

sleep 10
crawler_name_2=parquet-${crawler_name}
aws glue create-crawler --name ${crawler_name_2} \
--role ${crawler_role_arn_2} \
--database-name ${database_name} \
--targets '{
  "S3Targets": [
    {
      "Path": "s3://'"${s3_uri_2}"'"
    }
  ]
}'

# run crawler later
# awtart-crawler --name ${crawler_name_2}

```

check databrew job status
```sh
aws databrew start-job-run --name ${databrew_name}
aws databrew list-job-runs --name ${databrew_name}

```

if job status is `SUCCEEDED`, start crawler to create catalog in glue
```sh
aws glue start-crawler --name ${crawler_name}
aws glue start-crawler --name ${crawler_name_2}

```

#### using ui
- import recipe
    - [download](https://github.com/panlm/aws-labs/raw/main/eks-cloudwatch-log-firehose-s3/cwl-recipe.json)
![[Pasted image 20221006104556.png]]

- create dataset from s3 or glue catalog
![[Pasted image 20221006104147.png]]

- create project from this dataset and using existed recipe
![[Pasted image 20221006105033.png]]

![[Pasted image 20221006105336.png]]

- create job as you need

### athena
this table is created by firehose
![[Pasted image 20221007205651.png]]

this table is created by databrew job
![[Pasted image 20221007205729.png]]

## conclusion
满足客户需求，基于目前s3中保存的原始数据，并且可以进行字段拆分等二次处理，未来可以使用aws databrew进行更复杂的处理

## alternative 
- [export-cloudwatch-log-group-to-s3](./export-cloudwatch-log-group-to-s3)

## reference
- https://aws.amazon.com/blogs/architecture/stream-amazon-cloudwatch-logs-to-a-centralized-account-for-audit-and-analysis/
    - https://github.com/aws-samples/amazon-cloudwatch-log-centralizer
- https://aws.amazon.com/premiumsupport/knowledge-center/kinesis-firehose-cloudwatch-logs/
- https://www.chaossearch.io/blog/cloudwatch2s3-an-easy-way-to-get-your-logs-to-aws-s3
- [[eks-control-panel-log-cwl-firehose-opensearch]]
- [[cloudwatch-firehose-splunk]]
- [[eks-loggroup-description]]


