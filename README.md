# Test Secrets Service Class

## Overview
This is an example of how to grant access secrets from AWS Secrets Manager to EKS pods.

## Development Environment

To facilitate the development we have used [Devbox](https://www.jetify.com/devbox/docs/).

#### Configure Devbox

To install Devbox:

```sh
curl -fsSL https://get.jetify.com/devbox | bash
```

Start virtual shell:

```sh
devbox shell
```

# Step 1: Set up access control

## Create the Secret

We need to create a secret initially:

```sh
aws secretsmanager create-secret \
    --name TestSecret \
    --description "My test secret created with the CLI." \
    --secret-string "{\"user\":\"sergioi\",\"password\":\"XXXyyyZZZ000\"}"
``` 
Note the ARN address of the secret. In our case is `arn:aws:secretsmanager:us-east-1:142728997126:secret:TestSecret-ZQxKpG` .

## 1. Create an IAM OIDC provider for the cluster

```sh
cluster_name=basic-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Determine whether an IAM OIDC provider with your cluster's issuer ID is already in your account.

```sh
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If output is returned, then you already have an IAM OIDC provider for your cluster and you can skip the next step. If no output is returned, then you must create an IAM OIDC provider for your cluster.

Create an IAM OIDC identity provider for your cluster with the following command.

```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## 2. Assign IAM roles to Kubernetes service accounts

Create the file of the policy to grant access the pods to Secrets Manager:

```sh
cat >asm-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BasePermissions",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:142728997126:secret:TestSecret-ZQxKpG"
        }
    ]
}
EOF
```

Create the policy:

```sh
aws iam create-policy --policy-name my-asm-policy --policy-document file://asm-policy.json
```
Take note of the policy ARN, and export it to the environment.

```sh
export policy_arn=arn:aws:iam::111122223333:policy/my-asm-policy
```

Create an IAM role and associate it with a Kubernetes service account. You can use either eksctl or the AWS CLI.

```sh
eksctl create iamserviceaccount --name my-asm-service-account --namespace default --cluster $cluster_name --role-name my-asm-role --attach-policy-arn $policy_arn --approve
```

Confirm that the IAM role's trust policy is configured correctly.

```sh
aws iam get-role --role-name my-asm-role --query Role.AssumeRolePolicyDocument
```

Confirm that the policy that you attached to your role in a previous step is attached to the role.

```sh
aws iam list-attached-role-policies --role-name my-asm-role --query AttachedPolicies[].PolicyArn --output text
```
Note: I couldn't make this work id shows nothing.


View the default version of the policy.


```sh
aws iam get-policy --policy-arn $policy_arn
```

View the policy contents to make sure that the policy includes all the permissions that your Pod needs. If necessary, replace 1 in the following command with the version that's returned in the previous output.

```sh
aws iam get-policy-version --policy-arn $policy_arn --version-id v1

```

Confirm that the Kubernetes service account is annotated with the role.

```sh
kubectl describe serviceaccount my-asm-service-account -n default
```

# Step 2: Install and configure the ASCP

## To install the ASCP by using Helm

1. To ensure the repo is pointing to the latest charts, use `helm repo update`.
2. Add the Secrets Store CSI Driver chart.
```sh
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
```
3. Install the chart [2].
```sh
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true
```
4. Add the ASCP chart.
```sh
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
```
5. Install the chart. To use a FIPS endpoint, add the following flag: `--set useFipsEndpoint=true`
```sh
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```

# Step 3: Identify which secrets to mount

In this guide we are limiting the access to the only secret we created but you can modify you policy to access to all secrets, a group of them, etc.

To know more about how to define the mounts of the secrets en in provider class definition please visit the link [3] in the references section.

# Step 4: Mount the secrets as files in the Amazon EKS pod

To mount secrets in Amazon EKS
- Apply the SecretProviderClass to the pod with the command `kubectl apply -f SecretProviderClass.yaml`.
- Deploy your pod with the command `kubectl apply -f my-deployment.yaml`.
- The ASCP mounts the files.

Secrets should exists:

```sh
kubectl -n default get secrets
```

Verify the volume was mounted:

```sh
cd /mnt/secrets-store
ls -hatlr
cat credentials
```

Verify the environment variables referencing the secrets exists:

```sh
env | grep ^DB*
```

## References
1. https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html
2. https://stackoverflow.com/questions/73192808/env-variable-from-aws-secrets-manager-in-kubernetes
3. https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html#integrating_csi_driver_mount

## Credits

- Created by Sergio Rodríguez Inclán <srinclan@arcamo.org>
- AWS User Group Cochabamba: https://www.linkedin.com/groups/12914090/
