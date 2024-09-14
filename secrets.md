<a name="secrets"></a>
# Using Secrets in Kubernetes

Secrets have a similar nature and usage to `ConfigMaps`, so in this document, we will focus on the specific details of secrets due to their encoded nature.

Check out the [ConfigMaps demonstration](./configmaps.md) before continuing.

Table of Contents:

- [Creating Secrets](#create)
- [Decoding Secrets](#decode)
- [Using Secrets](#use)

<a name="create"></a>
## Creating Secrets

Secrets are special because they are encoded, making their creation slightly different from `ConfigMaps`.

### Manually, via a YAML file

We can create `Secrets` directly through YAML files. However, we need to encode the key values in `base64` beforehand (unlike when creating `ConfigMaps`).

If we want to add the values 'admin' and '1f2d1e2e67df' to a secret that represents a username and password, we must first do the following:

```
echo -n 'admin' | base64
YWRtaW4=
echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

Once we have the encoded values, we can create the `Secret` as follows:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-manual
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

We can then apply it using `kubectl`:

```
kubectl apply -f resources/secret-manual.yaml
```

Note: Instead of using the `data` section (which requires base64 encoding), we could use `stringData` to store the values as plain text.

### From files

Now imagine we have stored the username and password required by an application in two local files:

```
# Create files for the example.
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt
```

We can create a `Secret` with these two keys like this:
```
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
```

If we inspect the created `Secret`...

```
kubectl describe secrets/db-user-pass

Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

__Note__: As you can see, the keys are named differently (`username.txt` and `password.txt`). How could we ensure the keys are named `username` and `password` instead?

### From literals

Similar to `ConfigMaps`, we can add keys directly from the command line. __Important__: Some special characters (e.g., `$`, `\`, `*`, `!`) might need to be escaped to prevent the shell from interpreting them before running the command, so it's advisable to use single quotes around the values.

Examples:
```
kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests

# Example using single quotes to avoid escaping characters.
kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='
```

<a name="decode"></a>
### Decoding Secrets

If we retrieve the following from a Secret:

```
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

We can decode each key using the `base64 --decode` command:

```
$ echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
1f2d1e2e67df
```

<a name="use"></a>
### Using Secrets

The use of Secrets is similar to `ConfigMaps`. Review the [ConfigMaps demonstration](./configmaps.md) if you need clarification on how they work and how files or directories are mounted.

Here are some examples and comments for reference:

### Using Secrets as volumes

As with `ConfigMaps`, inside the container that mounts a Secret volume, the `keys` of the Secret appear as files, and the values of the Secret are decoded from base64 and stored inside these files.

- Example 1: The content of `mysecret` is mounted as a __directory__ at `/etc/foo` in the container (multiple files or whatever is in the Secret).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

- Example 2: The `username` key from the `mysecret` Secret will be mounted as a __file__ at `/etc/foo/my-group/my-username`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

### Using Secrets as environment variables

- Example: Two environment variables are configured from two different keys in the __same Secret__.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```

### End
[Back](./README.md)
