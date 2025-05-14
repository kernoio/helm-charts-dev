# Kerno Agent Helm Chart (dev)

## Add the repository to your local helm:

```bash

helm repo add kerno-dev https://kernoio.github.io/helm-charts-dev

```

## Install the Chart

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  --set api-key="<KERNO_API_KEY>"
```

## Pluggable storage

### AWS S3

#### Creating the bucket and necessary roles to access it from an EKS cluster.

To create and maintain an object storage for logs and stack traces, the following needs to be done:
- an S3 bucket is created
- a role with a that can be assumed from the cluster, with a trust relationship from the cluster oidc to the aws account
- a role policy allowing the role access to the S3 bucket 
- a bucket policy allowing the role to get, create and delete from the bucket
 
Here is a more detailed set of instructions:

1. Create an S3 bucket and make a note of the name
2. With your cluster's oidc url at hand, 
   - create a role with `Web identity` Trusted Entity type 
and choose your oidc provider url when choosing an `Identiy Provider`.
   - For the audience, choose `sts.amazonaws.com`. 
   - Add a condition with:
     - _Key_ `<oidc-url>:sub`, this option should be available in the dropdown. 
     - _Condition_ `StringLike` and
     - _Value_ `system:serviceaccount:kerno:*`
   - On the next page, do not choose any policy, a custom one for access to the Kerno bucket will be created in the next step. Click _Next_.
   - Fill in a role name, and description if desired. 
    
    The `Turst policy` json should look like the following:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider-url-without-https://>"
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

    This allows the kerno service account to assume a role with web identity in the AWS account.


3. Navigate to the created role, and click _Add permissions_ and _Create inline policy_. Use _JSON_ and paste the following, replacing `bucket-name` with the bucket name created in step 1.:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:*"
                ],
                "Resource": [
                    "arn:aws:s3:::<bucket-name>",
                    "arn:aws:s3:::<bucket-name>/*"
                ]
            }
        ]
    }
    ```
   Name the policy and create it.
 

4. Finally, navigate to the bucket created for Kerno data. Click on _Permissions_ and   _Edit_ the _Bucket Policy_ with the following JSON, where `role-arn` is the role created and `bucket-name` is the name of the bucket:
    ```
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

#### Filling in `values.yaml`

Using the `bucket-name` and `role-arn` from steps 1 and 2, fill in `values.yaml` with the following:
- `cloud: AWS`
- `bucketName: <bucket-name>`
- ```
  serviceAccountAnnotations: 
    eks.amazon.com/role-arn: <role-arn>
  ```
  
An example can be found in `examples/aws-values.yaml`

#### Installing

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --namespace kerno \
  -f path/to/values.yaml
```
