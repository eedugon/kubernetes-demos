<a name="pod"></a>
# Using ConfigMaps in Kubernetes

Table of contents:

- [Creating ConfigMaps from different sources](#create)
- [Using ConfigMaps as environment variables in pods](#env)
- [Using ConfigMaps as volumes and mount points in pods](#volume)

<a name="create"></a>
## Creating ConfigMaps from different sources

For the demonstration of various methods, we will use the following resources:

- Directory `configmap-data` containing source files for our `ConfigMaps`. Inside, you'll find:

  - File `game.properties` with the following content:

  ```
  enemies=aliens
  lives=3
  enemies.cheat=true
  enemies.cheat.level=noGoodRotten
  secret.code.passphrase=UUDDLRLRBABAS
  secret.code.allowed=true
  secret.code.lives=30
  ```

  - File `ui.properties` with the following content:
  ```
  color.good=purple
  color.bad=yellow
  allow.textmode=true
  how.nice.to.look=fairlyNice
  ```

  - Other files such as `datos-texto` and `filebeat-conf.yaml` to demonstrate that ConfigMaps can be sourced from any type of file.

### From declarative yaml

We can create a `ConfigMap` from a `yaml` file like any other Kubernetes object.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  conf.properties: |
    color=blue
    country=Spain
  conf.json: |
    {"color": "blue", "country": "Spain"}
  otrakey: "value"
  filetext: |
    this is the content
    of the ConfigMap key
    representing
    a text file  
```

To create it:

```
kubectl apply -f resources/configmap.yaml
```

### Directories

We can create a ConfigMap from the entire content of a directory. Each file will become a "key" (with the file name), and the value will be the file's content:

```
$ kubectl create configmap game-config --from-file=./resources/configmap-data/
configmap/game-config created
```

If we run `describe`, we will see the following:

```
$ kubectl describe configmaps game-config
Name:         game-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30

BinaryData
====

Events:  <none>
```

This is not the most common use case, as ConfigMaps are generally created from specific files.

### Files

We can create ConfigMaps from individual files using the `--from-file` option:

```
kubectl create configmap game-config-2 --from-file=resources/configmap-data/game.properties
```

### Env files

If the file contains environment variables in the correct format, the `--from-env-file` option allows saving each environment variable as a distinct `key` (instead of using the file name as the `key`).

```
kubectl create configmap game-config-env-file --from-env-file=resources/game-env-file.properties
```

The above will produce a `ConfigMap` like this:

```yaml
data:
  allowed: '"true"'
  enemies: aliens
  lives: "3"
```

Instead of the default file processing that would create:

```yaml
data:
  game-env-file.properties: |
    enemies=aliens
    lives=3
    allowed="true"

    # This comment and the empty line above it are ignored
```

__Notes__:
  - `--from-file` can be used multiple times in the same command to add multiple keys to the ConfigMap.
  - `--from-env-file` cannot be used multiple times in the same command; only the last file will be processed.

### Choosing `key` when creating ConfigMap from a file

The following syntax lets you choose the generated `key` instead of using the file name: `kubectl create configmap game-config-3 --from-file=<my-key-name>=<path-to-file>`.

For example:

```
kubectl create configmap game-config-3 --from-file=game-special-key=resources/configmap-data/game.properties

# The above command generates:
data
  game-special-key: |-
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
```

### Creating ConfigMap from literal values

This will generate as many keys and values as you add:

```
kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
```

The result will be:

```yaml
data:
  special.how: very
  special.type: charm
```

<a name="env"></a>
## Using ConfigMaps as environment variables in pods

### Defining environment variables from ConfigMaps

For the demonstration, we will create a simple `ConfigMap` with one `key`:

```
kubectl create configmap special-config --from-literal=special.how=very
```

We add the environment variable `SPECIAL_LEVEL_KEY` to a container in a pod, referencing the created ConfigMap and the key `special.how` (the value should be `very`).

To do this, we will use `valueFrom` in the environment variable specification.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-value-from
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
              name: special-config
              # Specify the key associated with the value
              key: special.how
  restartPolicy: Never
```

We create the pod:

```
kubectl apply -f resources/pod-configmap-value-from.yaml
```

We check the result:

```
$ kubectl logs pod-configmap-value-from| grep  SPECI
SPECIAL_LEVEL_KEY=very
```

## Adding all keys from a ConfigMap as environment variables

We will create the following ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config-multi
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

Run the command:

```
kubectl apply -f resources/configmap-multikeys.yaml
```

Use `envFrom` in the container spec to load an entire ConfigMap as environment variables.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-env-from
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: special-config-multi
  restartPolicy: Never
```

We launch the example:

```
kubectl apply -f resources/pod-configmap-env-from.yaml
```

We check the result:

```
$ kubectl logs pod-configmap-env-from | grep SPEC
SPECIAL_LEVEL=very
SPECIAL_TYPE=charm
```

### Using environment variables in command execution (CMD / ENTRYPOINT)

This is not strictly related to `ConfigMaps`, as environment variables can always be used in the `command` (entrypoint) or `args` of the container specification, regardless of whether they come from a `ConfigMap`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-command
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/echo", "$(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config-multi
              key: SPECIAL_LEVEL
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config-multi
              key: SPECIAL_TYPE
  restartPolicy: Never
```

To test and check:

```
$ kubectl apply -f resources/pod-configmap-command.yaml
pod/pod-configmap-command created

$ kubectl logs pod-configmap-command
very charm
```

<a name="volume"></a>
## Using ConfigMaps as volumes and mount points in pods

We can define a `volume` that loads data from a ConfigMap. This volume can then be `mounted` in the container as a file or directory, depending on the options used in both the volume definition and mount configuration.

### Mounting directories

When we create a `volume from a ConfigMap` and mount it to a destination, that destination will be a directory, and each `key` in the ConfigMap will be a file in the final directory, even if the `ConfigMap` only has one key.

Example of directory mounting:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-mount-volume
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config-multi
  restartPolicy: Never
```

To test this example:

```
kubectl apply -f resources/pod-configmap-mount-volume.yaml

kubectl logs pod-configmap-mount-volume
```

### Mounting files (I)

To mount only one `file` from a `ConfigMap` with multiple keys, we define the `key` of the `ConfigMap` directly in the volume declaration, and provide a `path` to indicate the final file name.

Example of file mount point:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-mount-file
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh","-c","ls -l /etc/config/filetest; echo; echo \"# Content\"; cat /etc/config/filetest" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config-multi
        items:
        - key: SPECIAL_LEVEL # ConfigMap key
          path: filetest # destination file
  restartPolicy: Never
```

The example above defines the `config-volume` volume, loading only the key `SPECIAL_LEVEL`, and stores it with the path `filetest` (the volume will have a file named `filetest` instead of `SPECIAL_LEVEL`). The volume is then mounted at `/etc/config`, so the `filetest` file ends up in `/etc/config`.

To test this example:

```
kubectl apply -f resources/pod-configmap-mount-file.yaml

kubectl logs pod-configmap-mount-file
```

The output from the previous command

### Mounting files (II)

The most typical case is to mount only one file from a `ConfigMap` where only that file is defined.

- Imagine we have a `ConfigMap` that defines `config.yaml` as follows:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-mount-simple
  namespace: default
data:
  config.yaml: |
    # this is a test yaml file
    key: value
    array:
      - "one"
      - "two"
    another: test
```

- The goal is to mount this file to `/app/config.yaml` in a container (for example).

To do this, there are two options:
- Define the volume, loading it from the `ConfigMap`, and specifying the `key` and `path` (as in the previous example).
- Define the volume by loading the entire `ConfigMap`, and in the `volumeMount`, specify a `subPath`. The `subPath` indicates that we only want to mount part of the volume (in this case, a specific file) to the destination. Otherwise, an entire `volume` will always be mounted as a `directory`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-mount-simple
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh","-c","cat /app/config.yaml" ]
      volumeMounts:
      - name: config-volume
        mountPath: /app/config.yaml # mount destination
        subPath: config.yaml # ConfigMap key
  volumes:
    - name: config-volume
      configMap:
        name: configmap-mount-simple
  restartPolicy: Never
```

If we removed the `subPath` in the previous example, the volume would be mounted as a directory `/app/config.yaml`, and inside that directory, we would find the file. Obviously, that is not what we want.

To test this example:
```
# Create ConfigMap and POD
kubectl apply -f resources/configmap-mount-simple.yaml
kubectl apply -f resources/pod-configmap-mount-simple.yaml

# View the pod logs (executes cat /app/config.yaml)
kubectl logs pod-configmap-mount-simple
# this is a test yaml file
key: value
array:
  - "one"
  - "two"
another: test
```

__Troubleshooting__

Common issues:
- You try to mount one or more files, and they are mounted as __directories__. Check the `ConfigMap`, and especially the `volume` and `volumeMount` definitions.

### End
[Back](./README.md)
