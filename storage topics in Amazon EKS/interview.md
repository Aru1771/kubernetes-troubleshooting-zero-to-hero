
### EBS CSI Driver, Dynamic Provisioning, IRSA & StorageClass — Interview Answer

In our Kubernetes environment, we used the **AWS EBS CSI Driver** for persistent storage and dynamic provisioning of EBS volumes.

In **EKS**, we installed the Amazon EBS CSI Driver as an EKS add-on. In our **kubeadm-based Kubernetes cluster**, we installed and configured the EBS CSI Driver separately.

We created a **StorageClass** using the EBS CSI provisioner:

```yaml
provisioner: ebs.csi.aws.com
```

When an application creates a PVC, Kubernetes uses the StorageClass and the EBS CSI driver to dynamically provision an EBS volume. The CSI driver then creates the corresponding PV and binds it to the PVC.

### IAM and IRSA

Since the EBS CSI controller needs to communicate with AWS APIs such as EC2/EBS, we configured **IAM permissions** for the CSI driver's Kubernetes ServiceAccount.

In EKS, we used **IRSA (IAM Roles for Service Accounts)**.

The flow is:

```text
EBS CSI Controller Pod
        ↓
Kubernetes ServiceAccount
        ↓
Projected ServiceAccount Token
        ↓
EKS OIDC Provider
        ↓
AWS STS
        ↓
AssumeRoleWithWebIdentity
        ↓
Temporary AWS Credentials
        ↓
AWS EBS APIs
```

We created an IAM role with the required EBS permissions and configured its **trust policy** to trust the EKS cluster's OIDC provider.

We also restricted the trust policy using conditions so that only the required Kubernetes ServiceAccount could assume the IAM role.

The ServiceAccount was then annotated with the IAM role ARN:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ebs-csi-controller-sa
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<ebs-csi-role>
```

Kubernetes uses the **TokenRequest API** to obtain a short-lived ServiceAccount token. The kubelet makes the projected token available to the pod, and the AWS SDK/credential provider uses the web identity token to obtain temporary credentials from AWS STS.

This avoids storing long-lived AWS access keys inside the pod.

### Topology and Stateful Applications

Since EBS volumes are **Availability Zone-specific**, we also considered topology while provisioning volumes.

For example, we can use:

```yaml
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.ebs.csi.aws.com/zone
        values:
          - ap-south-1a
          - ap-south-1b
```

We also use:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

This helps Kubernetes consider the pod's scheduling requirements before dynamically provisioning the EBS volume, which is important for stateful workloads.

For StatefulSets, we can combine storage topology with node labels/affinity, taints and tolerations, and other scheduling mechanisms to ensure workloads run on appropriate nodes/AZs.

### Reclaim Policy

We configured the StorageClass with an appropriate reclaim policy depending on the requirement.

For example:

```yaml
reclaimPolicy: Delete
```

With `Delete`, when the PVC is deleted and the dynamically provisioned PV is released, the associated EBS volume is also deleted.

With:

```yaml
reclaimPolicy: Retain
```

the underlying EBS volume is retained, allowing an administrator to recover or reuse the storage.

The important distinction is that the reclaim policy primarily controls what happens to dynamically provisioned storage when the **PVC is deleted and the PV is released**. Manually deleting a PV object is a separate operation and should not simply be described as the reclaim policy being applied to PV deletion.

### End-to-End Flow

The complete flow is:

```text
Application
    ↓
PVC
    ↓
StorageClass
    ↓
EBS CSI Driver
    ↓
IAM / IRSA
    ↓
AWS STS
    ↓
AWS EBS API
    ↓
EBS Volume Created
    ↓
PV Created
    ↓
PVC Bound to PV
    ↓
Pod Mounts the Volume
```

For stateful applications, we also make sure that the volume's Availability Zone and the pod's scheduling requirements are compatible.

### Interview Summary

If I had to explain it briefly in an interview, I would say:

> "We used the AWS EBS CSI Driver for dynamic persistent storage in Kubernetes. In EKS, we installed it as an EKS add-on,
> while in our kubeadm cluster we installed and configured the CSI driver manually. We created StorageClasses using `ebs.csi.aws.com`, and when applications created PVCs,
> the CSI driver dynamically provisioned EBS volumes and created the corresponding PVs.
>
> For AWS authentication, we used IRSA. The EBS CSI controller ServiceAccount was associated with an IAM role,
> and the IAM trust policy trusted the EKS OIDC provider and restricted access to the required ServiceAccount.
> Kubernetes provides a projected ServiceAccount token, which is used with AWS STS to obtain temporary credentials instead of storing AWS access keys in the pod.
>
> Because EBS volumes are AZ-specific, we used topology-aware provisioning and `WaitForFirstConsumer` where appropriate,
> along with node affinity and taints/tolerations for workload placement. Finally, we configured the StorageClass reclaim policy
>  as either `Delete` or `Retain` depending on the application's storage requirements. With Delete,
> the dynamically provisioned EBS volume is removed when the PVC is deleted; with Retain, the storage is preserved for recovery or reuse."
