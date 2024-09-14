<a name="pod"></a>
## Creating and Maintaining `Deployments` in Kubernetes

Table of contents:

- [Creating and basic information](#creation)
- [Updating deployments](#updates)
- [Scaling deployments](#scale)
- [Pausing rolling updates](#pause)

<a name="creation"></a>
### Creating and basic information about Deployments

We will start by defining the following deployment:

```yaml
apiVersion: apps/v1 # API version (it changes over time)
kind: Deployment  # TYPE: Deployment
metadata: # Deployment metadata
  name: nginx-deployment
#  namespace: test2
spec: # Deployment specification
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # Tells the controller to run 2 pods
  template:
    metadata: # Pod metadata
      labels:
        app: nginx
    spec: # Pod specification
      containers: # Declaration of pod containers
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

__IMPORTANT CONSIDERATIONS__:

* Selector: It must target the labels of the pods, and you must define them accurately. Ensure they are not too generic just in case!

* The `spec.template` block starts the pod definition, so `spec.template.spec` corresponds to the pod specification (which we already know).

Create the deployment:
```
kubectl apply -f resources/nginx-dep.yaml
```

Check the deployment status with `get` and `describe`. Observe and understand the fields:

```
$ kubectl get deployment nginx-deployment -o=wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES        SELECTOR
nginx-deployment   0/2     2            0           6s    nginx        nginx:1.7.9   app=nginx

$ kubectl describe deployment nginx-deployment
...
...
```

To check the deployment status in terms of configuration changes (rolling updates), we use the `rollout` command in kubectl.

```
$ kubectl rollout status deployment nginx-deployment
deployment "nginx-deployment" successfully rolled out
```

We can also display:

- ReplicaSet associated with the deployment:
```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5d59d67564   2         2         2       3m38s
```

- Pods with their labels:

```
kubectl get pods --show-labels
NAME                                READY   STATUS    RESTARTS      AGE     LABELS
nginx-deployment-5d59d67564-jvgbx   1/1     Running   0             4m5s    app=nginx,pod-template-hash=5d59d67564
nginx-deployment-5d59d67564-sfjvw   1/1     Running   0             4m5s    app=nginx,pod-template-hash=5d59d67564
```

<a name="updates"></a>
### Updating Deployments

- Suppose we want to update the image to `nginx:1.9.1`. We can:
  - Modify the yaml and reapply with `kubectl apply -f resources/nginx-dep.yaml`
  - Edit the deployment with `kubectl edit deployment nginx-deployment`
  - Use `kubectl --record set`:

  ```
  kubectl --record deployment.apps/nginx-deployment set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1
  ```

After applying the update, check the status of the changes being applied:

```
$ kubectl rollout status deployment.v1.apps/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

As we can see, the old pods are destroyed and replaced by new ones.

You can also verify the changes' status in a simpler way using:
  - `kubectl get deployment`
  - `kubectl get pod`

- Reverting changes:

Suppose we make a mistake.

Check deployment rollout history:
`kubectl rollout history deployment.v1.apps/nginx-deployment`

kubectl rollout history deployment.v1.apps/nginx-deployment --revision=2

kubectl rollout undo deployment.v1.apps/nginx-deployment

kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2

<a name="scale"></a>
### Scaling Deployments

Scaling (and downscaling) involves changing the number of replicas.

- Scale a deployment:

  - Basic:
  ```
  kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
  ```

  - If autoscaling is enabled in the cluster:
  ```
  kubectl autoscale deployment.v1.apps/nginx-deployment --min=10 --max=15 --cpu-percent=80
  ```

<a name="pause"></a>
### Pausing / stopping rolling updates

- Pausing rollouts: This tells the system to temporarily `not apply configuration changes`. This allows you to make multiple changes at once, for example:

1) Pause the deployment: `kubectl rollout pause deployment.v1.apps/nginx-deployment`
2) Make multiple changes. After each change, you will see that nothing is applied immediately:

```
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1
kubectl set resources deployment.v1.apps/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```

You will notice that `rollout history` does not show the change, and `rollout status` shows a message like:
```
Waiting for deployment "nginx-deployment" rollout to finish: 0 out of 10 new replicas have been updated...
```

3) Resume the deployment rollout: `kubectl rollout resume deployment.v1.apps/nginx-deployment`

Check that the changes are applied:
```
kubectl get deployment
kubectl get pod -w
kubectl rollout status deployment nginx-deployment
```

__Troubleshooting__

If the process gets stuck, always check:
- Status of the pods and the deployment.
- Describe the components that are not in the correct state.
- Pod logs if the pods are failing for some reason.


### End
[Back](./README.md)
