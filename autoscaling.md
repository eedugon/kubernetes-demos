<a name="autoscaling"></a>
# Autoscaling

Table of contents:

- [Initial Setup](#start)
- [Horizontal Pod Autoscaling (HPA)](#hpa)
- [Cluster Autoscaling](#cluster)

<a name="start"></a>
## Initial Setup

HPA functionality depends on the Kubernetes metrics server (installed by default with most installers or SaaS environments). In Minikube, you can enable the metrics server with the following command:

```bash
minikube addons enable metrics-server
```

We create an NGINX deployment exposed via a service:
- Single replica
- requests.cpu: 100m
- requests.limit: 200m

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hpa
spec:
  selector:
    matchLabels:
      app: nginx-hpa
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-hpa
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 200m
```

Create the deployment:

```bash
kubectl apply -f resources/nginx-hpa.yaml
```

We create the service using `kubectl expose` (best practice: `YAML with defined service`):
```bash
kubectl expose deployment nginx-hpa --port 80
```

We check that the `nginx-hpa` service is created.

<a name="hpa"></a>
## Horizontal Pod Autoscaling (HPA)

```bash
kubectl autoscale deploy nginx-hpa --cpu-percent=20 --min=2 --max=20
```

We can view the `spec` of the created resource with:

```bash
$ kubectl get hpa nginx-hpa -o=yaml
```

The equivalent `yaml` would be (Kubernetes version 1.20):

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  maxReplicas: 20
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-hpa
  targetCPUUtilizationPercentage: 20
```

__NOTE!!__ Autoscaling in version 1.23 changed the API to `v2`, and the syntax has changed. Additionally, many more types of metrics are supported.

In Kubernetes 1.23, the `yaml` would look like this:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  maxReplicas: 20
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 20
        type: Utilization
    type: Resource
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-hpa
```

(This is a great example of how the spec of some resources evolves as Kubernetes progresses.)

Now, we will deploy a load generator, for example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  selector:
    matchLabels:
      app: load-generator
  replicas: 1
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: busybox
        image: busybox:1.28
        command: ["sh"]
        args: ["-c", "while true; do wget -q -O- http://nginx-hpa >dev/null 2>&1; done"]
```

Another way to manually launch something similar is with this command:

```bash
kubectl run load-generator-manual --image=busybox --command -- sh -c 'while true; do wget -q -O- http://nginx-hpa >dev/null 2>&1; done'
```

The deployment will allow us to easily add replicas and generate more load :)

```bash
kubectl apply -f resources/load-generator-dep.yaml
```

Monitor in different windows:
- `kubectl get hpa -w`
- `kubectl get pod -w`

If the load doesn't increase, add replicas to the load generator:

```bash
kubectl scale deployment load-generator --replicas=4
```

<a name="cluster"></a>
## Cluster Autoscaling

Cluster autoscaling requires integration with an external service (cloud) to provision new nodes. In GKE, this can be enabled using the `Node auto-provisioning` option, defining limits, zones for node pools, etc.

When Kubernetes needs more resources than are available, it will request the cloud provider to add nodes. When nodes are no longer needed, they will be destroyed.

For this test, we can deploy the `nginx-affinity` example used in another demonstration (which had 200m requests) and scale it to 6 or 8 replicas.

```bash
k scale deployment nginx-affinity --replicas=12
```

If we check the pods that cannot be placed, we will see that the `cluster-autoscaler` component triggers a scale-up event:

```console
Events:
  Type     Reason            Age                From                Message
  ----     ------            ----               ----                -------
  Warning  FailedScheduling  37s (x2 over 38s)  default-scheduler   0/3 nodes are available: 3 Insufficient cpu.
  Normal   TriggeredScaleUp  35s                cluster-autoscaler  pod triggered scale-up: [{https://www.googleapis.com/compute/v1/projects/chrome-plateau-338520/zones/us-west1-a/instanceGroups/gke-keepcoding-default-pool-887bbdcb-grp 3->6 (max: 6)}]
```

## End
[Back](./README.md)
