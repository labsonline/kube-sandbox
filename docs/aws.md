# AWS

Create AWS credential using one of the following methods:

- [IRSA](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-roles-for-service-accounts)
- [Node IAM Role](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#node-iam-role)
- [Static Credentials (___LESS SECURE___)](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#static-credentials)

[Install AWS cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)\
[Install `clusterawsadm`](https://cluster-api-aws.sigs.k8s.io)

And prepare for deployment:

```shell
# Export credentials
export AWS_REGION="ca-central-1"
export AWS_ACCESS_KEY_ID="XXXXXX"
export AWS_SECRET_ACCESS_KEY="XXXXXX"
export AWS_SESSION_TOKEN=""             # Optional. If you are using Multi-Factor or Single Sign On Auth.

# Check for available IPs
# and update if less than 3
LIMIT=$(aws ec2 describe-account-attributes --attribute-names vpc-max-elastic-ips --query 'AccountAttributes[0].AttributeValues[0].AttributeValue' --output text)
USED=$(aws ec2 describe-addresses --query 'Addresses[*].PublicIp' --output text | wc -w)
AVAILABLE=$((LIMIT - USED))
echo "Available Public IPs: $AVAILABLE"

aws service-quotas request-service-quota-increase \
    --service-code ec2 \
    --quota-code L-0263D0A3 \
    --desired-value 20
aws service-quotas list-requested-service-quota-change-history \
    --service-code ec2
```

## KCM

```shell
# Create AWS k0rdent user
aws iam create-user --user-name $AWS_USERNAME

# Configure AWS access for k0rdent
clusterawsadm bootstrap iam create-cloudformation-stack

# Attach IAM policies to k0rdent user
AWS_ARN_ID=$(aws iam get-user --user-name $AWS_USERNAME --query 'User.Arn' --output text | sed -E 's/.*::([0-9]+):.*/\1/')
echo $AWS_ARN_ID

aws iam attach-user-policy --user-name $AWS_USERNAME --policy-arn arn:aws:iam::$AWS_ARN_ID:policy/controllers.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name $AWS_USERNAME --policy-arn arn:aws:iam::$AWS_ARN_ID:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name $AWS_USERNAME --policy-arn arn:aws:iam::$AWS_ARN_ID:policy/nodes.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name $AWS_USERNAME --policy-arn arn:aws:iam::$AWS_ARN_ID:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io

aws iam list-attached-user-policies --user-name $AWS_USERNAME

# Create AWS Credential for k0rdent user
aws iam create-access-key --user-name $AWS_USERNAME
```

See [README](../README.md#3-install-kcm) for to continue with deployment.

## KOF
