# Kubernetes Storage

Kubernetes can interface with many types of storage from many different places. 

No matter what or where the storage comes from, when it is exposed on the Kubernetes cluster it is called a volume. Different storage types and providers connect to Kubernetes via the Container Storage Interface (CSI).

The CSI talks to the Kubernetes persistent volume subsystem. This is a set of API objects that allow applications to consume storage.
* PersistentVolumes (PV) are how external storage is mapped on the cluster
* PersistentVolumeClaims (PVC) are similar to tickets that authorise applications (Pods) to use a PV

Points worth noting are:
* There are rules safeguarding access to a single volume from multiple Pods
* A single external storage volume can only be used a single PV

From a day-to-day Kubernetes interaction perspective, a developers only real interaction with the CSI will be referencing the appropriate interface plugin in your YAML manifest file (if you are not actively developing these plugins).

## Kubernetes Persistent Volume Subsystem

The three main resources in a Persistent Volume Subsystem are: 
* Persistent Volumes (PV)
* Persistent Volume Claims (PVC)
* Storage Classes (SC)

SCs can make PVs and PVCs dynamic

## An example

The following YAML file creates a PV object that maps back to the pre-created Google Persistent Disk called "uber-disk". The YAML file is called `gke-pv.yml`.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: test
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: uber-disk
```

A quick explanation of some properties

`.spec.accessModes` defines how the PV can be mounted. Three options exist: 
* `ReadWriteOnce` (RWO): defines a PPV can only be mounted/ bound as R/W by a single PVC. attempts from multiple PVCs to bind (claim) it will fail
* `ReadWriteMany` (RWM): defines a PV that can be bound as R/W by multiple PVCs. This is usually only supported by file and object storage such as NFS. Block storage usually only supports `RWO`
* `ReadOnlyMany` (ROM): defines a PV that can be bbound by multiple PVCs as R/O

A PV can only be opened in one mode. It is not possible for a single PV to have a PVC bound to it in different modes.

Pods do not act directly on PVs. They act on the PVC object that is bound to the PV

`.spec.storageclassName` tell Kubernetes to group this PV in a storage class called "Test". This is needed to make sure the PV will correctly bind with a PVC in a later step.

## Persistent Volume Reclaim Policy

Another property is `.spec.persistentVolumeReclaimPolicy`. this tells Kubernetes what to do with a PV when its PVC has been released. There are two policies:
* `Delete`
* `Retain`

### Delete

This is the default for PVs created dynamically via storage classes. This policy deleted the PV and associated storage resources on the external storage system. This results in data lost and should be used with caution

### Retain

Retain will keep the associated `PV` object on the cluster as well as any data stored on the associated external asset. This will prevent another PVC from using the PV in the future.

To reuse a retained PV you must perform the following steps
1. Manually delete the PV on Kubernetes
2. Reformat the associate storage asset on the external storage system
3. Recreate the PV

### Capacity

`.spec.capacity` tells Kubernetes how big the PV should be. The value can be less than the actual physical storage asset but not more.

The last line of the YAML links the PV to the name of the pre-created device on the backend.


The following command will create the PV above

```
kubectl apply -f gke-pv.yml
```

Which will output the following:

```
persistentvolume/pv1 created
```

Check that the PV exists

```
kubectl get pv pv1
```

Which will output the following:

```
NAME   CAPACITY   MODES   RECLAIM POLICY   STATUS      STORAGECLASS ...
pv1    10Gi       RWO     Retain           Available   test
```

## Persistent Volume Claim

The following YAML defines a PVC that can be used by a Pod to gain access to the pv1 PV created earlier. This file is called `gke-pvc.yml`.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: test
  resources:
    requests:
      storage: 10Gi
```
__Important__: The values in the `.spec` section must match with the PV the PVC is binding with

We can deploy the PVC with

```
kubectl apply -f gke-pvc.yml
```

Which will output:

```
persistentvolumeclaim/pvc1 created
```

To check that the PCV correctly binded to the PV we can

```
kubectl get pvc pvc1
```

Which will output:

```
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
pvc1   Bound    pv1      10Gi       RWO            test
```

We can deploy a single Pod to show how to an app can use a PV as an example. The file is called `volpod.yml`

```
apiVersion: v1
kind: Pod
metadata:
  name: volpod
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc1
  containers:
  - name: ubuntu-ctr
    image: ubuntu:latest
    command:
    - /bin/bash
    - "-c"
    - "sleep 60m"
    volumeMounts:
    - mountPath: /data
      name: data
```

`.spec.volumes` is to reference to the previously created PVC

Deploy the Pod with the following command.

```
kubectl apply -f volpod.yml
```

Which will output

```
pod/volpod created
```

`kubectl describe pod volpod` can be used to see that the Pod is successfully using the data volume and the `pvc1` claim.

## Storage Classes and Dynamic Provisioning

What has been outline in the examples above is correct and Fundamental to Kubernetes storage, but does not scale.

Storage classes allow the definition of different classes / tiers of storage.

Storage classes are defined the `storage.k8s.io/v1` API group with the resource type being StorageClass. They are referred to as `sc` in shorthand for `kubectl`

The following YAML defines a Storage Class called "fassed" that is based on AWS SSDs (io1). IT also specifies a performance of 10 IOPS per gigabyte.

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  zones: ap-southeast-1
  iopsPerGB: "10"
```

The `provisioner` field tells Kubernetes which plugin to use and `parameters` is to finely tune the type of storage to leverage from the back-end

## Multiple StorageClasses

Each defined StorageClass must relate to a single Provisioner. If an application needs multiple different Provisioners multiple StorageClasses must be defined.

Here is an example of a different StorageClass

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: portworx-db-secure
provisioner: kubernetes.io/portworx-volume
parameters:
  fs: "xfs"
  block_size: "32"
  repl: "2"
  snap_interval: "30"
  io-priority: "medium"
  secure: "true"
```

## Implementing StorageClasses

The basic workflow for deploying and using a StorageClass on the cluster is as follows:

1. Create your Kubernetes cluster with a storage backend.
2. Ensure te plugin for the storage backend is available
3. Create a StorageClasse object
4. Create a PVC object that references the StorageClass by name
5. Deploy a Pod that uses a volume based on the PVC

The workflow does not include creating a PV. This is because storage classes create PVs dynamically

The Following YAML contains the definitions for a SC, PVC and Pod.

The PodSpec references the PVC by name which refers to the SC by name

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast  # Referenced by the PVC
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc  # Referenced by the PodSpec
  namespace: mynamespace
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast    # Matches name of the SC
---
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: mypvc  # Matches PVC name
  containers: ...
  <SNIP>
```