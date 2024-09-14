# APM (Application Performance Management) Demonstration

Application instrumentation is a modern monitoring approach that integrates monitoring directly into the code.

Architecture/diagram of Elastic's APM solution is available in this [blog](https://www.elastic.co/blog/adding-free-and-open-elastic-apm-as-part-of-your-elastic-observability-deployment).

Official documentation can be found [here](https://www.elastic.co/guide/en/apm/guide/8.10/apm-overview.html).

We will need an APM Server connected to the Elastic Stack. Here’s how to set it up:

## APM Server Installation and APM Integration in Kibana

__Step 1__: Deploy the APM server connected to our Elastic Stack.

We will [deploy an APM Server using the Elastic ECK operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-apm-server.html) and connect it to the cluster created in the previous demonstration:

```yaml
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-server-quickstart
  namespace: default
spec:
  version: 8.10.2
  count: 1
  elasticsearchRef:
    name: logging-and-metrics
  kibanaRef:
    name: logging-and-metrics
```

- __Note__: As you can see, it’s incredibly easy!

The ECK operator will automatically create a `secret token` that we can use in our applications to connect. This information is stored in a `secret`. More details can be found [here](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-apm-connecting.html#k8s-apm-secret-token).

```bash
kubectl get secret/apm-server-quickstart-apm-token -o go-template='{{index .data "secret-token" | base64decode}}'
```

The connection URL for applications running inside the Kubernetes cluster will be the name of the created service:
- `https://apm-server-quickstart-apm-http.default.svc.cluster.local:8200`

For testing from our local machine, we can expose the APM service externally via a `LoadBalancer` or use `kubectl port-forward` for local testing.

In one terminal, monitor the APM server logs:

```
kubectl logs -f apm-server-quickstart-apm-server-f6b566797-nc22d
```

__Step 2__: Once the APM server is running, install the `APM` integration in Kibana. This will create the necessary templates and indices in Elasticsearch for APM to function properly.

- [APM Integration in Kibana](https://logging-and-metrics.keep.demo:5601/app/integrations/detail/apm/overview)

## Instrumenting a Python Application:

Elastic APM supports many programming languages via different [agents](https://www.elastic.co/guide/en/apm/agent/index.html).

The `Elastic Python APM agent` is documented [here](https://www.elastic.co/guide/en/apm/agent/python/current/getting-started.html), with a section specifically for `Flask`.

What do we need to add to our Flask application to connect it with the APM server?

- Add `elastic-apm[flask]` to the `requirements.txt` file to ensure `pip install` installs the necessary libraries.
- Add the following code:

```python
from elasticapm.contrib.flask import ElasticAPM

app = Flask(__name__)  # This line already existed

apm = ElasticAPM(app)
```

Is that all?

Yes! That’s all it takes! The agent configuration can be done directly through environment variables.

Important references:
- [Flask Guide](https://www.elastic.co/guide/en/apm/agent/python/current/flask-support.html)
- [Python Agent Configuration](https://www.elastic.co/guide/en/apm/agent/python/current/configuration.html)
- [Supported Python Frameworks](https://www.elastic.co/guide/en/apm/agent/python/current/supported-technologies.html)

You can configure the agent using the following environment variables, which will be executed along with your Flask application:

- ELASTIC_APM_ENABLED --> Enables or disables APM.
- ELASTIC_APM_SERVICE_NAME --> Service name for our application.
- ELASTIC_APM_SERVER_URL --> APM server URL.
- ELASTIC_APM_SECRET_TOKEN --> Token for authenticating with the APM Server. Several [authentication mechanisms](https://www.elastic.co/guide/en/apm/server/current/authorization.html) are available.
- ELASTIC_APM_VERIFY_SERVER_CERT --> Use this if you want to skip certificate verification when using private certificates.

Example of environment variables:

```bash
ELASTIC_APM_SECRET_TOKEN="V4PA1O96P574YxtB3dy2q5xH"
ELASTIC_APM_SERVICE_NAME="flask-counter"
ELASTIC_APM_SERVER_URL="https://34.76.192.60:8200"
ELASTIC_APM_ENABLED="True"
ELASTIC_APM_VERIFY_SERVER_CERT="False"
```

### Generating Traffic for the Application:

- Example for local testing:

```bash
while :; do curl -q localhost:5000 > /dev/null 2>&1; done
```

- Example with a Kubernetes pod:

(Work in progress)

```
```

### End
[Back](./README.md)
