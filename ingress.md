<a name="ingress"></a>
## Ingress Demonstration and Examples

The `Ingress Controller` is a reverse proxy that we will make accessible from outside the cluster and configure using `Ingress` objects.

For our exercise, we will install the NGINX Ingress Controller (open-source version), available [here](https://kubernetes.github.io/ingress-nginx/deploy/). Each ingress controller has its own features and peculiarities.

In this demonstration:
* [Creating prerequisite resources](#ingress-prep)
* [Installing the Ingress Controller](#ingress-install)
* [Simple Ingress with default backend](#ingress-default)
* [Simple Ingress with virtual host and path distribution](#ingress-path)
* [Multiple virtual hosts with real DNS](#ingress-vs)
* [Securing traffic via SSL / TLS (HTTPS)](#ingress-ssl)
* [SSL passthrough](#ingress-pass)
* [Ingress for TCP/UDP services](#ingress-tcp)

<a name="ingress-prep"></a>
### Preparation

For the demo, we will use 3 backend applications:
- NGINX Deployment with 2 replicas + `ClusterIP` service listening on port `80` (we use the same NGINX as in the previous demonstrations).
- Simple `apple` application + `ClusterIP` service listening on port `5678`.
- Simple `banana` application + `ClusterIP` service listening on port `5678`.

The `apple/banana` applications are simple example pods, using a basic image that always returns the same response:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: banana-app
  namespace: default
  labels:
    app: banana
spec:
  containers:
    - name: banana-app
      image: hashicorp/http-echo
      args:
        - "-text=banana"
---
kind: Service
apiVersion: v1
metadata:
  name: banana-service
  namespace: default
spec:
  selector:
    app: banana
  ports:
    - port: 5678 # Default port for image
```

Deploy components:

```
# Deploy NGINX if not already deployed:
kubectl apply -f resources/nginx-dep.yaml

# Deploy NGINX Service of type ClusterIP (no need to expose it externally as this will be done via Ingress).
kubectl apply -f resources/svc-ClusterIP.yaml

# Deploy apple and banana applications
kubectl apply -f resources/apple_pod_and_service.yaml
kubectl apply -f resources/banana_pod_and_service.yaml
```

<a name="ingress-install"></a>
### Installing the Ingress Controller

This demonstration has been tested with version `1.3.1` of the controller, whose original manifest is available in this repository. You can obtain an updated version by following [these instructions](https://kubernetes.github.io/ingress-nginx/deploy/).

Currently (May 2023), version 1.7.1 of the controller is available, and there are significant changes in Kubernetes 1.26 regarding IngressClasses.

__NOTE!!__ If you are on GKE, you may need to grant your `gcloud` user cluster-admin permissions on the cluster (explained [here](https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke)). To do so:

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

To install it, run:

```
kubectl apply -f resources/ingress-controller-v1.3.1.yaml
```

The controller can also be installed and managed via `Helm`.

Once installed, we can observe the installed resources in the `ingress-nginx` namespace.

Pay special attention to the `LoadBalancer` service. This means that our ingress controller (reverse proxy) will be exposed externally. Take note of the external IP address for access and testing.

```
kubectl get service -n ingress-nginx
```

The ingress-controller is installed with `ingress class` = `nginx`, so all our ingresses must include the annotation `kubernetes.io/ingress.class: nginx` to ensure they are processed by our controller.

UPDATE: In older versions of Kubernetes, the ingress class annotation was mandatory. Now (1.26), you must use `IngressClassName`.

<a name="ingress-default"></a>
### Ingress with Default Backend

We will create an ingress (named `nginx-ingress-default-backend`) that simply redirects all traffic to the `nginx` service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-default-backend
  #annotations:
  #  kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: nginx
      port:
        number: 80
```

__Creating the Ingress__

```
kubectl apply -f resources/ingress-01-singleService.yaml
```

Once the ingress is created, we can inspect its details using `get` and `describe`. The `describe` command is especially useful as it will show events and any potential issues with our Ingress:

```
kubectl get ingress
kubectl describe ingress nginx-ingress-default-backend
```

__Testing the Ingress__

For this test, simply open a browser or use `curl` to connect to the external IP of the ingress. Traffic should be forwarded to the `nginx` service and then to the corresponding pods.

<a name="ingress-path"></a>
### Ingress with Virtual Host and Path Distribution

A more complex example:
- A virtual host named `foo.bar.com` with two path-based rules:
  - If the URL starts with /apple, the traffic will be sent to the `apple-service`.
  - If the URL starts with /banana, the traffic will be sent to the `banana-service`.
  - If the URL starts with /nginx, the traffic will be sent to the `nginx` service.

- In this case, an annotation `nginx.ingress.kubernetes.io/rewrite-target` is required in the ingress. This allows:
  - If the user accesses `http://foo.bar.com/nginx/index.html`, the reverse proxy will "strip" the prefix from the URL before forwarding it to the backend, resulting in `http://nginx/index.html` (__correct__), instead of `http://nginx/nginx/index.html` (__which would not exist__).
  - More details can be found in the official documentation [here](https://kubernetes.github.io/ingress-nginx/examples/rewrite/).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-simple-fanout-example
  annotations:
    kubernetes.io/ingress.class: nginx
    # nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /apple(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 5678
      - path: /banana(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: banana-service
            port:
              number: 5678
      - path: /nginx(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

__Creating the Ingress__

```
kubectl apply -f resources/ingress-02-simpleFanout.yaml
```

Once the ingress is created, check the details using `get` and `describe`. The `describe` command is important as it will display events and indicate if there are any issues with the Ingress:

```
kubectl get ingress
kubectl describe ingress nginx-simple-fanout-example
```

__Testing Ingress__

This example is based on a fictional URL (`foo.bar.com`), so we will need to modify the `/etc/hosts` file on our machine to point to the correct IP.

Take note of the external IP of the Ingress object (it should be the same as the controller's IP) and configure your PC to access `http://foo.bar.com` with that IP by adding the following entry to the `/etc/hosts` file:

```
34.105.62.166 foo.bar.com
```

Test the following accesses:
- http://foo.bar.com/apple
- http://foo.bar.com/banana
- http://foo.bar.com/nginx
- http://foo.bar.com/

<a name="ingress-vs"></a>
### Ingress with Multiple Virtual Hosts and Real DNS

In the following example, we will set up multiple virtual hosts with real DNS using `nip.io` for automatic resolution (we will need to modify the proposed YAML file).

We will have 3 different virtual hosts, each for a different application. Two of them will have a specific FQDN, and the third one (banana) will be accessible with any other FQDN or when accessed directly by IP.

Make sure to adjust the FQDNs (`host`) in the proposed YAML to look like:
- `nginx.IP-OF-THE-LB.nip.io`
- `apple.IP-OF-THE-LB.nip.io`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-name-based-vs-and-default
  annotations:
    kubernetes.io/ingress.class: nginx
    #nginx.ingress.kubernetes.io/rewrite-target: /  # not needed in this example.
spec:
  rules:
  - host: nginx.34-105-62-166.nip.io
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx
            port:
              number: 80
  - host: apple.34-105-62-166.nip.io
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: apple-service
            port:
              number: 5678
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: banana-service
            port:
              number: 5678
```

__Creating the Ingress__

```
kubectl apply -f resources/ingress-04-nameBasedVS-2.yaml
```

Once the ingress is created, check the details using `get` and `describe`. The `describe` command is important because it will show events and indicate any issues with our Ingress:

```
kubectl get ingress
kubectl describe ingress nginx-name-based-vs-and-default
```

__Testing the Ingress__

Open your browser and check if the result matches expectations (adapt the IPs to your environment):
- http://nginx.34-105-62-166.nip.io (should go to nginx)
- http://apple.34-105-62-166.nip.io (should go to apple)
- http://otracosa.34-105-62-166.nip.io (should go to banana)
- http://34.105.62.166 (should go to banana)

__Note__: In this example, the last rule (banana) is incompatible with the Ingress created with the default backend, so make sure it is deleted.

<a name="ingress-ssl"></a>
### Securing Application Access via SSL / TLS (HTTPS)

When our applications do not support SSL / TLS (or even if they do but we prefer to handle it this way by design), we can configure our reverse proxy (ingress controller) to handle SSL termination.

The traffic flow will be:
```
Client === (HTTPS / SSL) ===> Ingress Controller === (Plain HTTP) ===> Service ===> Pods
```

To do this, we will need to generate SSL certificates, which can be done in several ways:
- Manually, if we already have the certificate, the key, and the CA(s). The process is simple and is described in the official ingress controller documentation [here](https://kubernetes.github.io/ingress-nginx/user-guide/tls/) and [here](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls).
- Using a tool like `cert-manager`, which helps with the automatic generation of certificates from [Letsencrypt](https://letsencrypt.org).

### Installing cert-manager

Official documentation [here](https://cert-manager.io/docs/installation/)

- __Important__: If you're on GKE, you need to grant your `gcloud` user cluster-admin permissions (explained [here](https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke)). To do this:

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

Certificate generation: https://cert-manager.io/docs/usage/

Ingress integration with Cert-manager: https://cert-manager.io/docs/usage/ingress/

Installation: `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.yaml`

https://cert-manager.io/docs/installation/

- Install `cert-manager` (default values):

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.yaml
```

- Create `ClusterIssuers` integrated with Letsencrypt (staging and prod):

```yaml
# Prod issuer (more limitations compared to staging. It is recommended to start with staging)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: edu.gherran@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    #server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-prod-issuer-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
---
# Staging issuer (creates certificates)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: edu.gherran@gmail.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-staging-issuer-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

To create these resources:
```
kubectl apply -f resources/cert-issuers/
```

- Now we can generate our Ingress. Adapt the FQDNs / IP addresses to your environment.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ssl
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-staging
    #cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - nginx.35-197-125-202.nip.io
    secretName: nginx-ingress-ssl-cert
  rules:
  - host: nginx.35-197-125-202.nip.io
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx
            port:
              number: 80
```

__NOTE!__: Make sure the `nginx` service and its pods are created and available :)

Wait for the certificate to be generated (`kubectl get certificates`) and ready, then test access to the ingress using HTTPS, just like in the previous examples.

Letsencrypt's `staging` certificates are not trusted by default in browsers, so if we want the certificate to appear trusted, we will need to download the CAs and add them to our operating system's security configuration. More about Letsencrypt's staging environment [here](https://letsencrypt.org/docs/staging-environment/).

Extras:
- To disable basic HTTP access for the previous ingress, you would need to modify the controller itself and change the service to not expose port 80 (`kubectl get services -n ingress-nginx`).

- GKE's ingress controller allows annotations in the `Ingress` object to disable HTTP, but the NGINX ingress controller does not.

- As you'll notice, cert-manager has added many CRDs (`Custom Resource Definitions`) to the cluster. You can check them out and analyze their `describe` output using the following commands:

```
kubectl get clusterissuers
kubectl get certificates
kubectl get certificaterequests
```

<a name="ingress-pass"></a>
### SSL Passthrough

If your application already listens for HTTPS (SSL/TLS) and you want the reverse proxy to pass through SSL connections without terminating them:

- You need to configure the controller to allow SSL passthrough by adding the `--enable-ssl-passthrough` argument to the controller's main deployment (check the installation manifest).

- Add the annotation `nginx.ingress.kubernetes.io/ssl-passthrough: "true"` in the Ingress object so that the controller sends the TLS connections directly to the backend instead of processing the encrypted traffic.

More info [here](https://kubernetes.github.io/ingress-nginx/user-guide/tls/#ssl-passthrough) and [here](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#ssl-passthrough).

Example of Ingress with SSL passthrough (__Note__: It uses the old API version (`v1beta1`), which should be updated to the new syntax).

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  name: es1ns1-kibana-ingress
  namespace: ns1
spec:
  tls:
  - hosts:
    - es1ns1.34.122.64.213.nip.io
  rules:
  - host: es1ns1.34.122.64.213.nip.io
    http:
      paths:
      - path: /
        backend:
          serviceName: es1ns1-kb-http
          servicePort: 5601
```

<a name="ingress-tcp"></a>
### TCP / UDP Services

- By default, Ingress only accepts HTTP/HTTPS traffic.
- Exposing applications with other protocols is more complex.
- The NGINX ingress controller allows this, though implementation can be somewhat complicated:
  - More details [here](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)

Steps:
  - Create a ConfigMap with the information about ports and services to expose.
  - Modify the controller's configuration and add the execution parameters `--tcp-services-configmap` and `--udp-services-configmap` as needed.

### End
[Back](./README.md)


