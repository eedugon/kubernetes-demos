<a name="pod"></a>
# Liveness & Readiness Probes

<a name="liveness"></a>
## Liveness and Readiness Probes

The following example simulates the following scenario:
- The pod starts up and takes 20 seconds to be ready.
- After 40 seconds from startup, the liveness check will fail, and the pod will be restarted.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    command: ['sh', '-c']
    args:
      - |
        touch /tmp/healthy; sleep 20
        touch /tmp/ready; sleep 20
        rm -rf /tmp/healthy
        sleep 300
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 1
      successThreshold: 1
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/ready
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 1
      successThreshold: 1
```

To run the example:

```bash
kubectl apply -f resources/pod-probes.yaml
```

After launching it, open two different terminal windows. In one, monitor the pod's status with:
```bash
kubectl get pod liveness-exec -w
```

In the other window, you can check the events with:

```bash
kubectl describe pod liveness-exec
```

### End
[Back](./README.md)

