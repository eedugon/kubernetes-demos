<a name="resources"></a>
# Scheduling

Table of Contents:

- [NodeSelector](#node-selector)
- [Node Affinity](#node)
- [Pod Affinity and Anti-Affinity](#pod)
- [Taints and Tolerations](#taints)

The following examples are based on a GKE cluster with 3 nodes.

- [NodeSelector](#node-selector) (basic scheduling)

<a name="node-selector"></a>
### NodeSelector

NodeSelector allows us to select which nodes can run our pods. It is the most basic level of scheduling in Kubernetes.

- The following pod can only be scheduled on nodes with the label `disktype=ssd`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

To deploy it:
```
kubectl apply -f resources/nodeSelector.yaml
```

If no nodes have that label, the pod will remain in a `Pending` state with the following message (from the describe output):

```
$ kubectl describe pod nginx-ssd
Events:
  Type     Reason             Age                From                Message
  ----     ------             ----               ----                -------
  Warning  FailedScheduling   18s (x2 over 20s)  default-scheduler   0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
  Normal   NotTriggerScaleUp  18s                cluster-autoscaler  pod didn't trigger scale-up:
```

To fix this, we can add the label to one of the nodes:

```
kubectl label node gke-keepcoding-default-pool-5a194f9d-phgm disktype=ssd
```

Once the label is added, the pod will start on the corresponding node.

<a name="node"></a>
## Node Affinity

For the node affinity tests, we'll add the label `apps=frontend` to one of our nodes:

```
kubectl label node node3 apps=frontend
```

The following example defines an NGINX deployment (with a single replica) with these affinity rules:
- Required: It cannot run on the node with the hostname (label `kubernetes.io/hostname`) `node3`.
- Preferred: It should run on nodes with the `apps=frontend` label.
- Additionally, the container requires `200m` vCPUs.

Edit the YAML file to include the hostname of one of the nodes that does not have the label.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-affinity
spec:
  selector:
    matchLabels:
      app: nginx-affinity
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: NotIn
                values:
                - gke-keepcoding-default-pool-f89b3810-51h7
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: apps
                operator: In
                values:
                - frontend
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
```

Deploy the NGINX pod and verify that it runs on the node with the correct label:

```
kubectl apply -f resources/nginx-affinity.yaml
```

Scale the deployment to 2 or 3 replicas and notice how all the pods run on the same node (until resources run out):
```
kubectl scale deployment nginx-affinity --replicas 2
```

Continue scaling it until no more pods can be added, and verify that none are scheduled on the node we excluded in the example.

<a name="pod"></a>
## Pod Affinity and Anti-Affinity

Let's assume we have a Redis database and a web application, and we want to:
- Ensure that Redis pods do not share the same host (`antiAffinity`).
- Ensure that web application pods are scheduled on different hosts, and only on nodes where Redis pods are running.

This can be implemented as follows:

- Redis: Pods cannot be scheduled (`antiAffinity`) on nodes (`topologyKey --> hostname`) that already have a pod with the label `redis=cache` (its own label). This separation is enforced (`required`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-pod-affinity
spec:
  selector:
    matchLabels:
      app: redis-cache
  replicas: 3
  template:
    metadata:
      labels:
        app: redis-cache
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - redis-cache
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

- Web Application: Separate pods from the same deployment (as before) and ensure that they are scheduled on nodes with Redis pods (`app=redis-cache`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server-pod-affinity
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - redis-cache
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

Start by deploying the web application service and observe what happens:

```
kubectl apply -f resources/web-server-pod-affinity.yaml
```

At this point, everything remains in `Pending`, which is expected due to the rules and the fact that Redis is not yet running on any nodes.

If you describe any of the pods, you will see something like this:

```
Events:
  Type     Reason             Age                From                Message
  ----     ------             ----               ----                -------
  Normal   NotTriggerScaleUp  97s                cluster-autoscaler  pod didn't trigger scale-up:
  Warning  FailedScheduling   12s (x3 over 99s)  default-scheduler   0/3 nodes are available: 3 node(s) didn't match pod affinity rules, 3 node(s) didn't match pod affinity/anti-affinity rules.
```

Now, deploy Redis and observe that everything starts as expected:
```
kubectl apply -f resources/redis-pod-affinity.yaml
```

<a name="taints"></a>
## Taints and Tolerations

To add a taint:
```
kubectl taint nodes host1 special=true:NoSchedule
```

To remove a taint, add a `-` at the end of the taint definition:
```
kubectl taint nodes host1 special:NoSchedule-
```

For example, if we want to dedicate a series of nodes for monitoring applications, we could add a taint to those nodes like this:
```
kubectl taint nodes monitoring-node01 area=monitoring:NoSchedule
```

The monitoring workloads would need a toleration like this to be scheduled on those nodes:

```yaml
        tolerations:
          - key: "area"
            operator: "Equal"
            value: "monitoring"
            effect: "NoSchedule"
```

But __remember__! Tolerations in pods do not prevent them from running on nodes without taints. A toleration simply means the pod can tolerate the taint but doesn't force it to run on tainted nodes.

When dedicated nodes are required, a __combination of taints/tolerations with affinity rules__ is typically used unless all nodes in the cluster have taints.

Lastly, here's an example of a toleration that accepts any kind of taint. This is useful, for example, in a `DaemonSet` that should bypass taints and run on all nodes.

```yaml
        # tolerate any taint (we want it to run on all nodes). Irrelevant if no taints are used in k8s.
        tolerations:
          - effect: NoExecute
            operator: Exists
          - effect: NoSchedule
            operator: Exists
          - key: CriticalAddonsOnly
            operator: Exists
```

## End
[Back](./README.md)
