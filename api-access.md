# API Access from Pods

In this section:
- Introduction
- API access using `kubectl` installed in a pod.
- API access using `curl` from a pod (without `kubectl`).
- Example with [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- Creating a pod without API access.

## Introduction

In Kubernetes, by default:

- There is a service called `kubernetes` in the `default` namespace that provides API access to any pod (`kubernetes.default.svc.cluster.local` or simply `kubernetes.default.svc`).
- All pods run with a `serviceaccount` (`default` if not specified).
- All pods have their `serviceaccount` token available in the `/var/run/secrets/kubernetes.io/serviceaccount` directory.

Relevant documentation:
- [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Accessing Cluster from the API](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster)
- [Accessing Cluster API from within a Pod](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/)
- [API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/): A good reference for basic operations.

Important `Pod` spec parameters:
- `serviceAccountName`: defines the Service Account to use; if not defined, `default` is used.
- `automountServiceAccountToken`: specifies whether the pod should automatically mount the authentication token for API access. The default is `true` (for security, consider setting it to `false` if your pods don't need API access).

## API Access with Kubectl

Note: Example obtained from [this site](https://itnext.io/running-kubectl-commands-from-within-a-pod-b303e8176088)

- __Step 1__: We need an image that has `kubectl` installed. We can create it:

```bash
# Dockerfile
FROM debian:buster
RUN apt update && \
      apt install -y curl && \
      curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl && \
      chmod +x ./kubectl && \
      mv ./kubectl /usr/local/bin/kubectl
CMD kubectl get po
```

We build the image and push it to DockerHub:

```bash
docker build --platform linux/amd64 -t eedugon/internal-kubectl:latest .
docker push eedugon/internal-kubectl:latest
```

- __Step 2__: We test it in the cluster using one of the following methods:

Defining a pod in a `yaml` file:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: internal-kubectl
spec:
  containers:
    - name: internal-kubectl
      image: eedugon/internal-kubectl:latest
```

Or directly with `kubectl run`:

```bash
kubectl run kubectl-test --image eedugon/internal-kubectl:latest
```

- Did the access work?

## API Access with `curl` from a Pod (without `kubectl`)

We start a pod with an image that has `curl` installed (e.g., `busybox`):

```
kubectl run curl-test -it --image=radial/busyboxplus:curl --rm
```

We explore the system:
```bash
$ cd /var/run/secrets/kubernetes.io/serviceaccount
$ ls -l
total 0
lrwxrwxrwx    1 root     root          13 Oct  4 06:41 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     root          16 Oct  4 06:41 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root          12 Oct  4 06:41 token -> ..data/token
```

Remember that the API is accessible via the DNS name `kubernetes`.

```bash
ping kubernetes
curl kubernetes
curl https://kubernetes
curl --cacert ca.crt https://kubernetes:443
curl --cacert ca.crt --header "Authorization: Bearer $(cat token)" https://kubernetes:443
curl --cacert ca.crt --header "Authorization: Bearer $(cat token)" https://kubernetes:443/api

# Bingo, it works and we have access. Now let's see if we can get the pods...
curl --cacert ca.crt --header "Authorization: Bearer $(cat token)" https://kubernetes:443/api/v1/pods
```

In the official docs, there is a [more elaborate example](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/#without-using-a-proxy) :)

## Example with RBAC

So far:
- We've seen how to access the API from pods, both with and without `kubectl`.
- We've confirmed that by default, we don't have permissions for almost anything.

Let's create a ServiceAccount and grant permissions via `Roles` and/or `ClusterRoles`. We'll do everything in a namespace called `prueba-rbac` (you can also do this in the `default` namespace):

```bash
kubectl create namespace prueba-rbac
kubectl -n prueba-rbac create serviceaccount prueba-sa
kubectl create role role-prueba -n prueba-rbac --verb get,list,delete --resource pod
kubectl create rolebinding role-binding-prueba-sa -n prueba-rbac --serviceaccount prueba-rbac:prueba-sa --role role-prueba
```

Remember that the `--dry-run client -o=yaml` option is super useful for generating `yaml` files!

We could also have created this using the following `yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: prueba-sa
  namespace: prueba-rbac
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: role-prueba
  namespace: prueba-rbac
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: role-binding-prueba-sa
  namespace: prueba-rbac
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: role-prueba
subjects:
- kind: ServiceAccount
  name: prueba-sa
  namespace: prueba-rbac
```

- __Note__: The `role` we created as an example grants `get`, `list`, and `delete` permissions on pods in the `prueba-rbac` namespace only. If we wanted to grant permissions for pods in all namespaces, we would need to create a `ClusterRole` and `ClusterRoleBinding` instead of `Role` and `RoleBinding`.

Testing access:

Generate the yaml skeleton:

The `--serviceaccount` option no longer exists in `kubectl run`, so we need to generate the pod's yaml first:

```bash
kubectl run curl-test -n prueba-rbac --image=nginx --dry-run=client -o yaml > curl-test-serviceaccount-test.yaml
```

We edit it and adjust it like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: curl-test
  name: curl-test
  namespace: prueba-rbac
spec:
  serviceAccountName: prueba-sa # Add this line
  containers:
  - image: nginx
    name: curl-test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

We create it, open a shell via `kubectl exec -it curl-test bash`, and...

```bash
cd /var/run/secrets/kubernetes.io/serviceaccount

curl --cacert ca.crt --header "Authorization: Bearer $(cat token)" https://kubernetes.default.svc:443/api/v1/namespaces/prueba-rbac/pods
```

## Creating a Pod without API Access

If we want to block API access from the pod, we disable the automatic token mounting by setting `automountServiceAccountToken: false`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: curl-test-2
  name: curl-test-2
  namespace: prueba-rbac
spec:
  serviceAccountName: prueba-sa
  automountServiceAccountToken: false
  containers:
  - image: nginx
    name: curl-test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

We will confirm that the `/var/run/secrets/` directory does not exist.
This doesn't mean the service account doesn't have permissions, but rather that the pod cannot retrieve the connection data.

## Extras / Suggestions

- Use one of the Kubernetes API interaction libraries. More [here](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/#using-official-client-libraries).
