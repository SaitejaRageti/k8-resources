📘 EFS & EBS in Kubernetes – Static vs Dynamic Provisioning
1. EBS in Kubernetes

Type: Block storage (per pod, single node unless using EBS Multi-Attach).

Provisioning Methods:

🔹 Static Provisioning

Admin creates the PersistentVolume (PV) in advance with EBS volume ID.

PVC must match the PV spec.

Example (Static EBS PV + PVC):

Pre-created EBS volume in AWS (e.g. vol-0abcd12345)

🔹 How to Create an EBS Volume (to use in K8s static provisioning)
Step 1: Create the EBS Volume
✅ Via AWS Console

Go to EC2 → Volumes → Create Volume.

Choose:

Volume Type: e.g., gp3.

Size: e.g., 10 GiB.

Availability Zone (AZ): Must match the AZ of the EC2 worker node that will use it (e.g., us-east-1a).

Encryption (optional).

Click Create Volume.

You will now see the Volume ID (like vol-0123456789abcdef0).

apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-0abcd12345
    fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

🔹 Dynamic Provisioning

Uses StorageClass with EBS CSI driver.

PVC automatically provisions an EBS volume on AWS.

Example (Dynamic EBS):

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ebs-sc


👉 A new 20Gi EBS volume is created automatically when the PVC is bound.

2. EFS in Kubernetes

Type: Shared file storage (multi-node, multi-pod).

Requires EFS CSI Driver.

🔹 Static Provisioning

Admin creates a PV using an existing EFS file system ID.

Step 1: Create the EFS File System
✅ Using AWS Console

Go to EFS service in AWS Console.

Click Create file system.

Choose:

VPC → must be the same VPC where your worker nodes run.

Availability Zones / Subnets → pick the same AZs as your worker nodes.

Performance mode → General Purpose (default) or Max I/O.

Throughput mode → Bursting (default) or Provisioned.

Encryption → optional.

Click Create.

Once created, you’ll see a File System ID (e.g., fs-12345678).

📌 That fs-12345678 is what you’ll reference in Kubernetes.

# Example (Static EFS PV + PVC):

apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-12345678  # EFS FileSystem ID
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi

# 🔹 Dynamic Provisioning

# With EFS Access Points, PVCs automatically create new subdirectories inside the EFS file system.

# Example (Dynamic EFS):

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "700"
reclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: efs-sc


# 👉 A new EFS access point & subdirectory is dynamically created when PVC is applied.

# 3. Comparison in K8s
# Storage	Static Provisioning	Dynamic Provisioning
# EBS	PV must reference existing EBS volume	PVC triggers new EBS volume creation
# EFS	PV uses existing EFS filesystem	PVC creates subdir (via Access Point)

# ✅ Exam-style takeaway:

# EBS → Good for single pod (RWO).

# EFS → Good for multiple pods/nodes (RWX).

# Dynamic provisioning saves admin effort by auto-creating volumes or directories.