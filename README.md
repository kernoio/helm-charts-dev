# Kerno Agent Helm Chart

## Add the Helm Repository

```bash

helm repo add kerno-dev https://kernoio.github.io/helm-charts-dev
```

## Installing the Chart

Replace `<KERNO_API_KEY>` with your actual API key:

```bash

helm upgrade --install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  --set apiKey="<KERNO_API_KEY>" \
  --atomic
```

## Persistent Storage

Kerno allows storing logs and stack traces persistently within AWS accounts 
and GCP projects with S3 and Cloud Storage respectively. 
Below are steps needed to create and configure access to an object storage resource.

### AWS S3 for Persistent Storage

#### 1. Create an S3 Bucket

Create a bucket in S3 and note the name. This bucket will be used to store Kerno logs and stack traces.

#### 2. Set Up IAM Role for EKS Access

Create an IAM role that your EKS cluster can assume via its OIDC provider:

- **Trusted Entity Type**: Web Identity
- **Identity Provider**: Use your cluster's OIDC URL
- **Audience**: `sts.amazonaws.com`
- **Condition**:
    - **Key**: `<oidc-url>:sub`
    - **Operator**: `StringLike`
    - **Value**: `system:serviceaccount:kerno:*`

##### Trust Policy Example

Replace placeholders with your actual values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider-url-https-prefix-removed>"
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

#### 3. Attach Permissions to the IAM Role

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

#### 4. Update the Bucket Policy

Allow the IAM role access to the bucket using the role arn from step 2:

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

#### 5. Update `values.yaml`

Set the following values in your `values.yaml`:

```yaml
cloud: AWS
bucketName: <bucket-name>
serviceAccountAnnotations:
  eks.amazonaws.com/role-arn: <role-arn>
```
replacing `<bucket-name>` and `<role-arn>`.

An example can be found in `examples/aws-values.yaml`.

#### 6. Install the Chart Using Your Config

```bash

helm upgrade --install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  -f path/to/values.yaml \
  --atomic
```
---
### GCP Cloud Storage for Persistent Storage

#### Prerequisites

- Ensure that **Workload Identity Federation** is enabled on the cluster.
- Enable the **GKE Metadata Server** on at least one node pool.


#### Step 1: Create a Google Service Account

In the GCP project, create a Service Account, noting its email address.

#### Step 2: Allow Workload Identity Access to the Service Account

1. Navigate to the Service Account in the IAM console.
2. Click **"Grant Access"**.
3. Add the following principal: `<project-id>.svc.id.goog[kerno/kerno-sa]`, 
 replacing `<project-id>` with the GCP project id.
4. Select the `Service Account Admin` role.
5. Save the configuration.

#### Step 3: Create a Cloud Storage Bucket

Create a bucket in the same GCP project as the GKE cluster.

#### Step 4: Grant the Service Account Access to the Bucket

1. Navigate to the Cloud Storage bucket.
2. Click **"Grant Access"**.
3. For **Principal**, enter the Service Account's email.
4. For **Role**, select `Storage Admin`.
5. Save the changes.

#### Step 5: Configure Helm Chart Values

Update your `values.yaml` with the following content:

```yaml
cloud: GCP
bucketName: <bucket-name>
serviceAccountAnnotations:
  iam.gke.io/gcp-service-account: <service-account-email>
```
replacing `<bucket-name>` and `<service-account-email>`.

An example can be found in `examples/gcp-values.yaml`.

### 6. Install the Chart Using Your Config

```bash

helm upgrade --install kerno-agent kerno-dev/agent \
  -f path/to/values.yaml \
  --create-namespace \
  --namespace kerno \
  --atomic 
```
---

> **_NOTE_** 
> 
> The `values.yaml` should be committed to version control so it can
> be used when upgrading the Kerno helm chart. The API Key should 
> also be kept secret.
> To pass the API Key in CI, use an empty string in `values.yaml` and
> override it using `--set apiKey=${{ secrets.KERNO_API_KEY }}`.
> For example, if using Github Actions the command becomes:

```bash

 helm upgrade --install kerno-agent kerno-dev/agent \
  -f path/to/values.yaml \
  --create-namespace \
  --namespace kerno \
  --set apiKey=${{ secrets.KERNO_API_KEY}}
  --atomic 
```

> It can also be used with an IaC framework that can manage Helm charts such as Terraform or Pulumi.
> Simply follow the docs to deploy a Helm Chart for the provider using the appropriate values
> where needed.
