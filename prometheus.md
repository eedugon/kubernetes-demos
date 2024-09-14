<a name="prometheus"></a>
# Prometheus and Grafana

Cluster used:
- 3 GKE nodes with `E2 standard` (2 vCPUs / 8 GB RAM). Ensure you have `cluster-admin` permissions.

```
kubectl create clusterrolebinding \
cluster-admin-binding \
--clusterrole=cluster-admin \
--user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
```

In this demonstration, we will install the Prometheus and Grafana Stack using [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus).

`kube-prometheus` is a project from the Prometheus Operator that installs a full stack with the following components:
- Prometheus Operator
- Prometheus in high availability
- Alert Manager in high availability
- Node exporter on every node
- Kubernetes metrics adapter
- Kube-state-metrics
- Grafana

Installation is straightforward, and we can do it via manifests following [these instructions](https://github.com/prometheus-operator/kube-prometheus/blob/main/README.md#quickstart) or via Helm charts using the [official chart](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack).

In this example, we will use the Helm chart:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

After installation, we can connect to all the components, but we will first access `Grafana`.
Since there's no SSL, we will use port-forwarding from the PC to access Grafana:

```
kubectl port-forward svc/my-kube-prometheus-stack-grafana 8888:80
```

To log in to Grafana, use: `admin` / `prom-operator`.

To access Prometheus:

```
k port-forward svc/my-kube-prometheus-stack-prometheus 9090:9090
```

And visit http://localhost:9090

You can search for metrics like container memory usage and quickly generate a graph.

### End
[Back](./README.md)

