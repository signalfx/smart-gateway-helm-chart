SignalFx Smart Gateway
======================

The Smart Gateway is a key component in your deployment of SignalFx
Microservices APM: it receives all the distributed traces from your instrumented
applications, generates metrics for each unique span and trace path, and selects
the interesting, erroneous or outlier traces to forward to SignalFx. It is
designed to run within your environment, close to your application, and operate
reliably and at scale.



Current chart version is `0.2.0`

## Quick Start

This Helm chart deploys a SignalFx Smart Gateway cluster. It requires an
existing etcd cluster for the gateway nodes to connect to.

### Setup an etcd Cluster

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

### Build the SignalFx Smart Gateway Image

Before installing the Helm chart, a SignalFx Smart Gateway image must be
generated.

1. [Download the SignalFx Smart Gateway binary][c]
2. [Use the documented dockerfile to build the image][d]
3. Place the image in a registry accessible to the Kubernetes cluster where the
Helm chart will be installed.

### Install the Helm Chart

To use this chart with Helm, add our SignalFx Helm chart repository to Helm
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

For a cluster, be sure to set values for:
- `signalFxAccessToken`
- `clusterName`
- `targetClusterAddresses`
- `image.repository`
- `image.tag`

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
template while installing the helm chart.

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
installing the helm chart.

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
[values.yaml] lists `gateway.conf.ListenFrom`.  There is a default SignalFx
Listener configuration stored in `listeners[0]` and it is configured to listen
on port `18080`. Please refer to the Listener [documentation][e] for more
information about Listeners.

#### Forwarders
The `forwarders` configuration is a list of SignalFx Smart Gateway Forwarders
configuration JSON objects. These forwarders will be merged into the
[values.yaml] list `gateway.conf.ForwardTo`.  Please note this difference in
configuration.  While the default SignalFx Listener is under `listeners[0]`,
but the default SignalFx Forwarder is under `gateway.conf.ForwardTo`.

#### Target Cluster Addresses
The `targetClusterAddresses` configuration is a list of etcd client addresses
for the gateways to connect to etcd. This configuration will be inserted into
`gateway.conf.TargetClusterAddresses`.

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
| clusterName | string | `""` | the name of the cluster (REQUIRED) |
| forwarders | list | `[]` | additional SignalFx Gateway forwarder config objects.  These will be merged into the gateway.conf.ForwardTo list. |
| fullnameOverride | string | `""` | overrides the full name of the helm chart |
| gateway | object | `{"advertisedSFXListenerAddress":"$(POD_IP):18080","advertisedSFXRebalanceAddress":"$(POD_IP):2382","affinity":{},"conf":{"ForwardTo":[{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}],"ListenFrom":[],"LogDir":"-"},"containerPorts":[{"containerPort":18080,"name":"sfx-listener","protocol":"TCP"},{"containerPort":2382,"name":"sfx-rebalance","protocol":"TCP"}],"count":1,"livenessProbe":{"httpGet":{"path":"/healthz","port":"sfx-listener"},"initialDelaySeconds":15},"nodeSelector":{},"readinessProbe":{},"resources":{},"tolerations":[],"volumeClaimTemplates":[],"volumeMounts":[],"volumes":[]}` | configurations for gateway nodes |
| gateway.advertisedSFXListenerAddress | string | `"$(POD_IP):18080"` | is an environment variable that will be set in the gateway pod. It is used to configure SignalFx Smart Samplers to advertise their configured SignalFx Listener in the cluster. By default it is set to the $(POD_IP):<PORT> where this config specifies the port, and the "$(POD_IP)" is filled in by kubernetes |
| gateway.advertisedSFXRebalanceAddress | string | `"$(POD_IP):2382"` | is an environment variable that will be set in the gateway pod.  It is used to configure the SignalFx Smart Samplers to advertise their configured RebalanceAddress in the cluister.  By defaul it is set to the $(POD_IP):<PORT> where this config specifies the port, and the "$(POD_IP)" is filled in by kubernetes |
| gateway.affinity | object | `{}` | kubernetes affinity configuriations to apply to the pod |
| gateway.conf | object | `{"ForwardTo":[{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}],"ListenFrom":[],"LogDir":"-"}` | represents the SignalFx Smart Gateway's config file.  You can directly set additional configurations `--set gateway.conf.<GATEAY CONFIG KEY>=` |
| gateway.conf.ForwardTo | list | `[{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}]` | the gateway forwarders.  This list is merged with the forwarders list. |
| gateway.conf.ForwardTo[0] | object | `{"AuthTokenEnvVar":"SFX_AUTH_TOKEN","Name":"signalfxforwarder","TraceSample":{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"},"Type":"signalfx"}` | the default SignalFx forwarder |
| gateway.conf.ForwardTo[0].AuthTokenEnvVar | string | `"SFX_AUTH_TOKEN"` | the environment variable to read for the SignalFx Access Token |
| gateway.conf.ForwardTo[0].Name | string | `"signalfxforwarder"` | the name of the SignalFx forwarder |
| gateway.conf.ForwardTo[0].TraceSample | object | `{"BackupLocation":"/var/lib/gateway/data","ListenRebalanceAddress":"0.0.0.0:2382"}` | configurations for the SignalFx Smart Sampler |
| gateway.conf.ForwardTo[0].TraceSample.BackupLocation | string | `"/var/lib/gateway/data"` | the location inside the container to back up sampling data |
| gateway.conf.ForwardTo[0].TraceSample.ListenRebalanceAddress | string | `"0.0.0.0:2382"` | the address to listen on for cluster rebalances |
| gateway.conf.ForwardTo[0].Type | string | `"signalfx"` | the type of the SignalFx forwarder |
| gateway.conf.ListenFrom | list | `[]` | the gateway listeners.  This list is merged with the listeners list. |
| gateway.conf.LogDir | string | `"-"` | the directory to log to.  A value of "-" will log to stdout |
| gateway.containerPorts | list | `[{"containerPort":18080,"name":"sfx-listener","protocol":"TCP"},{"containerPort":2382,"name":"sfx-rebalance","protocol":"TCP"}]` | a list of kubernetes container port definitions apply to the container |
| gateway.containerPorts[0].containerPort | int | `18080` | the container port of the SignalFx Listener |
| gateway.containerPorts[0].name | string | `"sfx-listener"` | the name of the SignalFx Listener containerPort |
| gateway.containerPorts[0].protocol | string | `"TCP"` | the protocol of the SignalFx Listener |
| gateway.containerPorts[1].containerPort | int | `2382` | the container port of the SignalFx Forwarder's rebalance server |
| gateway.containerPorts[1].name | string | `"sfx-rebalance"` | the name of the SignalFx Forwarder's rebalance containerPort |
| gateway.containerPorts[1].protocol | string | `"TCP"` | the protocol of the SignalFx Forwarder's rebalance server |
| gateway.count | int | `1` | number of gateway pods to deploy |
| gateway.livenessProbe | object | `{"httpGet":{"path":"/healthz","port":"sfx-listener"},"initialDelaySeconds":15}` | a kubernetes liveness probe to determine the health of the container |
| gateway.nodeSelector | object | `{}` | kubernetes node selectors to apply to the pod |
| gateway.readinessProbe | object | `{}` | a kubernetes readiness probe to determine when the container is ready |
| gateway.resources | object | `{}` | kubernetes deployment resources |
| gateway.tolerations | list | `[]` | kubernetes deployment tolerations to apply to the pod |
| gateway.volumeClaimTemplates | list | `[]` | volume claim templates to attach to replicas |
| gateway.volumeMounts | list | `[]` | kubernetes volume mounts to apply to the container |
| gateway.volumes | list | `[]` | kubernetes volumes to apply to the pod |
| image | object | `{"pullPolicy":"IfNotPresent","pullSecrets":[{"name":""}],"repository":"signalfx-smart-gateway","tag":""}` | configurations for the deployed docker image |
| image.pullPolicy | string | `"IfNotPresent"` | defines when to pull the image |
| image.pullSecrets | list | `[{"name":""}]` | the image pull secrets of kubernetes secrets to use when pulling images  (REQUIRED) |
| image.repository | string | `"signalfx-smart-gateway"` | the container image to pull |
| image.tag | string | `""` | the container image tag to pull (REQUIRED) |
| ingress.enabled | bool | `false` |  |
| listeners | list | `[{"ListenAddr":"0.0.0.0:18080","Name":"signalfxlistener","RemoveSpanTags":[],"Type":"signalfx"}]` | SignalFx Gateway listener config objects.  These will be merged into the gateway.conf.ListenFrom list. |
| listeners[0] | object | `{"ListenAddr":"0.0.0.0:18080","Name":"signalfxlistener","RemoveSpanTags":[],"Type":"signalfx"}` | The default SiganlFx listener |
| listeners[0].ListenAddr | string | `"0.0.0.0:18080"` | the address for the SignalFx Listener to listen on |
| listeners[0].Name | string | `"signalfxlistener"` | the name of the SignalFx Listener |
| listeners[0].RemoveSpanTags | list | `[]` | the list of span tags for the SignalFx Listener to remove |
| listeners[0].Type | string | `"signalfx"` | the type of the SignalFxListener |
| nameOverride | string | `""` | overrides the name of the helm chart |
| service | object | `{"ports":[{"nameSuffix":"sfx-listener","port":18080,"protocol":"TCP","targetPort":"sfx-listener"}],"type":"ClusterIP"}` | configurations for the service exposeing the gateway's SignalFx listener |
| service.ports | list | `[{"nameSuffix":"sfx-listener","port":18080,"protocol":"TCP","targetPort":"sfx-listener"}]` | the ports to configure on the service |
| service.ports[0].nameSuffix | string | `"sfx-listener"` | a suffix to apply to the port name |
| service.ports[0].port | int | `18080` | the port to use on the service |
| service.ports[0].protocol | string | `"TCP"` | the protocol of the service |
| service.ports[0].targetPort | string | `"sfx-listener"` | the name of the container port to send requests to |
| service.type | string | `"ClusterIP"` | the type of service |
| signalFxAccessToken | string | `""` | access token for SignalFx.  (REQUIRED) |
| targetClusterAddresses | list | `nil` | a list of etcd client addresses to connect to (REQUIRED) |

[a]: https://github.com/coreos/etcd-operator
[b]: https://github.com/helm/charts/tree/master/stable/etcd-operator
[c]: https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#downloading-a-specific-version-of-the-smart-gateway
[d]: https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#building-a-docker-image
[e]: https://github.com/signalfx/integrations/tree/master/gateway#listenfrom
[f]: https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#install-and-configure-the-smart-gateway
[values.yaml]: ./values.yaml
