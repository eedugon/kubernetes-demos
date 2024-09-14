<a name="pod"></a>
# StatefulSets and Storage

To fully understand this demonstration, it is essential to be familiar with the following concepts:
- Pods
- DNS resolution within the cluster
- Headless Services
- PersistentVolumes and PersistentVolumeClaims
- StatefulSets

The demonstration assumes that the cluster is capable of dynamically creating `PersistentVolumes`.

Table of Contents:

- [Creating a StatefulSet and key considerations](#creation)
- [Basic checks](#checks)
- [Exposing StatefulSet externally via LoadBalancer](#expose)
- [Scaling StatefulSets](#scale)
- [Applying changes, rolling updates](#update)
- [Deleting the StatefulSet and preserving data](#delete)

<a name="operations"></a>

## Creating a StatefulSet

For this demonstration and to describe the `StatefulSets`, we will use `nginx` (for simplicity), although it is not typically associated with Stateful workloads.

We'll start with an NGINX StatefulSet with 2 replicas, which includes a __PersistentVolumeClaim__ that will manage the creation of individual disks for our pods.

The StatefulSet requires an associated `Headless` service, so we'll include it in our YAML definition:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless # must match the 'serviceName' in the StatefulSet
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # required to make it headless
  selector:
    app: stateful-demo # matches the pods
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: stateful-demo # must match .spec.template.metadata.labels
  serviceName: "nginx-headless" # must match the headless service name
  replicas: 2 # default is 1
  template:
    metadata:
      labels:
        app: stateful-demo # must match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

Create the StatefulSet:
```
kubectl apply -f resources/statefulset-web.yaml
```

__NOTE__: The YAML will fail. Investigate why and fix it!

__KEY CONSIDERATIONS ABOUT THE MANIFEST__:

* Selector: Like in Deployments, it must target the pods.
* As in Deployments, the `spec.template` block starts the pod definition.
* A headless service (specified in `serviceName`) is required.
* `volumeClaimTemplates`: Defines PVCs (`PersistentVolumeClaims`) that request the system to create PVs (`PersistentVolumes`). The final volume will be mounted in the location specified in the pod's `volumeMounts` block.
* `storageClassName` must be defined in the system.
* Maintenance and scaling of the `StatefulSet` can be done the same way as with `Deployments`.
* __Note__: To expose the `StatefulSet` externally, you need to create a separate Service without modifying the headless service.

__Troubleshooting__:

- Ensure that the associated PVCs and PVs are created correctly, or the pods won’t be able to start.
- Always run a `describe` on the pods if they are in `Pending` or `Init` state.
- Check PVCs and PVs with `kubectl get pvc` and `kubectl describe pvc xxxxx`.
- Verify that the `storageClassName` exists in the system (`kubectl get storageclasses`).

<a name="checks"></a>
## Basic Checks

- Start by reviewing the created resources:

  ```
  kubectl get pods -l app=stateful-demo
  kubectl get service nginx-headless
  kubectl get statefulset web
  kubectl get pvc
  kubectl get pv
  ```

Check the pod names — they have a static identity.

  - Verify the `hostnames` of each pod:

  ```
  $ for i in 0 1; do kubectl exec "web-$i" -- sh -c 'hostname'; done
  web-0
  web-1
  ```

  - Run a `busybox:1.28` pod to perform DNS checks:

  ```
  $ kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm
  ```

  Possible DNS resolutions:

  ```
  # Direct resolution of one of the pods
  nslookup web-0.nginx-headless

  # The headless service
  nslookup nginx-headless
  nslookup nginx-headless.default.svc.cluster.local
  ```

__Troubleshooting__: DNS resolution issues? Check the `StatefulSet` manifest and make sure that `serviceName` matches the headless service name.

  - Delete one or all pods (you’ll observe how they automatically restart):

  ```
  kubectl delete pod web-0
  kubectl delete pod -l app=stateful-demo
  ```

  Recheck the hostnames and verify that they remain the same.

  - Demonstrating independent persistence:

  Create an `index.html` file on each of the NGINX instances (`/usr/share/nginx/html/` is the mount point for the persistent volume):

  ```
  for i in 0 1; do kubectl exec "web-$i" -- sh -c 'echo "$(hostname)" > /usr/share/nginx/html/index.html'; done
  ```

  Then, perform a curl request on each pod to check the page they serve:
  ```
  $ for i in 0 1; do kubectl exec -i -t "web-$i" -- curl http://localhost ; done
  web-0
  web-1
  ```

<a name="expose"></a>
## Exposing the StatefulSet Externally

  - Expose the application with an external LoadBalancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-web-lb
spec:
  type: LoadBalancer
  selector:
    app: stateful-demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Create the LoadBalancer:

```
  kubectl apply -f resources/svc-web-lb.yaml
```

Wait for the external load balancer to be created, then access the application via your browser.

<a name="scale"></a>
## Scaling StatefulSets

Scaling can be done in the same way as with deployments, but it is more logical to modify the original YAML file and reapply it with `kubectl apply -f` to ensure that your configuration reflects the actual state.

(Option 1)
```
kubectl scale sts web --replicas=5
```

Another way to apply these changes is with `kubectl patch`:

(Option 2)
```
kubectl patch sts web -p '{"spec":{"replicas":3}}'
```

As mentioned, the ideal approach is to modify the original YAML file and reapply it:

(Option 3)
```
vi resources/statefulset-web.yaml # to change the replicas.
kubectl apply -f resources/statefulset-web.yaml
```

<a name="rolling"></a>
## Applying Changes: Rolling Updates

Update strategies are defined in the `updateStrategy` field. The most common value is `RollingUpdate`.

We can update the container image to a new version and apply the changes. The methods are the same as when modifying deployments:

- Edit the original YAML and apply the changes (__recommended__).
- Directly edit the Kubernetes object (`kubectl edit sts web`).
- Apply the change using `kubectl patch`.

To observe the rolling update in action, in one terminal, run the following command:

```
kubectl get pod -l app=stateful-demo -w
```

In another window, trigger a change (for example, update the image to `nginx-slim:0.7`).

<a name="delete"></a>
## Deleting the StatefulSet and Its Data

Resources to consider when deleting:
- StatefulSet
- Associated Services
- PersistentVolumeClaims and PersistentVolumes.

To delete the `StatefulSet` only:

```
kubectl delete sts web
```

This will remove the StatefulSet definition and pods but leave the PVC (`PersistentVolumeClaim`) intact.

If you want to delete the persistent data/disks on all nodes, you will need to delete the PVCs manually:

```
kubectl delete pvc www-web-1
...
...
```

If for some reason you want to delete the StatefulSet but keep the pods running, you can use the `--cascade=orphan` option:

```
kubectl delete sts web --cascade=orphan
```

## Complete Cleanup

A complete cleanup can be done as follows:

Delete everything created by the original `yaml` (StatefulSet and service), as well as the `LoadBalancer` we created:

```
kubectl delete -f resources/statefulset-web.yaml
kubectl delete -f resources/svc-web-lb.yaml
```

__CAUTION__: The next step will delete the disks/volumes and all associated data.

Delete PVCs:

```
kubectl delete pvc -l app=stateful-demo
```

## End
[Back](./README.md)
