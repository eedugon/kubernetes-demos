<a name="hello"></a>

# Kubernetes Hello World

In this section:
- Minikube hello-world
- GKE hello-world

<a name="install"></a>

Pre-requisites:
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/)
- For the `GKE hello world` create a GKE cluster
- For the `Minikube hello world` install Minikube following the [official documentation](https://minikube.sigs.k8s.io/docs/start).

## Minikube Install and Start

Start your Minikube cluster if it's not started already:

```bash
minikube start
```

Verify that kubectl has been automatically configured to point to the new cluster:

```bash
kubectl get nodes
kubectl config get-contexts # check the current context
kctx # with the plugin and alias
```

## Create a deployment with nginx image

```bash
kubectl create deployment hello-world --image=nginx --replicas=2
```

## Expose the deployment to the outside world

```bash
kubectl expose deployment hello-world --type=NodePort --port=80
```

This (`NodePort`) exposes the created deployment in all Kubernetes workers on a very high random port (the port `80` of the previous command describes the internal port), like `30768`.

## Obtain the URL

In a real cluster we would just need to access any worker IP address on the selected port, but as our Kubernetes worker (minikube) is running as a docker, we cannot access it directly from our laptop. In order to create a tunnel to the service:


```bash
minikube service hello-world
```

The previous returns something like:

```bash
|-----------|-------------|-------------|---------------------------|
| NAMESPACE |    NAME     | TARGET PORT |            URL            |
|-----------|-------------|-------------|---------------------------|
| default   | hello-world |          80 | http://192.168.49.2:30768 | # That's the IP address of the docker, which would be theoretically our entry point.
|-----------|-------------|-------------|---------------------------|
🏃  Starting tunnel for service hello-world.
|-----------|-------------|-------------|------------------------|
| NAMESPACE |    NAME     | TARGET PORT |          URL           |
|-----------|-------------|-------------|------------------------|
| default   | hello-world |             | http://127.0.0.1:52821 |
|-----------|-------------|-------------|------------------------|
🎉  Opening service default/hello-world in default browser...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

__NOTES:__
- Accessing services from Minikube is not straight forward as it is in real Kubernetes clusters, it requires an extra tunnel to be created by Minikube (the same can be appled to `LoadBalancer` services).

# GKE Hello-world

## Let's do the same towards a real GKE cluster

Create the cluster in GKE and follow these steps adapting your commands:

- Gain access to the GKE cluster by clicking on `Connect` and copying the `gcloud` command. This will configure and activate a kubectl context to access the cluster:

```bash
gcloud container clusters get-credentials YOUR_CLUSTER --zone europe-west1-b --project elastic-support-k8s-dev
```

- Check that you have access to the cluster (same as before):

```bash
kubectl get nodes
kubectl config get-contexts # check the current context
kctx # with the plugin and alias
```

- Create and expose the deployment (this time as a `LoadBalancer`):

```bash
kubectl create deployment hello-world --image=nginx --replicas=2
kubectl expose deployment hello-world --type=LoadBalancer --port=80
```

- Check the generated service and wait until there's an `EXTERNAL IP` available.

```bash
kubectl get service hello-world

NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-world   LoadBalancer   34.118.235.16   <pending>     80:32217/TCP   3s
```

- As soon as the IP is ready, you can access it from your browser `http://IP_ADDRESS`.

- Take a look at the generated cloud resources:
  - Load Balancer for our service
  - Virtual Machines for the GKE cluster


As a final tip, let's run the deployment creation command with these extra flags:

```bash
kubectl create deployment hello-world --image=nginx --replicas=2 --dry-run=client -o=yaml
```

### End
[Back](./README.md)

