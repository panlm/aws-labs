---
created: 2022-08-04 13:24:34.806
last_modified: 2022-08-04 13:24:34.806
tags: kubernetes aws/network/route53 
---
```ad-attention
title: This is a github note

```
# externaldns-for-route53
[link](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)

## install
```sh
EKS_CLUSTER_NAME=ekscluster1
EXTERNALDNS_NS=default
AWS_REGION=us-east-2
DOMAIN_NAME=external.panlm.xyz

cat >policy.json <<-EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

POLICY_NAME=AllowExternalDNSUpdates-${RANDOM}
aws iam create-policy --policy-name ${POLICY_NAME} --policy-document file://policy.json

# example: arn:aws:iam::XXXXXXXXXXXX:policy/AllowExternalDNSUpdates
export POLICY_ARN=$(aws iam list-policies \
 --query 'Policies[?PolicyName==`'"${POLICY_NAME}"'`].Arn' --output text)

eksctl create iamserviceaccount \
  --cluster $EKS_CLUSTER_NAME \
  --name "external-dns" \
  --namespace ${EXTERNALDNS_NS:-"default"} \
  --override-existing-serviceaccounts \
  --attach-policy-arn $POLICY_ARN \
  --approve

```

```sh
echo ${DOMAIN_NAME}
echo ${AWS_REGION}

envsubst >externaldns-with-rbac.yaml <<-EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods","nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  labels:
    app.kubernetes.io/name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: default # change to desired namespace: externaldns, kube-addons
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  labels:
    app.kubernetes.io/name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: external-dns
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: k8s.gcr.io/external-dns/external-dns:v0.12.2
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=${DOMAIN_NAME} # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws
            - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
#            - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
            - --registry=txt
            - --txt-owner-id=external-dns
          env:
            - name: AWS_DEFAULT_REGION
              value: ${AWS_REGION} # change to region where EKS is installed
     # # Uncommend below if using static credentials
     #        - name: AWS_SHARED_CREDENTIALS_FILE
     #          value: /.aws/credentials
     #      volumeMounts:
     #        - name: aws-credentials
     #          mountPath: /.aws
     #          readOnly: true
     #  volumes:
     #    - name: aws-credentials
     #      secret:
     #        secretName: external-dns
EOF

kubectl create --filename externaldns-with-rbac.yaml \
  --namespace ${EXTERNALDNS_NS:-"default"}

```

## setup hosted zone
[[route53-subdomian]]

```sh
aws route53 create-hosted-zone --name "${DOMAIN_NAME}." \
  --caller-reference "external-dns-test-$(date +%s)"

ZONE_ID=$(aws route53 list-hosted-zones-by-name --output json \
  --dns-name "${DOMAIN_NAME}." --query HostedZones[0].Id --out text)

aws route53 list-resource-record-sets --output text \
  --hosted-zone-id $ZONE_ID --query \
  "ResourceRecordSets[?Type == 'NS'].ResourceRecords[*].Value | []" | tr '\t' '\n'

# using output as value to add NS record on your domain registrar

```

### private hosted zone
you also could create private hosted zone and associate to your vpc. plugin will insert/update record in your private hosted zone. ([link](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html))

## verify - service sample
```sh
envsubst >nginx.yaml <<-EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.${DOMAIN_NAME}
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
EOF

kubectl create --filename nginx.yaml

```

```sh
aws route53 list-resource-record-sets --output json --hosted-zone-id $ZONE_ID \
  --query "ResourceRecordSets[?Name == 'nginx.${DOMAIN_NAME}.']|[?Type == 'A']"

aws route53 list-resource-record-sets --output json --hosted-zone-id $ZONE_ID \
  --query "ResourceRecordSets[?Name == 'nginx.${DOMAIN_NAME}.']|[?Type == 'TXT']"

dig +short nginx.${DOMAIN_NAME}.

curl nginx.${DOMAIN_NAME}.

```

## verify - ingress sample
```sh
envsubst >nginx-ingress.yaml <<-EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/tags: Environment=dev,Team=test,Application=nginx
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: server.${DOMAIN_NAME}
      http:
        paths:
          - backend:
              service:
                name: nginx
                port:
                  number: 80
            path: /
            pathType: Prefix
EOF

kubectl create --filename nginx-ingress.yaml

```

```sh
aws route53 list-resource-record-sets --output json --hosted-zone-id $ZONE_ID \
  --query "ResourceRecordSets[?Name == 'server.${DOMAIN_NAME}.']"

dig +short server.${DOMAIN_NAME}.

curl server.${DOMAIN_NAME}.

```





