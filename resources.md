<a name="resources"></a>
# CPU and Memory Resources

Table of Contents:

- [Namespace and Initial Setup](#start)
- [Checking CPU Limits](#cpu)
- [Checking Memory Limits](#memory)

<a name="start"></a>
## Prerequisites and Initial Setup

For this demonstration, you need to have the `metrics-server` or a monitoring system deployed in Kubernetes.

In Minikube, you can install `metrics-server` using `minikube addons enable metrics-server`.
In GKE, it is already installed.

To check if it's running:

```
$ kubectl get apiservices | grep -i metric
v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        129m
```

For this demonstration, we will create a `Namespace` called `resources-demo`:

```
kubectl create namespace resources-demo
```

<a name="cpu"></a>
## Checking CPU Limits

We will use the following definitions:

- Pod `cpu-demo` to stress the system:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
  namespace: resources-demo
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
```

Check the CPU usage of the created pod with:
```
kubectl top pod cpu-demo -n resources-demo
```

- Pod `cpu-demo-high` with high CPU request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-high
  namespace: resources-demo
spec:
  containers:
  - name: cpu-demo-ctr-2
    image: vish/stress
    resources:
      limits:
        cpu: "100"
      requests:
        cpu: "100"
    args:
    - -cpus
    - "2"
```

What happens with the previous pod? Find out and troubleshoot it.

<a name="memory"></a>
## Checking Memory Limits

- We will use the following pod, which has a memory limit of 200MB and will allocate 150MB upon startup:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: resources-demo
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

- This next pod has the same memory limit but will try to allocate 250MB:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-high
  namespace: resources-demo
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

If you observe this pod with `kubectl get pod`:

```
NAME               READY   STATUS      RESTARTS   AGE
memory-demo        1/1     Running     0          2m56s
memory-demo-high   0/1     OOMKilled   2          36s
```

To monitor events in real-time:
```
$ kubectl get pod -w

memory-demo-high   1/1     Running            2          22s
memory-demo-high   0/1     OOMKilled          2          23s
```

For more details, you can run:

```
$ kubectl get pod memory-demo-high -o=yaml
...
...
    state:
      terminated:
        containerID: containerd://4d8d77579c6a684d89b746f1acb4d6191305e86855fc8d888255930f4fbbebdb
        exitCode: 1
        finishedAt: "2022-02-07T12:34:45Z"
        reason: OOMKilled
        startedAt: "2022-02-07T12:34:44Z"
```

Also, checking the node where the pod is running (`kubectl get pod -o=wide`) and running a `describe` on it might provide additional insights:

```
kubectl describe node gke-demo-default-pool-7b2ab140-b7vr

Events:
  Type     Reason      Age                  From            Message
  ----     ------      ----                 ----            -------
  Warning  OOMKilling  7m59s                kernel-monitor  Memory cgroup out of memory: Killed process 100490 (stress) total-vm:256776kB, anon-rss:100584kB, file-rss:268kB, shmem-rss:0kB, UID:0 pgtables:236kB oom_score_adj:975
  Warning  OOMKilling  7m57s                kernel-monitor  Memory cgroup out of memory: Killed process 100545 (stress) total-vm:256776kB, anon-rss:100584kB, file-rss:268kB, shmem-rss:0kB, UID:0 pgtables:244kB oom_score_adj:975
```

## End
[Back](./README.md)
