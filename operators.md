<a name="pod"></a>
# Operadores en Kubernetes

Demostraremos el uso de Operadores en Kubernetes utilizando el operador de Elastic ECK (Elastic Cloud on Kubernetes) y desplegando una solución completa de Observability para Kubernetes.

Indice de contenidos:

- [Instalación de operador ECK (Elastic Cloud on Kubernetes)](#install)
- [Despliegue de cluster de Elasticsearch y Kibana](#deploy)
- [Despliegue de Metricbeat como DaemonSet para métricas](#metrics)
- [Despliegue de Filebeat como DaemonSet para logs](#logs)
- [Despliegue de Heartbeat como Deployment para chequeos "uptime"](#uptime)
- [Instalación de Dashboard customizado](#dashboard)

(Nota: actualización 2023-09: ECK v2.9.0, Elastic Stack v 8.9.2)

Cluster utilizado:
- 3 GKE con 3 nodos `E2 standard` (2vCPUs / 8G RAM)

<a name="install"></a>
## Instalación de ECK (Elastic Cloud on Kubernetes)

ECK es el Operador de Elastic para la gestión de los componentes del Elastic Stack en Kubernetes.

- Para instalarlo en GKE primero tendremos que asegurarnos de tener un rol de cluster-admin:

```
kubectl create clusterrolebinding \
cluster-admin-binding \
--clusterrole=cluster-admin \
--user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
```

- La instalación es muy sencilla, seguiremos la [documentación oficial](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-installing-eck.html). Se puede instalar como `helm chart` o directamente con los ficheros `yaml`. Para esta demostración usaremos los YAML.

```
kubectl create -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml

# to check logs of the operator...
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

ECK tiene dos modalidades de licencia:
- Basic (por defecto): Es gratuita y permite la orquestación de todos los componentes con licencia Basic.
- Enterprise (requiere licencia Enterprise): Orquesta los componentes con todas las características de las licencias Platinum/Enterprise.

Para activar una trial Enterprise en ECK:

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
## Despliegue de Cluster de Elasticsearch y Kibana

Referencias:
- [Quickstart](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html)
- Configuraciones detalladas de [Elasticsearch](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-elasticsearch-specification.html)
- Configuraciones detalladas de [Kibana](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-kibana.html)

Detalles despliegue:
- Tipo de servicio para Elasticsearch: `ClusterIP`
- Recursos Elasticsearch: "400m" cpu / 2G RAM / 10G disco persistente
- Número de nodos de Elasticsearch: 3
- Tipo de servicio para Kibana: `LoadBalancer`


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
        accessModes:
        - ReadWriteOnce
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

Lanzamos el despliegue:
```
kubectl apply -f resources/monitoring/stack-elasticsearch-kibana.yaml
```

Podemos comprobar el estado de los recursos desplegados con:
```
kubectl get elasticsearch
kubectl get kibana
```

La `password` del usuario `elastic` (el administrador) estará guardada en un secret. Para obtenerla podemos ejecutar:
```
kubectl get secret logging-and-metrics-es-elastic-user -o=jsonpath={.data.elastic} | base64 --decode
```

Esperamos a que el LoadBalancer esté activo y Kibana levantado y accedemos.


Referencias despligue:
- [Quickstart Elasticsearch](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html)
- [Quickstart Kibana](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-kibana.html)
- [Control de recursos y valores por defecto](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-managing-compute-resources.html#k8s-default-behavior) el opera

<a name="metrics"></a>
## Despliegue de Metricbeat como DaemonSet para métricas

Instalaremos Metricbeat siguiendo la referencia de la página [Running Metricbeat on Kubernetes](https://www.elastic.co/guide/en/beats/metricbeat/current/running-on-kubernetes.html) pero utilizando ECK en lugar de un despliegue de DaemonSet directo.

Ejemplos de despliegue de Beats con ECK [aquí](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-beat-configuration-examples.html)

El módulo de Kubernetes de Metricbeat, al igual que prometheus, requiere de la aplicación `kube-state-metrics` como proveedor de métricas del estado del cluster.

La [instalación de kube-state-metrics](https://github.com/kubernetes/kube-state-metrics#kubernetes-deployment) es muy sencilla

__OJO__! Si instalas el stack de `kube-prometheus` probablemente ya tengas `kube-state-metrics` instalado. Considéralo.

- Instalamos `kube-state-metrics`:

```
kubectl apply -f resources/monitoring/kube-state-metrics-v2.10.0/standard/
```

- Instalamos Metricbeat para métricas, __analizando los manifiestos__:

```
kubectl apply -f resources/monitoring/metrics
```

<a name="logs"></a>
## Despliegue de Filebeat como DaemonSet para logs

Instalaremos Metricbeat siguiendo la referencia de la página [Running Filebeat on Kubernetes](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html) pero utilizando ECK en lugar de un despliegue de DaemonSet directo.

- Instalamos Filebeat para logs, __analizando los manifiestos__:

```
kubectl apply -f resources/monitoring/logs
```

<a name="uptime"></a>
## Despliegue de Heartbeat como Deployment para chequeos "uptime"

- Instalamos Heartbeat para `uptime`, __analizando los manifiestos__:

```
kubectl apply -f resources/monitoring/uptime
```

## Despliegue de deployment con modulos específicos para logs y métricas activados

Desplegaremos un apache y un redis activando los módulos correspondientes de Filebeat y Metricbeat vía `annotations` (hints based autodiscover).

Revisa y lanza los siguientes YAMLs:

```
kubectl apply -f resources/monitoring/apache-monitored.yaml
kubectl apply -f resources/monitoring/redis-monitored.yaml
```

Tras lanzarlos metricbeat y filebeat comenzarán el procesado de los módulos.

<a name="dashboard"></a>
## Instalación de Dashboard customizado

Abrimos Kibana, entramos en `Stack Management-->Saved Objects`, hacemos click en `Import` e importamos el fichero `resources/monitoring/eedugon-dashboards.ndjson` disponible en el repositorio. Seleccionamos la opción de `sobrescribir` en caso de conflictos.

Posteriormente vamos a `dashboards`, buscamos el dashboard `eedugon-k8s-data-checks` y lo abrimos. Ese dashboard analiza el envío de logs y métricas en general (una monitorización de nuestros agentes de observability).

# Conclusiones Operadores

- El despligue de aplicaciones conocidas por el operador es mucho más sencillo (aunque debemos documentarnos y conocer las aplicaciones igualmente). Siempre que sepamos qué queremos, hacerlo a través del operador será más sencillo.

- El operador se ha encargado de conectar los Beats con el cluster de Elasticsearch de forma segura y con SSL / TLS.

- Los beats solamente definen `elasticsearchRef` con el identificador del cluster de Elasticsearch, y es el operador el que se encarga de que Filebear reciba los parámetros de conexión correctos (host, port, usuario o token, etc.)

- Los objetos `Beat`, `Elasticsearch` y `Kibana` no existen por defecto en Kubernetes, han sido declarados por el Operador.

Ejemplo filebeat control metadatos:
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-beat-configuration-examples.html#k8s_filebeat_with_autodiscover_for_metadata

### Fin
[Volver](./README.md)
