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

### AWS

To create and maintain an object storage for logs and stack traces, the following needs doing.
- an S3 bucket is created
- a role with a that can be assumed from the cluster, with a trust relationship from cluster oidc to the aws account. 
- a role policy allowing the role access to the S3 bucket 
- a bucket policy allowing the bucket to get, create and delete from the bucket
 
Here is a more detailed set of instructions

1. Create the bucket and obtain the name
2. Create role with the following trust relationship, replacing `account-id` and `oidc-provider-url` (Note removing the prefix for the `Federated` key):
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
3. Create the role policy with the id of the role from the previous step and `bucket-name` is the name of the bucket created in step 1.:
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
4. Finally, create a bucket policy allowing all actions on the bucket, replacing `role-arn` with the arn of the role :

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
