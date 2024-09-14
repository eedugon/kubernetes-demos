<a name="resources"></a>
# Volumes

Table of Contents:

- [Basic Concepts](#concepts)
- [Creating an External Volume](#setup)
- [Pod Declaration](#pod)

<a name="concepts"></a>
## Basic Concepts

Summary of concepts:
- `Volumes`: Defined within pods and can be of many types, including:
  - `ConfigMap`
  - `Secret`
  - `EmptyDir` / `Hostpath` (local storage)
  - Network storage --> `gcePersistentDisk` (GCP disks).
  - ...

- `PersistentVolumes` (PVs): Persistent storage with a lifecycle managed by Kubernetes. PVs can be created manually or dynamically using `StorageClasses` and `PersistentVolumeClaims` (PVCs).

- Pods consume storage through `Volumes` or `volumeClaimTemplates`. Pods do not directly consume `PersistentVolumes`.

- For a pod to be associated with a `PersistentVolume`, both a PV and a PVC must be created, and the pod must be linked to the PVC (using `volumeClaimTemplates`). You can find a good example of manually created pod, PV, and PVC [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/).

In this demonstration, we'll use a Google Cloud `GKE` cluster and manually provisioned `gcePersistentDisk` volumes, without PVs or PVCs. We'll explore PVs, PVCs, and StorageClasses in the `StatefulSets` demonstration.

<a name="setup"></a>
## Initial Volume Setup

We create a disk in our cloud environment in the same region where the Kubernetes cluster is located.

```
gcloud compute disks create --size=10GB --zone=europe-west1-b my-data-disk

# Example:
# gcloud compute disks create my-data-disk --project=keep-contenedores --type=pd-balanced --size=10GB --zone=europe-west1-b
```

Next, we create a pod that mounts the volume at the `/test-pd` directory.

- Reference:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pd-test
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

- Create the POD:

```
kubectl apply -f resources/volume-pd-test.yaml
```

Once the pod is created, the pod's description (`kubectl describe`) should return something like this:
```
Events:
  Type    Reason                  Age   From                     Message
  ----    ------                  ----  ----                     -------
  Normal  Scheduled               9s    default-scheduler        Successfully assigned default/volume-pd-test to gke-demo-default-pool-7b2ab140-b7vr
  Normal  SuccessfulAttachVolume  5s    attachdetach-controller  AttachVolume.Attach succeeded for volume "test-volume"
  ...
  ...
```

You can connect to the pod and write something to the volume with:
```
kubectl exec -it volume-pd-test -- bash

# Once inside the pod
echo hello > /test-pd/test-file.txt
exit
```

Now delete the pod:
```
kubectl delete pod volume-pd-test
```

Recreate the pod:
```
kubectl apply -f resources/volume-pd-test.yaml
```

Check if the data still exists:
```
kubectl exec -it volume-pd-test -- ls -l /test-pd

total 24
-rw-r--r-- 1 root root     5 Feb  7 15:03 test-file.txt
```

Finally, clean up everything by deleting the pod and removing the disk from GCP:
```
kubectl delete -f resources/volume-pd-test.yaml
gcloud compute disks delete --zone=us-west1-a my-data-disk
```

__Conclusions__

- Managing volumes in this way would require manual volume creation and deletion by an administrator.
- Typically, you would work with `StorageClasses`, `PersistentVolumeClaims`, and `PersistentVolumes` for more efficient storage management.

## End
[Back](./README.md)
