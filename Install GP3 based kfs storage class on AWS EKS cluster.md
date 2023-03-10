# Install GP3 based kfs storage class on AWS EKS cluster.

## Steps:

1. [Create an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html). If it hasn't already been created. This will enable secure and automated authentication and authorization between your cluster and AWS resources using IAM roles.

> Note: Give any unique id after any policy or role name to understand which policy is attached to which role and cluster 

2. Create an IAM policy with a unique name, such as (Katonic-EKS_EBS_CSI_Driver_Policy-<unique_id >) to give permission to create a GP3-based volume. This policy should define the necessary permissions for the Amazon EBS CSI driver to manage EBS volumes on behalf of the Kubernetes cluster. 

```
example-iam-policy-for-gp3.json
```

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": [
            "CreateVolume",
            "CreateSnapshot"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/kubernetes.io/created-for/pvc/name": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeSnapshotName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    }
  ]
}

```


aws iam create-policy --policy-name Katonic-EKS_EBS_CSI_Driver_Policy-{{ unique_id.stdout }} --policy-document file://example-iam-policy-for-gp3.json
Note: Give unique id of to policy.

 

3. To retrieve your AWS account number, use the following command:

```
aws sts get-caller-identity --query "Account" --output text
```

4. To retrieve your IAM OIDC number, use the following command:

```
aws eks describe-cluster --name {{ cluster_name }} --region {{ aws_region }} --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5
```

5. Create a policy file katonic-trust-policy.json, which will define the trust relationship between your Kubernetes cluster and AWS. This policy should reference your AWS account number and OIDC number.

```
katonic-trust-policy.json
```

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/oidc_id_number"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/oidc_id_number:aud": "sts.amazonaws.com",
          "oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/oidc_id_number:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}

```
> Note: Update YOUR_AWS_REGION, YOUR_AWS_ACCOUNT_ID , oidc_id_number in this policy file.

 

6. Create an IAM role with a unique name, such as (Katonic_EKS_EBS_CSI_DriverRole_<unique_id>)by using the above policy.

```
aws iam create-role --role-name Katonic_EKS_EBS_CSI_DriverRole_<unique_id> --assume-role-policy-document file://"katonic-trust-policy.json"
```
> Note: Give unique id of to role.

 

7. Attach Katonic-EKS_EBS_CSI_Driver_Policy-<unique_id>  policy to Katonic_EKS_EBS_CSI_DriverRole_<unique_id> role.

```
aws iam attach-role-policy --policy-arn arn:aws:iam::<iam_number>:policy/Katonic-EKS_EBS_CSI_Driver_Policy-<unique_id> --role-name Katonic_EKS_EBS_CSI_DriverRole_<unique_id>
```

> Note: Update AWS IAM account number.

 

8. Deploy the Amazon EBS CSI driver to your cluster. 

> Note: This driver is installed in all regions except for the China region.

```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.14"
```

9. Annotate the ebs-csi-controller-sa Kubernetes service account with the Amazon Resource Name (ARN) of the IAM role that you created earlier. This will allow the service account to assume the IAM role and access the necessary AWS resources.

```
kubectl annotate serviceaccount ebs-csi-controller-sa -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::<iam_number>:role/Katonic_EKS_EBS_CSI_DriverRole_<unique_id>
```
> Note: Update AWS IAM account number and unique id

 

10. Restart the ebs-csi-controller deployment for the annotation to take effect.

```
kubectl rollout restart deployment ebs-csi-controller -n kube-system
```


11. Create a GP3 based storage class by defining a Kubernetes storage class with the appropriate parameters, such as storage size, volume type, and availability zone.

```
aws-kfs-sc.yml
```

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kfs 
parameters:
  type: gp3
  fsType: ext4 
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```
kubectl apply -f aws-kfs-sc.yml
```
