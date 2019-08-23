Deploying the SignalFx Smart Gateway With Helm
==============================================

The SignalFx Smart Gateway can be deployed in Kubernetes environments with
[Helm][g] using the [signalfx/signalfx-smart-gateway][h] chart. The chart
supports single instance deployments and multi-node cluster deployments. A
Kubernetes service will be created as part of the deployment for sending
datapoints, events, and traces to the Smart Gateways.

## Quick Start

The [signalfx/signalfx-smart-gateway][h] Helm chart deploys a SignalFx Smart
Gateway cluster. It requires an existing etcd cluster for the gateway nodes to
connect to.

### Build the SignalFx Smart Gateway Image

Before installing the Helm chart, a SignalFx Smart Gateway image must be
generated.

1. Download the SignalFx Smart Gateway [binary][c].
2. Use the [Dockerfile][d] to build the image.
3. Place the image in a registry accessible to the Kubernetes cluster where the
Helm chart will be installed.

### Install the Helm Chart

To use the chart with Helm, add our SignalFx Helm chart repository to Helm
like this:

```bash
$ helm repo add signalfx https://dl.signalfx.com/helm-repo
```

Then to ensure the latest state of the repository, run:

```bash
$ helm repo update
```

Then you can install the gateway using the chart name
`signalfx/signalfx-smart-gateway`.

#### Installing a Single Instance

For a single instance, be sure to set values for:
- `signalFxAccessToken`
- `clusterName`
- `image.repository`
- `image.tag`

```bash
$ helm install signalfx/signalfx-smart-gateway \
--set image.repository=<YOUR_SMART_GATEWAY_REPOSITORY> \
--set image.tag=<YOUR_SMART_GATEWAY_TAG> \
--set signalFxAccessToken=<YOUR_ACCESS_TOKEN> \
--set clusterName=<YOUR_CLUSTER_NAME>
```

#### Installing A Cluster

SignalFx Smart Gateway clusters require client access to an etcd cluster in
order to coordinate instances within the cluster.  

*See [Setting Up An etcd Cluster](#Setting-Up-An-etcd-Cluster)
for information about deploying an etcd cluster with the [etcd-operator][a]
Helm [chart][b].*

For a cluster, be sure to set values for:
- `signalFxAccessToken`
- `clusterName`
- `targetClusterAddresses`
- `image.repository`
- `image.tag`
- `gateway.count`

```bash
$ helm install signalfx/signalfx-smart-gateway \
--set image.repository=<YOUR_SMART_GATEWAY_REPOSITORY> \
--set image.tag=<YOUR_SMART_GATEWAY_TAG> \
--set signalFxAccessToken=<YOUR_ACCESS_TOKEN> \
--set clusterName=<YOUR_CLUSTER_NAME> \
--set targetClusterAddresses[0]=<YOUR_ETCD_CLUSTER_CLIENT_ADDRESS> \
--set gateway.count=3
```

A service will be created to forward requests to port `18080` on to the 
gateways' SignalFx Listener.

#### Setting Up An etcd Cluster

- Install the [etcd-operator][a] Helm [chart][b]
and configure the chart to setup RBAC resources for the operator

```bash
$ helm install \
--name etcd-operator \
--set customResources.createEtcdClusterCRD=true \
--set customResources.createBackupCRD=true \
--set customResources.createRestoreCRD=true \
stable/etcd-operator
```

- Update the chart to enable the cluster. `cluster.enabled` is ignored on
install.

```bash
$ helm upgrade --set cluster.enabled=true etcd-operator stable/etcd-operator
```

## Persisting Sampling Data

By default the SignalFx Smart Gateway forwarder is configured to persist and 
backup data to /var/lib/gateway/data. The Smart Gateway buffers in-flight trace
spans during its operation, and persists information about span and trace
duration at this location on shutdown. To ensure the Smart Gateway continues to
operate nominally and accurately selects the best traces to retain, you must
create a persistent volume mount for this location in the containers. This can
be done by specifying a volume claim template in `gateway.volumeClaimTemplates`.
Please configure the accessModes to be `ReadWriteOnce` so that a new claim will
be created with each replica. Please refer to the Kubernetes documentation on
Persistent Volume Claims (PVC's) for more details on configuring persistent
storage and claiming space in that storage. 

After specifying the volume claim templates, a volume mount should be configured
to use the persistent volume claim under `gateway.volumeMounts`.

### Setting a Persistent Volume Claim Template

This is an example of a volume claim template in this chart's values.yaml.

```yaml
gateway:
  volumeClaimTemplates:
    - metadata:
        name: "gateway-backups"
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "my-storage-class"
        resources:
          requests:
            storage: 20Gi
```

The following snippet shows how to configure the preceding volume claim 
template while installing the Helm chart.

```bash
--set gateway.volumeClaimTemplates[0].metadata.name="gateway-backups" \
--set gateway.volumeClaimTemplates[0].spec.accessModes[0]="ReadWriteOnce" \
--set gateway.volumeClaimTemplates[0].spec.storageClassName="storage-device" \
--set gateway.volumeClaimTemplates[0].spec.resources.requests.storage="20Gi"
```

### Setting a Volume Mount

This is an example of a volume mount in this chart's values.yaml.

```yaml
gateway:
  volumeMounts:
    - name: "gateway-backups"
      mountPath: "/var/lib/gateway/data"
```

The following snippet shows how to configure the preceding volume mount while
installing the Helm chart.

```bash
--set gateway.volumeMounts[0].mountPath="/var/lib/gateway/data" \
--set gateway.volumeMounts[0].name="gateway-backups"
```

## About This Chart

This chart deploys a cluster of SignalFx Smart Gateways and creates a service
definition in front of them.

### Important Configurations
Before proceeding, it is recommended that you look at the [values.yaml] file
included with the Helm chart. The key `gateway` represents specific
configurations about the SignalFx Smart Gateways.

For convenience, there are a few top level configurations in the [values.yaml]
to insert and configure Forwarders and Listeners for the Smart Gateway.

#### ClusterName

The SignalFx Smart Gateway must all use the same cluster name. As a convenience
there is a top level configuration called `clusterName`. It will be inserted
into `gateway.conf.ClusterName`.

#### Listeners

The `listeners` configuration is a list of SignalFx Smart Gateway Listener
configuration JSON objects. These listeners will be merged into the
[values.yaml] lists `gateway.conf.ListenFrom`. There is a default SignalFx
Listener configuration stored in `listeners[0]` and it is configured to listen
on port `18080`. Please refer to the Listener [documentation][e] for more
information about Listeners.

#### Forwarders

The `forwarders` configuration is a list of SignalFx Smart Gateway Forwarders
configuration JSON objects. These forwarders will be merged into the
[values.yaml] list `gateway.conf.ForwardTo`. Please note this difference in
configuration. While the default SignalFx Listener is under `listeners[0]`,
but the default SignalFx Forwarder is under `gateway.conf.ForwardTo`.

#### Target Cluster Addresses

The `targetClusterAddresses` configuration is a list of etcd client addresses
for the gateways to connect to etcd. This configuration will be inserted into
`gateway.conf.TargetClusterAddresses`.

#### Monitoring Via SignalFx Smart Agent

This helm chart configures an internal metrics server on the Smart Gateway that
the SignalFx Smart Agent can scrape for metrics about the Smart Gateway.

##### Smart Gateway

`gateway.conf.InternalMetricsListenerAddress` turns on the internal metrics
server on the Smart Gateway.  By default metrics are served on port 2383.  The
port is exposed on the container via `gateway.containerPorts[2]`.

##### Smart Agent

When a SignalFx Smart Agent is configured with the [Kubernetes Observer][i] it
will discover Smart Gateway instances and configure the internal-metrics monitor
according to the annotations on the pod.  This is driven by two default
Kubernetes annotations added by this helm chart to the Smart Gateway pod.  See
the helm chart config value `gateway.annotations` for more information.

#### gateway.conf

You'll notice that there is a configuration called `gateway.conf` that is a JSON
config objects representing plain SignalFx Smart Gateway config. It is passed in
directly so additional gateway configurations that are not in the
[values.yaml] can be set by directly specifying the path through this object.
Please refer to the [SignalFx Smart Gateway Deployment Guide][f] for more
information about configurations for the gateway. Please note that `listeners`
and `forwarders` will be merged into their corresponding fields in
`gateway.conf`. `gateway.conf.TargetClusterAddresses` is always overridden by
`targetClusterAddresses`. `gateway.conf.ClusterOperation` and will always be
ignored and set intelligently by the Helm chart.

## Chart Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| clusterName | string | `""` | The name of the SignalFx Smart Gateway cluster (REQUIRED) |
| forwarders | list | `[]` | Additional SignalFx Smart Gateway forwarder config objects (These will be merged into the `gateway.conf.ForwardTo` list) |
| fullnameOverride | string | `""` | Overrides the full name of the Helm chart |
| gateway | object | `{"advertisedSFXListenerAddress":"$(POD_IP):18080","advertisedSFXRebalanceAddress":"$(POD_IP):2382","affinity":{},"annotations":[{"agent.signalfx.com/monitorType.2383":"internal-metrics"},{"agent.signalfx.com/config.2383.path":"/internal-metrics"}],"conf":{"ForwardTo":[{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}],"InternalMetricsListenerAddress":"0.0.0.0:2383","ListenFrom":[],"LogDir":"-"},"containerPorts":[{"containerPort":18080,"name":"sfx-listener","protocol":"TCP"},{"containerPort":2382,"name":"sfx-rebalance","protocol":"TCP"},{"containerPort":2383,"name":"sfx-internal-metrics","protocol":"TCP"}],"count":1,"livenessProbe":{"httpGet":{"path":"/healthz","port":"sfx-listener"},"initialDelaySeconds":15},"nodeSelector":{},"readinessProbe":{},"resources":{},"tolerations":[],"volumeClaimTemplates":[],"volumeMounts":[],"volumes":[]}` | Configurations for SignalFx Smart Gateway nodes |
| gateway.advertisedSFXListenerAddress | string | `"$(POD_IP):18080"` | An environment variable that will be set in the gateway pod. It is used to configure the SignalFx Smart Samplers to advertise their configured SignalFx Listener in the cluster. By default it is set to `$(POD_IP):<PORT>` which is filled in by Kubernetes. |
| gateway.advertisedSFXRebalanceAddress | string | `"$(POD_IP):2382"` | An environment variable that will be set in the gateway pod.  It is used to configure the SignalFx Smart Samplers to advertise their configured rebalance address in the cluster.  By default it is set to `$(POD_IP):<PORT>` which is filled in by Kubernetes. |
| gateway.affinity | object | `{}` | Kubernetes affinity configuriations to apply to the pod |
| gateway.annotations | list | `[{"agent.signalfx.com/monitorType.2383":"internal-metrics"},{"agent.signalfx.com/config.2383.path":"/internal-metrics"}]` | A list of key/value objects representing annotations to apply to the pods. |
| gateway.annotations[0] | object | `{"agent.signalfx.com/monitorType.2383":"internal-metrics"}` | An annotation to configure the internal-metrics monitor on the SignalFx Smart Agent. |
| gateway.annotations[1] | object | `{"agent.signalfx.com/config.2383.path":"/internal-metrics"}` | An annotation to configure the path on the SignalFx Smart Agent's internal-metrics monitor. |
| gateway.conf | object | `{"ForwardTo":[{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}],"InternalMetricsListenerAddress":"0.0.0.0:2383","ListenFrom":[],"LogDir":"-"}` | The SignalFx Smart Gateway's config file.  You can directly set additional configurations by specifying their path like this: `--set gateway.conf.<GATEAY CONFIG KEY>=`. |
| gateway.conf.ForwardTo | list | `[{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}]` | The SignalFx Smart Gateway forwarders.  This list is merged with the `forwarders` list. |
| gateway.conf.ForwardTo[0] | object | `{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}` | The default SignalFx forwarder |
| gateway.conf.ForwardTo[0].AuthTokenEnvVar | string | `"SFX_AUTH_TOKEN"` | The environment variable to read to configure the SignalFx Access Token |
| gateway.conf.ForwardTo[0].Name | string | `"signalfxforwarder"` | The name of the SignalFx forwarder |
| gateway.conf.ForwardTo[0].TraceSample | object | `{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"}` | Configurations for the SignalFx Smart Sampler |
| gateway.conf.ForwardTo[0].TraceSample.BackupLocation | string | `"/var/lib/gateway/data"` | The location inside the container to back up sampling data |
| gateway.conf.ForwardTo[0].TraceSample.ListenRebalanceAddress | string | `"0.0.0.0:2382"` | The address to listen on for cluster rebalancing |
| gateway.conf.ForwardTo[0].Type | string | `"signalfx"` | The type of the SignalFx forwarder |
| gateway.conf.InternalMetricsListenerAddress | string | `"0.0.0.0:2383"` | The address and port to serve metrics about the gateway |
| gateway.conf.ListenFrom | list | `[]` | The SignalFx Smart Gateway listeners.  This list is merged with the `listeners` list. |
| gateway.conf.LogDir | string | `"-"` | The directory to log to.  The SignalFx Smart Gateway will log to `stdout` when `gateway.conf.LogDir` is set to `-`. |
| gateway.containerPorts | list | `[{"containerPort":18080,"name":"sfx-listener","protocol":"TCP"},{"containerPort":2382,"name":"sfx-rebalance","protocol":"TCP"},{"containerPort":2383,"name":"sfx-internal-metrics","protocol":"TCP"}]` | A list of Kubernetes container port definitions to apply to the container |
| gateway.containerPorts[0].containerPort | int | `18080` | The SignalFx listener port |
| gateway.containerPorts[0].name | string | `"sfx-listener"` | The name of the SignalFx listener container port |
| gateway.containerPorts[0].protocol | string | `"TCP"` | The protocol of the SignalFx listener |
| gateway.containerPorts[1].containerPort | int | `2382` | The port of the SignalFx Forwarder's rebalance server |
| gateway.containerPorts[1].name | string | `"sfx-rebalance"` | The name of the SignalFx forwarder's rebalance port |
| gateway.containerPorts[1].protocol | string | `"TCP"` | The protocol used by the SignalFx Forwarder's rebalance server |
| gateway.containerPorts[2].containerPort | int | `2383` | The port of the internal metrics server |
| gateway.containerPorts[2].name | string | `"sfx-internal-metrics"` | The name of the internal metrics server |
| gateway.containerPorts[2].protocol | string | `"TCP"` | The protocol used by the internal metrics server |
| gateway.count | int | `1` | The number of SignalFx Smart Gateways to deploy |
| gateway.livenessProbe | object | `{"httpGet":{"path":"/healthz","port":"sfx-listener"},"initialDelaySeconds":15}` | A Kubernetes liveness probe to determine the health of the container |
| gateway.nodeSelector | object | `{}` | Kubernetes node selectors to apply to the pod |
| gateway.readinessProbe | object | `{}` | A Kubernetes readiness probe used to determine when the container is ready |
| gateway.resources | object | `{}` | Kubernetes deployment resources |
| gateway.tolerations | list | `[]` | Kubernetes deployment tolerations to apply to the pod |
| gateway.volumeClaimTemplates | list | `[]` | Volume claim templates to attach to replicas |
| gateway.volumeMounts | list | `[]` | Kubernetes volume mounts to apply to the container |
| gateway.volumes | list | `[]` | Kubernetes volumes to apply to the pod |
| image | object | `{"pullPolicy":"IfNotPresent","pullSecrets":null,"repository":"","tag":""}` | Configurations for the docker image to be deployed |
| image.pullPolicy | string | `"IfNotPresent"` | The policy for pulling the image |
| image.pullSecrets | list | `nil` | The list of image Kubernetes Image Pull Secrets to use when pulling images |
| image.repository | string | `""` | The image to pull (REQUIRED) |
| image.tag | string | `""` | The image tag to pull (REQUIRED) |
| ingress.enabled | bool | `false` |  |
| listeners | list | `[{"ListenAddr":"0.0.0.0:18080","Name":"signalfxlistener","RemoveSpanTags":[],"Type":"signalfx"}]` | SignalFx Smart Gateway listener config objects (These are merged into the `gateway.conf.ListenFrom` list) |
| listeners[0] | object | `{"ListenAddr":"0.0.0.0:18080","Name":"signalfxlistener","RemoveSpanTags":[],"Type":"signalfx"}` | The default SignalFx listener |
| listeners[0].ListenAddr | string | `"0.0.0.0:18080"` | The address to listen on for requests |
| listeners[0].Name | string | `"signalfxlistener"` | The name of the SignalFx listener |
| listeners[0].RemoveSpanTags | list | `[]` | The list of rules for excluding span tags |
| listeners[0].Type | string | `"signalfx"` | The type of the SignalFx listener |
| nameOverride | string | `""` | Overrides the name of the Helm chart |
| service | object | `{"ports":[{"nameSuffix":"sfx-listener","port":18080,"protocol":"TCP","targetPort":"sfx-listener"}],"type":"ClusterIP"}` | Configurations for the Kubernetes service exposing the SignalFx Smart Gateway's SignalFx listener |
| service.ports | list | `[{"nameSuffix":"sfx-listener","port":18080,"protocol":"TCP","targetPort":"sfx-listener"}]` | Ports to configure on the service |
| service.ports[0].nameSuffix | string | `"sfx-listener"` | A suffix to apply to the port name |
| service.ports[0].port | int | `18080` | The port to use on the service |
| service.ports[0].protocol | string | `"TCP"` | The protocol of the service |
| service.ports[0].targetPort | string | `"sfx-listener"` | The name of the container port to send requests to |
| service.type | string | `"ClusterIP"` | The service type |
| signalFxAccessToken | string | `""` | The SignalFx Access Token used to send Datapoints, Events, and Traces to SignalFx (REQUIRED) |
| targetClusterAddresses | list | `nil` | A list of etcd client addresses to connect to (REQUIRED) |

[a]: https://github.com/coreos/etcd-operator
[b]: https://github.com/helm/charts/tree/master/stable/etcd-operator
[c]: https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#downloading-a-specific-version-of-the-smart-gateway
[d]: https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#building-a-docker-image
[e]: https://github.com/signalfx/integrations/tree/master/gateway#listenfrom
[f]: https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#install-and-configure-the-smart-gateway
[g]: https://github.com/helm/helm
[h]: https://github.com/signalfx/smart-gateway-helm-chart/tree/master/helm/signalfx-smart-gateway
[i]: https://docs.signalfx.com/en/latest/integrations/agent/kubernetes-setup.html#observers
[values.yaml]: https://github.com/signalfx/smart-gateway-helm-chart/blob/master/helm/signalfx-smart-gateway/values.yaml
