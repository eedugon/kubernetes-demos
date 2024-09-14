<a name="pod"></a>
# Operators in Kubernetes

We will demonstrate the use of Operators in Kubernetes using the Elastic ECK (Elastic Cloud on Kubernetes) operator, deploying a complete Observability solution for Kubernetes.

Table of contents:

- [Installing the ECK Operator (Elastic Cloud on Kubernetes)](#install)
- [Deploying Elasticsearch and Kibana cluster](#deploy)
- [Deploying Metricbeat as a DaemonSet for metrics](#metrics)
- [Deploying Filebeat as a DaemonSet for logs](#logs)
- [Deploying Heartbeat as a Deployment for uptime checks](#uptime)
- [Installing a customized Dashboard](#dashboard)

(Note: Updated as of 2023-09: ECK v2.9.0, Elastic Stack v 8.9.2)

Cluster used:
- 3 GKE nodes with `E2 standard` (2vCPUs / 8GB RAM)

<a name="install"></a>
## Installing ECK (Elastic Cloud on Kubernetes)

ECK is Elastic’s Operator for managing Elastic Stack components in Kubernetes.

- To install it on GKE, first ensure you have cluster-admin permissions:

```
kubectl create clusterrolebinding \
cluster-admin-binding \
--clusterrole=cluster-admin \
--user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
```

- The installation is straightforward, following the [official documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-installing-eck.html). You can install it via `helm chart` or directly using the provided `yaml` files. In this demonstration, we’ll use the YAML method.

```
kubectl create -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml

# To check the operator logs...
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

ECK offers two licensing modes:
- Basic (default): Free and allows orchestration of all components under the Basic license.
- Enterprise (requires Enterprise license): Manages components with all the features of Platinum/Enterprise licenses.

To activate an Enterprise trial license in ECK:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: eck-trial-license
  namespace: elastic-system
  labels:
    license.k8s.elastic.co/type: enterprise_trial
  annotations:
    elastic.co/eula: accepted 
EOF
```

<a name="deploy"></a>
## Deploying Elasticsearch and Kibana Cluster

References:
- [Quickstart](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html)
- Detailed [Elasticsearch configurations](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-elasticsearch-specification.html)
- Detailed [Kibana configurations](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-kibana.html)

Deployment details:
- Service type for Elasticsearch: `ClusterIP`
- Elasticsearch resources: "400m" CPU / 2GB RAM / 10GB persistent storage
- Number of Elasticsearch nodes: 3
- Service type for Kibana: `LoadBalancer`

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: logging-and-metrics
spec:
  http:
    service:
      spec:
        type: ClusterIP
  version: 8.10.2
  nodeSets:
  - name: default
    count: 3
    config:
      node.roles: [ master, data, ingest, transform, remote_cluster_client, ml ]
    podTemplate:
      spec:
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
            runAsUser: 0
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: -Xms1g -Xmx1g
          resources:
            requests:
              memory: 2Gi
              cpu: 400m
            limits:
              memory: 2Gi
              cpu: 2
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 10Gi
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: logging-and-metrics
spec:
  elasticsearchRef:
    name: logging-and-metrics
  version: 8.10.2
  count: 1
  http:
    service:
      spec:
        type: LoadBalancer
```

Deploy the stack:
```
kubectl apply -f resources/monitoring/stack-elasticsearch-kibana.yaml
```

Check the status of the deployed resources:
```
kubectl get elasticsearch
kubectl get kibana
```

The `elastic` user’s password (admin) is stored in a secret. You can retrieve it with:
```
kubectl get secret logging-and-metrics-es-elastic-user -o=jsonpath={.data.elastic} | base64 --decode
```

Wait for the LoadBalancer to become active, and Kibana to be up and running, and then access it.

<a name="metrics"></a>
## Deploying Metricbeat as a DaemonSet for Metrics

We will install Metricbeat following the [Running Metricbeat on Kubernetes](https://www.elastic.co/guide/en/beats/metricbeat/current/running-on-kubernetes.html) guide, but using ECK instead of directly deploying a DaemonSet.

Metricbeat’s Kubernetes module, like Prometheus, requires `kube-state-metrics` as a metrics provider for cluster state.

Installing `kube-state-metrics` is simple. The [installation guide](https://github.com/kubernetes/kube-state-metrics#kubernetes-deployment) is available here.

__Note__: If you have installed the `kube-prometheus` stack, you might already have `kube-state-metrics` installed.

- Install `kube-state-metrics`:

```
kubectl apply -f resources/monitoring/kube-state-metrics-v2.10.0/standard/
```

- Install Metricbeat for metrics, reviewing the manifests:

```
kubectl apply -f resources/monitoring/metrics
```

<a name="logs"></a>
## Deploying Filebeat as a DaemonSet for Logs

We will install Filebeat following the [Running Filebeat on Kubernetes](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html) guide, using ECK.

- Install Filebeat for logs, reviewing the manifests:

```
kubectl apply -f resources/monitoring/logs
```

<a name="uptime"></a>
## Deploying Heartbeat as a Deployment for Uptime Checks

- Install Heartbeat for uptime monitoring, reviewing the manifests:

```
kubectl apply -f resources/monitoring/uptime
```

<a name="dashboard"></a>
## Installing a Customized Dashboard

Open Kibana, go to `Stack Management --> Saved Objects`, click on `Import`, and upload the file `resources/monitoring/eedugon-dashboards.ndjson` from the repository. Select the `overwrite` option in case of conflicts.

Next, go to `dashboards`, search for the `eedugon-k8s-data-checks` dashboard, and open it. This dashboard will monitor log and metric submissions from our observability agents.

# Operator Conclusions

- Deploying known applications via the operator is much simpler (though you still need to understand and document the applications). If you know what you want, using the operator makes the process easier.
- The operator automatically connects Beats with the Elasticsearch cluster securely using SSL/TLS.
- Beats only need to define `elasticsearchRef` with the Elasticsearch cluster identifier, and the operator handles providing Filebeat with the correct connection parameters (host, port, user, token, etc.).
- The `Beat`, `Elasticsearch`, and `Kibana` objects are not default Kubernetes resources; they are introduced by the operator.

Example of filebeat metadata control:
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-beat-configuration-examples.html#k8s_filebeat_with_autodiscover_for_metadata

### End
[Back](./README.md)
