Technical Knowledge article

#  My Tips and Tricks for leveraging OpenTelemetry with a LGM stack to securely gather K8S telemetry data
#  (1/3) Kubernetes Events Collection

![](images/kube_otel.png)

## Introduction
OpenTelemetry is a CNCF project in the incubating stage. (https://www.cncf.io/blog/2021/08/26/opentelemetry-becomes-a-cncf-incubating-project/)

The numerous articles and talks about OpenTelemetry demonstrate increasing adoption within the Kubernetes and open-source community.
It's a solution designed to facilitate the collection and enrichment of telemetry data, including metrics, logs, and traces.
Its strength lies in bundling a comprehensive set of components and features for metric collection and processing, enabling integration with almost any system.

I implemented OpenTelemetry to collect events, logs, and metrics from a Kubernetes cluster, where previously I used tools like node-exporter, kube-stats-metrics, and promtail with the Loki/Grafana/Mimir suite.

Here, I share some implementation and security tips. This first article covers Kubernetes Events Collection (1/3), followed by    Kubernetes Logs Collection (2/3) and Kubernetes metrics collection (3/3).

## Prerequisite
 - A functional Kubernetes cluster
 - The LGM suite configured with 2 data sources:
    - https://loki.172.18.1.1.nip.io
    - https://mimir.172.18.1.1.nip.io

    The data sources are configured to receive data with basic authentication and require the presence of an X-Scope-OrgID field in the request header for authorization (in multi-tenant management mode).

## OpenTelemetry
OpenTelemetry implement an ETL (Extract, Transform, Load) approach. (https://en.wikipedia.org/wiki/Extract,_transform,_load)
![](images/etl.png)

This approach involves:
  - Extracting telemetry data from applications and services. For us, it means intercepting Kubernetes events, extracting container logs, and retrieving cluster metrics via kubelet.
  - Transforming and enriching them into a standardized format. For example, adding labels to facilitate processing in Grafana and indexing in Loki and Mimir.
  - Then Loading them to the storage or analysis system, thus to the Loki/Grafana/Mimir suite.

These three phases form the service/pipeline of a collector in the OpenTelemetry context with 3 phases: receiver (extract), processor (transform), export (load).

## First Tip: Using the OpenTelemetry-operator
https://github.com/open-telemetry/opentelemetry-operator

The OpenTelemetry-operator simplifies the deployment and dynamic configuration of OpenTelemetry Collectors in Kubernetes environments using the CRD.
It handles the deployment, update, and scaling of the collectors that we decide to instantiate.

It also provides Instrumentation CRD to supports injecting and configuring auto-instrumentation libraries for .NET, Java, Node.js, Python, and Go services. We will not use this feature in this article.

<img src="images/operator.png " style="width: 70%;" />

Let's install the OpenTelemetry-operator in a dedicated namespace called "otel".

```
helm repo add open-telemetry https://open-telemetry.github.io/OpenTelemetry-helm-charts
helm upgrade --install OpenTelemetry-operator open-telemetry/OpenTelemetry-operator -n otel --create-namespace
```

## Tip 2: "Do not put all your eggs in one basket" or "Implementing one collector per type of data"

If your collector gathers multiple sources of information through multiple receivers, the interruption of the collector stops the data collection from all receivers.
If you inadvertently introduce an error while editing your manifest, the collector will not start, and no data will be collected, even for correctly configured receivers.

I therefore encourage you to declare:
 - one collector for cluster events
 - one collector for cluster logs
 - one collector for cluster metrics
 - and specific collectors for your applications.

## Implementing our Kubernetes Events Collector
### Tip 3 :  RBAC
A quick reminder, there are mainly two ways to read events from a Kubernetes cluster through the API server.
```
kubectl describe pod <podname>
kubectl get events
```

If you are able to retrieve this information, it means your user account allows you to read this information from your cluster. By default, our OpenTelemetry collector does not have this privilege. We need to grant it permission to read this information. How? Thanks to RBAC ;)

The manifest otel/otel_rbac_K8S-events.yaml grants read access to Kubernetes Events for pods using the serviceAccount 'otel-k8sevent' in the namespace 'otel'.

```
kubectl apply -f  otel/otel_rbac_K8S-events.yaml
```
The collector is configured to use this serviceAccount.
```
  ...
  metadata:
    name: k8s-event-collector
    namespace: otel
  spec:
    serviceAccount: otel-k8sevents
  ...

```

### Tip 4 : SECRET and ENV
To export telemetry information to the Loki data source as logs in our LGM suite, the collector exporter must authenticate using a login/password and transmit the identifier of our tenant via the X-Scope-OrgID field in the HTTP request header. This information is "sensitive" and should not be written in plain text in the collector manifest! We will use the Kubernetes "Secret" object to store this information.

```
kubectl apply -f  otel/otel_secret_loki.yaml
```

In the collector declaration, we will be able to read the secrets through environment variables, but also define new variables to store the node_name, for example.
```
  ...
  spec:
    serviceAccount: otel-k8sevents
    env:
    - name: K8S_NODE_NAME
     valueFrom:
       fieldRef:
         fieldPath: spec.nodeName
   - name: OPEN_TELEMETRY_COLLECTOR_ORGID
      valueFrom:
       secretKeyRef:
         name: loki-creds
         key: X-SCOPE-ORGID
    - name: OPEN_TELEMETRY_COLLECTOR_USERNAME
     valueFrom:
       secretKeyRef:
         name: loki-creds
         key: USER
   - name: OPEN_TELEMETRY_COLLECTOR_PASSWORD
     valueFrom:
       secretKeyRef:
         name: loki-creds
         key: PASSWORD
  ...
```
and environment variables will be referenced using "$" such as:
```
    ...
    headers:
      X-Scope-OrgID: $OPEN_TELEMETRY_COLLECTOR_ORGID
    ...
```

### Tip 5 : The Deployment Mode for the Collector
The collector can be deployed in four modes: deployment, statefulset, daemonset, and sidecar.
Today, we will exclusively focus on discussing the Deployment and DaemonSet modes, considering our specific use case.

- If we need to collect logs from each container or kubelet metrics from each node, we need to install a collector on each node of our cluster. In this case, we will choose the "DaemonSet" mode. One collector instance will be deploy on each node.

<img src="images/daemonset.png " style="width: 60%;" />

- Event collection is done by querying the Kubernetes API server. Only one instance is required, regardless of its location. In this case, we will use the "Deployment" mode.

<img src="images/deployment.png " style="width: 60%;" />

```
  ...
  spec:
    serviceAccount: otel-k8sevents
    mode:  deployment
    env:
  ...
```

###  Tip 6 : K8S-Event Config
The 'Config' of the OpenTelemetry Collector is divided into 5 steps:
 - Receivers, which retrieve telemetry data
 - Processors, which handle and transform events
 - Exporters, which send events to their storage destinations
 - Extensions to manage specifics operation like authentication
 - Service, which connects and orchestrates the previous configurations

The image "otel/OpenTelemetry-collector-contrib" (https://github.com/open-telemetry/OpenTelemetry-collector-contrib) includes an extensive set of plugins (receivers, processors, exporters) allowing integration with almost any system.

We will implement the following configuration for our k8s-event-collector :
![](images/service-events.png)

Let's start with the Receivers block:
```
  ...
  config : |
   receivers:
      k8s_events:
        namespaces: []
        auth_type: serviceAccount

  ...
```
The k8s_events receiver collects events from all namespaces using the serviceAccount to authenticate with the Kubernetes API Server.
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8seventsreceiver/README.md


The Processors block will allow us to enrich the collected data by adding attributes such as node, cluster, and receiver.
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourceprocessor/README.md
```
  ...
  config : |
    receivers: ...
    processors:
      resource/k8s_events:
        attributes:
          - action: insert
            key: cluster
            value: $OPEN_TELEMETRY_COLLECTOR_ORGID
          - action: insert
            key: node
            value: $K8S_NODE_NAME
          - action: insert
            key: receiver
            value: 'k8s_event'
          - action: insert
            key: loki.resource.labels
            value: node,receiver,cluster
    ...
```
Here we find an example of the utilization of our previoulsy defined environment variables OPEN_TELEMETRY_COLLECTOR_ORGID and K8S_NODE_NAME.

The Exporters block allows exporting the collected and enriched data to their final destination: Loki.
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/lokiexporter/README.md

```
  ...
  config : |
    receivers: ...
    processors: ...
    exporters:
      loki:
        endpoint: https://loki.172.18.1.1.nip.io/loki/api/v1/push
        headers:
          X-Scope-OrgID: $OPEN_TELEMETRY_COLLECTOR_ORGID
        auth:
          authenticator: basicauth/client
        tls:
          insecure: false
          insecure_skip_verify: true
    ...
```
If your endpoint is using an insecure http channel, *insecure* must be set to true, and  *insecure_skip_verify* be omited.
If your endpoint is using an insecure https channel with a self-signed-certificate, *insecure* must be set to false ,and *insecure_skip_verify* to true

The Extension block allows us to configure the authentication mechanism for the exporter.
```
  ...
  config : |
    receivers: ...
    processors: ...
    exporters: ...
    extensions:
      basicauth/client:
        client_auth:
          username: $OPEN_TELEMETRY_COLLECTOR_USERNAME
          password: $OPEN_TELEMETRY_COLLECTOR_PASSWORD
    ...
```

The implementation of our 4 steps is orchestrated by the Services block.
```
  config : |
    receivers: ...
    processors: ...
    exporters: ...
    extensions: ...
    service:
      extensions: [basicauth/client]
      pipelines:
        logs:
          receivers: [k8s_events]
          processors: [resource/k8s_events]
          exporters: [loki]
  ```
Here is our complete OpenTelemetry file, you can view it here. It is ready to be deployed.
```
kubectl apply -f  otel/opentelemetry-k8s_event.example.yaml
```

### View the collected data in Loki/Grafana Dashboard
Finaly we can connect to our grafana instance and explore Loki DataSource ... apply a filter and Yes we got our kubernetes events!!!!

![](images/loki-events.png)

I hope you enjoy this article, so please let me know !!!!

And see you soon for the next one : Kubernetes Logs Collection (2/3)

