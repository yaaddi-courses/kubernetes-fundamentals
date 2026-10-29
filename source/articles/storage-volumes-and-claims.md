# Storage: volumes, persistent volumes and claims

**Covers:** Volume, PersistentVolume, PersistentVolumeClaim, PV, PVC, StorageClass

## What happens to data when a pod dies {#overview}
Anything a container writes to its own file system disappears when the container is replaced. For a web server, that is fine. For a database, it is a disaster. Kubernetes answers by separating two questions: "where is the storage?" and "what does the application need?". A developer can then ask for storage without knowing how it is provided. We will build this up from the simplest piece.

## Volume {#volume}
A volume is storage attached to a pod that outlives a single container restart. Without one, whatever a container wrote to its local file system is lost the moment the container is replaced. You declare a volume in the pod and mount it at a path in each container that needs it.

There are many kinds. A simple `emptyDir` lives as long as the pod and is shared by its containers, which makes it a useful scratch area. Others connect to network disks, cloud storage or a configuration object. Notice the limit: a plain volume lasts only as long as the pod, so data that must survive the pod needs persistent storage.

## PersistentVolume {#persistentvolume}
A PersistentVolume is real storage that was prepared in the cluster ahead of time, waiting for someone to claim it. It belongs to the cluster, not to any pod or namespace. It records details such as the size, how it can be accessed and where the storage actually is, for example a cloud disk or a network share. Its life is independent of any pod, so the data stays when pods come and go.

## PersistentVolumeClaim {#persistentvolumeclaim}
Now the other side. A PersistentVolumeClaim is a pod's request for storage, and it is separate from the PersistentVolume that supplies it. The claim states the need: "10 gigabytes, readable and writable by one node at a time". Kubernetes finds a PersistentVolume that fits and binds the two together. The pod refers only to the claim.

The split is deliberate. Developers describe their needs, and administrators decide what is provided. The same application definition works on a laptop cluster and in the cloud, because only the supply side differs.

## PV {#pv}
PV is the everyday short form of PersistentVolume. You will see it in command output. For example, `kubectl get pv` lists the volumes, their sizes, whether each is `Available` or `Bound`, and which claim holds it.

## PVC {#pvc}
PVC is the short form of PersistentVolumeClaim. `kubectl get pvc` lists the claims in a namespace with their status. A status of `Pending` means that no matching storage was found, and that is the first thing to check when a pod will not start.

## StorageClass {#storageclass}
Preparing disks by hand for every claim would be slow. A StorageClass lets Kubernetes create storage automatically at the moment a claim needs it. It names a provisioner, such as a cloud disk service, and parameters such as the disk type. A claim that names a class triggers the creation of a fresh volume that matches.

This is called dynamic provisioning, and most cloud clusters work this way. Classes can describe tiers, such as `fast` for solid-state disks and `cheap` for slower ones. A further option, the reclaim policy, decides whether the disk is deleted or kept when the claim is removed. For important data, keep it.

## A worked example {#example}
Let's give a database pod a claim of 20 gigabytes:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shop-db-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: shop-db
spec:
  containers:
    - name: db
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: shop-db-data
```

Here is what happens:

1. The claim `shop-db-data` asks for 20 Gi from the class `fast`.
2. The StorageClass creates a 20 Gi cloud disk on demand, and a PersistentVolume for it, and binds the claim to it.
3. The pod mounts the claim, and the database writes its files to the disk.
4. If the pod is deleted and recreated, even on another node, the new pod attaches the same claim and finds the data.

Check the status with `kubectl get pvc shop-db-data`. `Bound` means it worked, and `Pending` means nothing matched.

Now count the cost of a mistake. Had the database used a plain volume, a pod restart would erase everything. With 20 Gi of data and a cloud disk costing about 0.10 dollars per gigabyte a month, the persistent disk costs about 2 dollars a month. That is a small price against losing the database.

The reclaim policy matters here. With `Delete`, removing the claim deletes the disk and the data. For a database, choose `Retain`, so an accidental deletion of the claim leaves the data available for recovery.

## A common mistake {#mistake}
A common slip is keeping a database on a plain volume or in the container's own file system, and then losing the data at the next restart. Use a claim.

Another slip is forgetting the reclaim policy, so deleting the claim also deletes the disk.

## Key points {#key-points}
- A plain volume survives a container restart but not necessarily the pod. A PersistentVolume outlives pods.
- A PVC asks for storage, and the cluster binds it to a PV. Pods refer only to the claim.
- A StorageClass creates storage on demand, which is dynamic provisioning.
- Keep the reclaim policy on `Retain` for data that matters.
