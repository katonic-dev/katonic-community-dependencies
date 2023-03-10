# Enable Cluster Autoscaler in AWS EKS cluster

> Note: Create an AWS EKS cluster using the eksctl command line and use the --asg-access option.

### Step 1: Take EKS cluster access.
aws eks --region example_region update-kubeconfig --name cluster_name

### Step 2: Create an IAM policy
Create an IAM policy that grants the permissions that the Cluster Autoscaler requires to use an IAM role. Replace all of the example values with your own values throughout the procedures.

Create an IAM policy.

Update the cluster name and save the following contents to a file that's named cluster-autoscaler-policy.json. 

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/my-cluster": "owned"
                }
            }
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeAutoScalingGroups",
                "ec2:DescribeLaunchTemplateVersions",
                "autoscaling:DescribeTags",
                "autoscaling:DescribeLaunchConfigurations",
                "ec2:DescribeInstanceTypes"
            ],
            "Resource": "*"
        }
    ]
}
```

b. Create the policy with the following command. You can change the value for policy-name.

```
aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json
```
OR

You can create this policy from AWS console also.

i. Go to IAM Policies 

ii. Click on create policy and click on JSON.

iii. Now put the policy here


### Step 3:  Create IAM OIDC identity provider
To enable the Cluster Autoscaler to access the EKS cluster, you need to create an IAM OIDC identity provider. [Follow this documentation to create OIDC identity provider](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

### Step 4: Create IAM role and attach IAM Policy
You can create an IAM role and attach an IAM policy to it using eksctl or the AWS Management Console. Select the desired tab for the following instructions. There are two ways to do this 

A. By using eksctl command line
Run the following command if you created your Amazon EKS cluster with eksctl. If you created your node groups using the --asg-access option, then replace AmazonEKSClusterAutoscalerPolicy with the name of the IAM policy that you created and replace the iam_number with you AWS iam numver

```
eksctl create iamserviceaccount \
  --cluster=${EksCluster} \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::iam_number:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```
 

B. By AWS Management Console
* Open the IAM console at https://console.aws.amazon.com/iam/.

* In the left navigation pane, choose Roles. Then choose Create role.

* In the Trusted entity type section, choose Web identity.

* In the Web identity section:

    * For Identity provider, choose the OpenID Connect provider URL for your cluster (as shown in the cluster Overview tab in Amazon EKS).

    * For Audience, choose sts.amazonaws.com.

* Choose Next.

* In the Filter policies box, enter AmazonEKSClusterAutoscalerPolicy. Then select the check box to the left of the policy name returned in the search.

* Choose Next.

* For Role name, enter a unique name for your role, such as AmazonEKSClusterAutoscalerRole.

* For Description, enter descriptive text such as Amazon EKS - Cluster autoscaler role.

* Choose Create role.

*  After the role is created, choose the role in the console to open it for editing.

* Choose the Trust relationships tab, and then choose Edit trust policy.

* Find the line that looks similar to the following:

```
"oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
```

Change the line to look like the following line. Replace EXAMPLED539D4633E53DE1B71EXAMPLE with your cluster's OIDC provider ID. Replace region-code with the AWS Region that your cluster is in.

```
"oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:kube-system:cluster-autoscaler"
```
Choose Update policy to finish.

 

### Step 5: Create cluster-autoscaler-autodiscover.yaml file
The next step is to create a YAML file named cluster-autoscaler-autodiscover.yaml that specifies the configuration for the Cluster Autoscaler deployment. This file should include information about the IAM role and the EKS cluster.

```
cluster-autoscaler-autodiscover.yaml
```

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources:
      - "namespaces"
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create","list","watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
            - --balance-similar-node-groups
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt #/etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"

```

### Step 6. Apply this cluster-autoscaler-autodiscover.yaml file
```
kubectl apply -f /root/platform-deployment/tasks/EKS/cluster-autoscaler-autodiscover.yaml
```

### Step 7. Patch the deployment to add the cluster-autoscaler.kubernetes.io/safe-to-evict annotation
To ensure that the Cluster Autoscaler can safely evict pods during scale-down events, you need to patch the deployment to add the "cluster-autoscaler.kubernetes.io/safe-to-evict" annotation to the Cluster Autoscaler pods. 

This can be done using the following command:

```
kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
```

### Step 8. 
Open the Cluster Autoscaler releases page from GitHub in a web browser and find the latest Cluster Autoscaler version that matches the Kubernetes major and minor version of your cluster. 

For example, if the Kubernetes version of your cluster is 1.25, find the latest Cluster Autoscaler release that begins with 1.25. Record the semantic version number (1.25.n) for that release to use in the next step.

Set the Cluster Autoscaler image tag to the version that you recorded in the previous step with the following command. Replace 1.25.n with your own value.

```
kubectl set image deployment cluster-autoscaler \
  -n kube-system \
  cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.25.0
```