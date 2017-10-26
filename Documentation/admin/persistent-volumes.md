# Persistent Volumes

Persistent Volumes (PVs) provide storage resources in a cluster, and have a lifecycle independent of any individual Pod that uses the PV. This allows the storage resource to persist even when the Pods which use them are cycled.

Persistent Volumes may be statically or dynamically provisioned. They may be customized for use by defining properties such as performance, size, or access mode.

Storage classes provide the means to define the ‘class’ of available storage.

For more information, see [Persistent Volumes][persistent-volumes] in the Kubernetes documentation.

## Defining Default Storage Classes

When Persistent Volumes are statically provisioned, the `StorageClass` objects are requested by name. The `storageClassName` defined in the Persistent Volume must match the `metadata:
 name` defined in the `StorageClass` it references.

When Persistent Volumes are dynamically provisioned, the `StorageClass` fields `provisioner`, `parameters`, or `reclaimPolicy` may be used.

Specify a default `StorageClass` for PVCs that don’t request a specific class.

`StorageClass` objects cannot be updated once they are created.

For more information, see [StorageClasses][storage-classes] in the Kubernetes documentation.

### Defining Storage Classes for AWS

Use kubectl to create an AWS StorageClass:

```
kubectl create -f
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: aws-ebs-sc
provisioner: kubernetes.io/aws-ebs
```

### Defining Storage Classes for Azure

Tectonic supports both Azure Disks and Azure Files.
* Use Azure Files for dev and debugging tools that need access from many VMs.
* Use Azure Disks for data accessed only from within the VM to which it is attached.

For more information on Azure storage class types, see [When to use Azure Blobs, Azure Files, or Azure Disks][azure-blobs] in the Microsoft Azure documentation.

Use kubectl to create an Azure Disk `StorageClass` of `kind: managed`:

```
kubectl create -f
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
   name: azure-disk-managed
provisioner: kubernetes.io/azure-disk
parameters:
   storageaccounttype: Standard_LRS
   kind: managed
```

## Defining Persistent Volumes

Persistent Volumes may be defined statically for use across the cluster. Static Persistent Volume definitions must specify a `StorageClass`, which must exist before the Persistent Volume is defined.

To define Persistent Volumes, go to *Administration > Persistent Volumes*, and click *Create*. Enter the definition, and click *Create*.

The following example creates a Persistent Volume which references an existing `StorageClass`, using `storageClassName: slow`.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: EBSpv
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: aws://ca-central-1b/vol-0c9
  capacity:
    storage: 8Gi
  persistentVolumeReclaimPolicy: Delete
  storageClassName: aws-ebs-sc
```

### Azure Persistent Volumes

Tectonic uses Managed Disks for the VMs. As a result, a Persistent Volume Claim (PVC) using an Azure Disk storage class that has `kind: shared` (the default) will fail to be mounted by the container.

To define a PVC on Azure, create a `StorageClass` of `kind: managed`. Then, use those classes when creating dynamic or static Persistent Volumes.

## Defining Persistent Volume Claims

Persistent Volumes may also be defined dynamically, by creating a Persistent Volume Claim which references an existing `storageClassName`.

To define Persistent Volume Claims, go to *Workloads > Persistent Volume Claims*, and click *Create*. Enter the definition, and click *Create*.

The following example creates an AWS Persistent Volume Claim, by referencing an existing storage class by name, using `storageClassName: aws-ebs-sc`.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.alpha.kubernetes.io/storage-class: default
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/aws-ebs
  labels:
    app: db
  name: db-pv-claim
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: aws-ebs-sc
```


[azure-blobs]: https://docs.microsoft.com/en-us/azure/storage/common/storage-decide-blobs-files-disks
[storage-classes]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storageclasses
[persistent-volumes]: https://kubernetes.io/docs/concepts/storage/persistent-volumes
