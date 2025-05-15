# Kerno Agent Helm Chart

## Add the Helm Repository

```bash

helm repo add kerno-dev https://kernoio.github.io/helm-charts-dev
```

## Installing the Chart

Replace `<KERNO_API_KEY>` with your actual API key:

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  --set apiKey="<KERNO_API_KEY>"
```

## Using AWS S3 for Storage

To enable persistent storage of logs and stack traces in AWS S3, follow the steps below to set up the required infrastructure:

### 1. Create an S3 Bucket

Create a bucket in S3 and note the name. This bucket will be used to store Kerno logs and stack traces.

### 2. Set Up IAM Role for EKS Access

Create an IAM role that your EKS cluster can assume via its OIDC provider:

- **Trusted Entity Type**: Web Identity
- **Identity Provider**: Use your cluster's OIDC URL
- **Audience**: `sts.amazonaws.com`
- **Condition**:
    - **Key**: `<oidc-url>:sub`
    - **Operator**: `StringLike`
    - **Value**: `system:serviceaccount:kerno:*`

#### Trust Policy Example

Replace placeholders with your actual values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider-url>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "<oidc-provider-url>:sub": "system:serviceaccount:kerno:*"
        },
        "StringEquals": {
          "<oidc-provider-url>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### 3. Attach Permissions to the IAM Role

Attach an inline policy granting access to the bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::<bucket-name>",
        "arn:aws:s3:::<bucket-name>/*"
      ]
    }
  ]
}
```

### 4. Update the Bucket Policy

Allow the IAM role access to the bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<role-arn>"
      },
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::<bucket-name>",
        "arn:aws:s3:::<bucket-name>/*"
      ]
    }
  ]
}
```

### 5. Update `values.yaml`

Set the following values in your `values.yaml`:

```yaml
cloud: AWS
bucketName: <bucket-name>
serviceAccountAnnotations:
  eks.amazonaws.com/role-arn: <role-arn>
```

You can find an example in `examples/aws-values.yaml`.

### 6. Install the Chart Using Your Config

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  -f path/to/values.yaml
```
---
