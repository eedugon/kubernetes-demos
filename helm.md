<a name="inicio"></a>
# Helm Charts

Table of contents:

- [Creating Charts: Initial Steps](#create)
- [Simple Example](#simple)
- [Example with Dependencies](#dependency)
- [Example with Multiple Components](#multiple)
- [Exploring More Complex Options](#adv)

<a name="create"></a>
## Creating Charts: Initial Steps

For this demonstration, we will convert part of the content from previous demos into a Helm chart (an NGINX, its service, and an Ingress object).

The files we'll be working with are available in the directory `resources/helm/original`, and include:
- `nginx-deployment.yaml`
- `service.yaml`
- `configmap-web.yaml`
- `ingress.yaml`

The goal is to `package` our set of manifests using a Helm chart, aiming to make all or most of the parameters flexible and `configurable`.

### Creating the Chart Skeleton

For this step, the name you choose is important since it will be the name of your package, and changing it later can be tricky without modifying multiple references.

```
helm create demo-nginx
```

Our package will be called `demo-nginx`, and by default, we will have a skeleton with several files.

It is recommended to review the default setup since it follows the `best practices` and is the recommended way to create our packages.

### Default Objects and Helpers

In our templates, everything we enclose between `{{` and `}}` will be processed by the Helm engine. Within these sections, we can use functions, pipelines, reference objects, etc.

The __main objects__ we can use include:

- `{{ .Release.Name }}` --> Name of the release passed during installation.
- `{{ .Chart.Name }}` --> Name of the chart (in our case, `demo-nginx`).

Helpers are useful because they process/validate/truncate data to ensure it is compatible.

The __default helpers__ we can use are:

- `fullname`: `{{ include "demo-nginx.fullname" . }}` --> the full name will be the release name + the chart.
- `labels` and `selectorLabels`: `{{- include "keepcoding-nginx.selectorLabels" . | nindent 2 }}` --> Very useful as it will create unique labels by default, so we don't have to worry about them.

`Fullname` is very useful because it helps create unique resource names.

__Learn to use `selectorLabels` from the provided examples in the skeleton__.

### Using `values.yaml`

Whenever we want something to be `configurable`, we will pass it to `values.yaml` and reference it as an object.

For example, if in our Kubernetes manifest we have:

```yaml
containers:
  - name: container1
  image: "nginx:1.7.9"
```

And we want to make the `image` configurable, we can change the file to:

```yaml
containers:
  - name: container1
  image: {{ .Values.image }}
```

And in `values.yaml` add:
```
image: "nginx:1.7.9"
```

__Why is this useful if we've kept the same value?__

What we've placed in `values.yaml` is the default value in case the user doesn't provide an `override`. However, it will always be possible to override any value during installation:

```
helm install -f my-values.yaml test demo-nginx/

# content of my-values.yaml
image: nginx:1.9.1
```

In conclusion, everything we use from `.Values` is configurable by the end users.

### Creating Templates

With this, we can start converting our original manifests into templates.

<a name="simple"></a>
## Simple Example:

As a first test, we will completely clear `values.yaml` and all the files in the skeleton, and we will recreate everything from scratch, bringing in our original `yaml` files and converting them into templates.

We clear the following from the skeleton:
- `values.yaml`
- All files in the `templates` directory except for `_helpers.tpl`, including those in the `templates/tests` directory.

__ConfigMap__

Our original ConfigMap was:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: static-page
data:
  index.html: |
    HELLO! Welcome to the test website
    Thanks for visiting
```

We need to choose a name that is dynamic, and we can make the content of `index.html` configurable. To do this, we convert it to:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cm
data:
  index.html: |
{{ .Values.content | indent 4 }}
```

__NOTE__!
* We will need to add `content` to `values.yaml` and give it an initial value.
* Take note of the name chosen for the `ConfigMap` (`{{ .Release.Name }}-cm`), as some other resources will need to reference it.

__Deployment__

Our original deployment was:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx # Label for the DEPLOYMENT, not the POD.
spec:
  selector:
    matchLabels:
      app: nginx # Must target the PODs.
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx # Label for the PODs.
    spec:
      volumes:
        - name: www-volume
          configMap:
            name: static-page
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: apps
                operator: In
                values:
                - frontend
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 100m
            memory: 100M
        volumeMounts:
        - name: www-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
```

This manifest needs a bit more work:
- Deployment name --> make it dynamic and __make a good decision__ (e.g. .Chart.Name, .Release.Name, fullname?).
- `Selector` and `pod labels` --> __IMPORTANT__, remember they need to match, and the future service will also need to use them. The recommendation here is to use the `helper` mentioned earlier.
- Deployment labels --> It's also recommended to use the `helper` for labels.
- `replicas` --> this should be __configurable__.
- `ConfigMap` volume --> It will need to point to the previously created ConfigMap.
- `affinity`: Do we want our chart to have affinity by default? __Let's think about that__.
- `resources`: Should it have default resources? Whatever the answer, __they should be configurable in any case__.
- `image`: This should be __configurable__.

A simple solution could be:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "demo-nginx.labels" . | nindent 4 }}    
spec:
  selector:
    matchLabels:
      {{- include "demo-nginx.selectorLabels" . | nindent 6 }}
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      labels:
        {{- include "demo-nginx.selectorLabels" . | nindent 8 }}
    spec:
      volumes:
        - name: www-volume
          configMap:
            name: {{ .Release.Name }}-cm
      containers:
        - name: nginx
          image: "{{ .Values.image }}"
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: {{ .Values.cpuRequest }}
            limits:
              cpu: {{ .Values.cpuLimit }}
          volumeMounts:
            - name: www-volume
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
```

In the above solution:
- We omitted affinity.
- For the deployment name, we used the `release name` (it would be better to use the `fullname` helper).
- We used `helpers` for labels and selectorLabels.
- We referenced the `ConfigMap` with the same name used before.
- The image will require the following in `values.yaml`:
  - `replicaCount`
  - `image`
  - `cpuRequest`
  - `cpuLimit`

__Service__

The original service was:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http
```

Elements to `templatize`:
- Service name --> it should be something like `fullname`.
- Service type, __configurable__ (we'll let the user choose between `ClusterIP`, `NodePort`, or `LoadBalancer`).
- Service listening port, __configurable__ (not the targetPort, as we have not made the listening port of our NGINX configurable).
- `selector`: __Very important that it matches the pod labels__.

A possible solution would be something like:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "demo-nginx.fullname" . }}
  labels:
    {{- include "demo-nginx.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    {{- include "demo-nginx.selectorLabels" . | nindent 4 }}
```

For values.yaml, we will need:
- `service.type`
- `service.port`

__values.yaml__

The minimum we will need in `values.yaml` for the above to generate valid manifests will be:

```yaml
# For the configmap
content: |+
  <h1>Test website. DEFAULT VALUE</h1>
  <p>Thanks for visiting</p>

# For the deployment
replicaCount: 1
image: "nginx:1.7.9"
cpuRequest: "100m"
cpuLimit: "200m"

# For the service
service:
  type: ClusterIP
  port: 80
```

__NOTES.txt__

This file will be processed by the Helm engine after installation and displayed. For this first example, we will include the following as a demonstration, a proof of concept, and to explore the content of variables:

```
THANK YOU FOR INSTALLING {{ .Chart.Name }}
THANK YOU FOR INSTALLING {{ include "demo-nginx.name" . }}

Release name used: {{ .Release.Name }}
Helper demonstration:

name: {{ include "demo-nginx.name" . }}
fullname: {{ include "demo-nginx.fullname" . }}
chart: {{ include "demo-nginx.chart" . }}
labels (new line and indented):
{{- include "demo-nginx.labels" . | nindent 2 }}
selectorLabels (indented):
{{- include "demo-nginx.selectorLabels" . | nindent 2 }}

For more details, you can run:
  $ helm status {{ .Release.Name }}
  $ helm get all {{ .Release.Name }}
  $ kubectl get pod
```

### Compiling and Testing the Template

The first thing we do is ask Helm to process it.

```
helm template --debug demo-nginx/

# Other options
helm template -f my-values.yaml --name-template my-install demo-nginx/
```

We analyze the output, redirect it to a file, and open it with a text editor.
We check that everything looks good.

The next test consists of a `dry-run` installation, without actually installing anything:

```
helm install test --dry-run --debug demo-nginx

# Other options
helm install test --dry-run -f my-values.yaml --namespace test demo-nginx/
helm install test --dry-run -f my-values.yaml --namespace test --create-namespace demo-nginx/
```

If everything looks good, we proceed with the installation:

```
helm install test demo-nginx
```

To test access without exposing it externally, we can do:

```
kubectl port-forward pod/POD_NAME 8888:80
```

And access it from our browser at [https://localhost:8888](https://localhost:8888).

__Installation Comments__

- Pod names ---> They are not very descriptive as they only have the release name. It would be better to use `Release.Name` + `Chart.Name`, or the `fullname` helper.

## Installing Packages with Our Configuration

Now that we have a very basic package, we will customize parameters.

To do this, we create a `my-values.yaml` file in a directory outside the chart and include the specific configuration we want, for example:

```
content: |+
  <h1>HELLO! CUSTOMIZED WEB THROUGH VALUES</h1>
  <h2>Thanks for visiting</h2>
```

One of the advantages of Helm packages is that __we can install the same package multiple times__ (even in the same namespace), and the resources should not conflict with the name (except if the package is poorly designed).

Now, let's install a new release called `custom-test`, passing the `my-values.yaml` file to override the default values:

```
helm install -f my-values.yaml custom-test demo-nginx/
```

<a name="dependency"></a>
## Creating a Chart with Dependencies on Another Chart

In this example, we will create a chart with a primary deployment (application) and a MySQL database declared as a dependency.

__IMPORTANT__: When declaring a dependency, you must familiarize yourself with the dependent package, know the service name it will use, the secrets it will create, and everything necessary to integrate it into your chart.

- Add in `Chart.yaml`:

```
dependencies:
- name: mysql
  version: ~9.3.2
  repository: https://charts.bitnami.com/bitnami
  condition: mysql.enabled
```

(The `condition` clause allows the dependency to be optional.)

- Download dependencies:
```
helm dependency build my-chart/
```

- Test rendering in any of the following ways:

```
helm template .
helm template --set mysql.enabled=true
```

- To configure the dependent chart (`mysql`) in the example, we can include the following in our `values.yaml`:

```
# Main chart content in values.yaml...
ingress:
  enabled: true
  ...
  ...

# Content for the MySQL dependency
mysql:
  auth:
    createDatabase: true
    database: mydb
    username: user
    initdbScripts:
      my_init_script.sql: |
        use mydb;
        create table students (first_name varchar(255), last_name varchar(255));      
```

To know the allowed `values` content for the MySQL Helm chart, study the official chart documentation on [artifacthub.io](https://artifacthub.io/packages/helm/bitnami/mysql).

<a name="multiple"></a>
## Creating a Chart with Multiple Controllers (Deployments, StatefulSets, ...)

The challenges we face when deploying multiple controllers in our chart (for example, two deployments, a deployment, and a statefulSet, etc.) are:

- Resource names: If we simply use `{{.Release.Name}}` or the `fullname` helper, it won't be sufficient.

- Pod labels and selectors: If we use the helper, it won't be enough since the different resources need unique labels.

What can we do about this?

Tips:

- Always add some content to the Kubernetes resource name to make it more descriptive.

- Always add a custom `label` to your components even if you use the helpers.

<a name="adv"></a>
## Exploring More Complex Options

Let's take a deeper look at the default content of the Helm chart:

```
helm create demo-analysis
```

Explore the content and focus on:

- `values.yaml`, its structure, and the features it offers (they are `best practices`). You will see that many parts are left to be filled directly with Kubernetes syntax (for example, `resources`, `tolerations`, `affinity`).

- Look at the `deployment.yaml` and see how it generates the final deployment, how it uses the various objects from values, and where it uses the `helpers`.

- The relationship between `replicas` and `.Values.autoscaling.enabled`.

- `nodeSelector`, `affinity`, and `tolerations` are in `width` blocks, but `resources` is not. Why?


## End
[Back](./README.md)

