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

An example can be found in `examples/aws-values.yaml`.

### 6. Install the Chart Using Your Config

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  -f path/to/values.yaml
```
---
## Using GCP Cloud Storage for Persistent Logs

To enable persistent storage of logs and stack traces in Google Cloud Storage, follow the steps below to provision the necessary infrastructure and configure your GKE cluster accordingly.

### Prerequisites

- Ensure that **Workload Identity Federation** is enabled on the cluster.
- Enable the **GKE Metadata Server** on at least one node pool.

### Step 1: Create a Cloud Storage Bucket

Create a bucket in the same GCP project as the GKE cluster. This bucket will store Kerno logs and stack traces. Record the bucket name for later use.

### Step 2: Create a Google Service Account

In the GCP project, create a Service Account. Note the email address of the newly created account.

### Step 3: Grant the Service Account Access to the Bucket

1. Navigate to the Cloud Storage bucket in the console.
2. Click **"Grant Access"**.
3. For **Principal**, enter the Service Account's email.
4. For **Role**, select `Storage Admin`.
5. Save the changes.

### Step 4: Allow Workload Identity Access to the Service Account

1. Navigate to the Service Account in the IAM console.
2. Click **"Grant Access"**.
3. Add the following principal: `<project-id>.svc.id.goog[kerno/kerno-sa]`
4. Select the `Service Account Admin` role.
5. Save the configuration.

### Step 5: Configure Helm Chart Values

Update your `values.yaml` with the following content:

```yaml
cloud: GCP
bucketName: <bucket-name>
serviceAccountAnnotations:
  iam.gke.io/gcp-service-account: <service-account-email>
```

An example can be found in `examples/gcp-values.yaml`.

### 6. Install the Chart Using Your Config

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  -f path/to/values.yaml
```
---
