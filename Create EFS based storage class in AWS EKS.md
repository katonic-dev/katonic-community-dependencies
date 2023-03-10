# Create EFS based storage class in AWS EKS

## Step 1: [Create an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html). If already created then skip this step.

This step is necessary to enable the Kubernetes service accounts to authenticate against IAM and use AWS resources. You can create an IAM OIDC provider for your cluster using the AWS Management Console, AWS CLI, or AWS SDKs.

## Step 2: Create AWS EFS with a standard storage class to enable the encryption of data. 

You need to create an EFS file system that can be used as persistent storage for your Kubernetes pods. You can choose the standard storage class to enable the encryption of data at rest. Encryption at rest uses AWS Key Management Service (KMS) to encrypt your data.
![image](https://user-images.githubusercontent.com/99641515/224278387-1cef0975-4e88-411f-b9a6-50bf426c15d5.png)


## Step 3: In the network access select the VPC where you deploy the EKS cluster. After that select the Availability zone and subnet IDs of your created VPC. 

When creating the EFS file system, you need to specify the VPC where you deploy the EKS cluster, and then select the Availability Zone and Subnet IDs for your EFS file system. This ensures that the EFS file system is created in the same network where your EKS cluster is running.
![image](https://user-images.githubusercontent.com/99641515/224278448-9f8cd54c-99fa-4d2a-b661-2c70ddfc4e16.png)


## Step 4: Skip the remaining options and create the EFS.

After selecting the VPC and Subnet IDs, you can skip the remaining options and create the EFS file system.

## Step 5: Copy the EFS ID

After the EFS file system is created, copy the EFS ID. You will need this ID in the next step.

## Step 6: Create kfs-shared.yml file and update the fileSystemId with the EFS ID

Create a Kubernetes storage class YAML file named kfs-shared.yml. Update the fileSystemId with the EFS ID that you copied in step 5. This YAML file defines the Kubernetes storage class for EFS.

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: kfs-shared
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: change-fs-id
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional
allowVolumeExpansion: true
reclaimPolicy: Delete
```

## Step 7: Apply this kfs-shared.yml

After updating the YAML file, apply it using kubectl apply -f kfs-shared.yml command. This will create a storage class in Kubernetes that can be used to provision EFS volumes dynamically.

```
kubectl apply -f  kfs-shared.yml
```

Once the above steps are completed, you can use this storage class to provision EFS volumes dynamically and mount them to your Kubernetes pods as persistent storage.
