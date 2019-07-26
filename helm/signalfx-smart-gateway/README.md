SignalFx Smart Gateway
======================

The Smart Gateway is a key component in your deployment of SignalFx Microservices APM: it receives all the distributed traces from your instrumented applications, generates metrics for each unique span and trace path, and selects the interesting, erroneous or outlier traces to forward to SignalFx. It is designed to run within your environment, close to your application, and operate reliably and at scale.


Current chart version is `0.1.0`

## Quick Start

This Helm chart deploys a SignalFx Smart Gateway cluster. It requires an existing etcd cluster
for the gateway nodes to connect to.

### Setup an etcd cluster

- Install the [etcd-operator](https://github.com/coreos/etcd-operator) Helm [chart](https://github.com/helm/charts/tree/master/stable/etcd-operator) and configure the chart to setup RBAC resources for the operator

```
$ helm install --name etcd-operator  --set customResources.createEtcdClusterCRD=true,customResources.createBackupCRD=true,customResources.createRestoreCRD=true stable/etcd-operator
```

- Update the chart to enable the cluster.  `cluster.enabled` is ignored on install.

```
$ helm upgrade --set cluster.enabled=true etcd-operator stable/etcd-operator
```

### Build the SignalFx Smart Gateway Image

Before installing the Helm chart, a SignalFx Smart Gateway image must be generated.

1. [Download the SignalFx Smart Gateway binary](https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#downloading-a-specific-version-of-the-smart-gateway)
2. [Use the documented dockerfile to build the image](https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#building-a-docker-image)
3. Place the image in a registry accessible to the Kubernetes cluster where the Helm chart will be installed.

### Install the Helm Chart

To use this chart with Helm, add our SignalFx Helm chart repository to Helm
like this:

```
$ helm repo add signalfx https://dl.signalfx.com/helm-repo
```

Then to ensure the latest state of the repository, run:

```
$ helm repo update
```

Then you can install the gateway using the chart name `signalfx/signalfx-smart-gateway`.

Be sure to set values for:
- signalFxAccessToken
- clusterName
- targetClusterAddresses
- image.tag (using `latest` is not recommended)

```
$ helm install signalfx/signalfx-smart-gateway \
--set image.tag=<YOUR_SMART_GATEWAY_TAG> \
--set signalFxAccessToken=<YOUR_ACCESS_TOKEN> \
--set clusterName=<YOUR_CLUSTER_NAME> \
--set targetClusterAddresses[0]=<YOUR_ETCD_CLUSTER_CLIENT_ADDRESS> \
--set distributor.count=3 \
--set gateway.count=3
```

A service will be created to forward requests to port 18080 on to the gateways' or the distributors' SignalFx Listener.

## About This Chart

This chart deploys a cluster of SignalFx Smart Gateways and optionally deploys a SignalFx Smart Gateway Distributor layer in front of the SignalFx Smart Gateways.  This chart will also create a service definition in front of the Smart Gateways or the Distributor layer.

### Important Configurations
Before proceeding, it is recommended that you look at the [values.yaml](./values.yaml) file included with the Helm chart.  There are two keys `gateway` and `distributor` each represents specific configurations about the SignalFx Smart Gateways and the SignalFx Smart Gateway Distributors.  

For convenience, there are a few top level configurations in the values.yaml to insert and configure Forwarders 
and Listeners for the Smart Gateway. 

#### ClusterName
The SignalFx Smart Gateway and SignalFx Smart Gateway Distributors must all use the same cluster name.  As a convenience
There is a top level configuration called `clusterName`.  It will be inserted into `gateway.conf.ClusterName` and
`distributor.conf.ClusterName`.  You can override this global configuration by directly setting gateway.conf.ClusterName
and `distributor.conf.ClusterName`.

#### Listeners
The `listeners` configuration is a list of SignalFx Smart Gateway Listener configuration JSON objects. These listeners 
will be merged into the values.yaml lists `gateway.conf.ListenFrom` and `distributor.conf.ForwardTo`. There is a default 
SignalFx Listener configuration stored in `listeners[0]` and it is configured to listen on port 18080. Please refer to 
the Listener [documentation](https://github.com/signalfx/integrations/tree/master/gateway#listenfrom) for more 
information about Listeners.

#### Forwarders
The `forwarders` configuration is a list of SiganlFx Smart Gateway Forwarders configuration JSON objects.  These
forwarders will be merged into the values.yaml list `gateway.conf.ForwardTo` and `distributor.conf.ForwardTo`.  Unlike
`listeners`, there are default SignalFx Forwarder configurations stored in `gateway.conf.ForwardTo[0]` and
`gateway.conf.ForwardTo[0]`.  This is because the forwarders are configured slightly differently between the
gateway and distributor forwarders.

#### Target Cluster Addresses
The `targetClusterAddresses` configuration is a list of etcd client addresses for the gateways and distributors to
connect to etcd.  This configuration is merged with the list `gateway.conf.TargetClusterAddresses` and 
`distributor.conf.TargetClusterAddresses`.

#### gateway.conf and distributor.conf
You'll notice that there are configurations called `gateway.conf` and `distributor.conf` they are JSON config objects representing
plain SignalFx Smart Gateway config.  They are passed in directly and so additional gateway/distributor configurations that are not
in the values.yaml can be set by directly specifying the path through these objects.  Please refer to the 
[SignalFx Smart Gateway Deployment Guide](https://docs.signalfx.com/en/latest/apm/apm-deployment/smart-gateway.html#install-and-configure-the-smart-gateway)
for more information about configurations for the gateway.  Please note that `listeners`, `forwarders`, and `targetClusterAddresses`
will be merged into their corresponding fields in `gateway.conf` and `distributor.conf`.



## Chart Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| clusterName | string | "" | the name of the cluster (REQUIRED) |
| distributor | object | {} | configurations for distributor nodes |
| distributor.advertisedSFXListenerAddress | string | "$(POD_IP):18080" |  |
| distributor.advertisedSFXRebalanceAddress | string | "$(POD_IP):2382" |  |
| distributor.affinity | object | {} | kubernetes affinity configuriations to apply to the pod |
| distributor.conf | object | {} |  |
| distributor.conf.ClusterOperation | string | "client" | the cluster operation the gateway should |
| distributor.conf.ForwardTo | list | [] | the gateway forwarders.  This list is merged with the forwarders list. |
| distributor.conf.ForwardTo[0] | object | {} | the SignalFx Forwarder |
| distributor.conf.ForwardTo[0].AuthTokenEnvVar | string | "SFX_AUTH_TOKEN" |  |
| distributor.conf.ForwardTo[0].BufferSize | int | 1000000 |  |
| distributor.conf.ForwardTo[0].DrainingThreads | int | 50 |  |
| distributor.conf.ForwardTo[0].MaxDrainSize | int | 5000 |  |
| distributor.conf.ForwardTo[0].Name | string | "signalfxforwarder" |  |
| distributor.conf.ForwardTo[0].TraceSample | object | {} |  |
| distributor.conf.ForwardTo[0].TraceSample.Distributor | bool | true |  |
| distributor.conf.ForwardTo[0].Type | string | "signalfx" |  |
| distributor.conf.ListenFrom | list | [] | the gateway listeners.  This list is merged with the listeners list. |
| distributor.conf.LogDir | string | "-" | the directory to log to.  A value of "-" will log to stdout |
| distributor.conf.TargetClusterAddresses | list | [] | a list of etcd client addresses to connect to |
| distributor.containerPorts | list | [] | a list of kubernetes container port definitions apply to the container |
| distributor.containerPorts[0] | object | {} |  |
| distributor.containerPorts[0].containerPort | int | 18080 | the container port of the SignalFx Listener |
| distributor.containerPorts[0].name | string | "sfx-listener" | the name of the SignalFx Listener containerPort |
| distributor.containerPorts[0].protocol | string | "TCP" | the protocol of the SignalFx Listener |
| distributor.containerPorts[1] | object | {} |  |
| distributor.containerPorts[1].containerPort | int | 2382 | the container port of the SignalFx Forwarder's rebalance server |
| distributor.containerPorts[1].name | string | "sfx-rebalance" | the name of the SignalFx Forwarder's rebalance containerPort |
| distributor.containerPorts[1].protocol | string | "TCP" | the protocol of the SignalFx Forwarder's rebalance server |
| distributor.count | int | 0 | number of gateway pods to deploy |
| distributor.livenessProbe | object | {} | a kubernetes liveness probe to determine the health of the container |
| distributor.livenessProbe.httpGet | object | {} |  |
| distributor.livenessProbe.httpGet.path | string | "/healthz" |  |
| distributor.livenessProbe.httpGet.port | string | "sfx-listener" |  |
| distributor.livenessProbe.initialDelaySeconds | int | 15 |  |
| distributor.nodeSelector | object | {} | kubernetes node selectors to apply to the pod |
| distributor.readinessProbe | object | {} | a kubernetes readiness probe to determine when the container is ready |
| distributor.resources | object | {} | kubernetes deployment resources |
| distributor.tolerations | list | [] | kubernetes deployment tolerations to apply to the pod |
| distributor.volumeMounts | list | [] | kubernetes volume mounts to apply to the container |
| distributor.volumes | list | [] | kubernetes volumes to apply to the pod |
| forwarders | list | [] | additional SignalFx Gateway forwarder config objects.  These will be merged into the gateway.conf.ForwardTo and distributor.conf.ForwardTo lists. |
| fullnameOverride | string | "" | overrides the full name of the helm chart |
| gateway | object | {} | configurations for gateway nodes |
| gateway.advertisedSFXListenerAddress | string | "$(POD_IP):18080" |  |
| gateway.advertisedSFXRebalanceAddress | string | "$(POD_IP):2382" |  |
| gateway.affinity | object | {} | kubernetes affinity configuriations to apply to the pod |
| gateway.conf | object | {} |  |
| gateway.conf.ClusterOperation | string | "client" | the cluster operation the gateway should |
| gateway.conf.ForwardTo | list | [] | the gateway forwarders.  This list is merged with the forwarders list. |
| gateway.conf.ForwardTo[0] | object | {} |  |
| gateway.conf.ForwardTo[0].AuthTokenEnvVar | string | "SFX_AUTH_TOKEN" |  |
| gateway.conf.ForwardTo[0].BufferSize | int | 1000000 |  |
| gateway.conf.ForwardTo[0].DrainingThreads | int | 50 |  |
| gateway.conf.ForwardTo[0].MaxDrainSize | int | 5000 |  |
| gateway.conf.ForwardTo[0].Name | string | "signalfxforwarder" |  |
| gateway.conf.ForwardTo[0].TraceSample | object | {} |  |
| gateway.conf.ForwardTo[0].TraceSample.ListenRebalanceAddress | string | "0.0.0.0:2382" |  |
| gateway.conf.ForwardTo[0].Type | string | "signalfx" |  |
| gateway.conf.ListenFrom | list | [] | the gateway listeners.  This list is merged with the listeners list. |
| gateway.conf.LogDir | string | "-" | the directory to log to.  A value of "-" will log to stdout |
| gateway.conf.TargetClusterAddresses | list | [] |  |
| gateway.containerPorts | list | [] | a list of kubernetes container port definitions apply to the container |
| gateway.containerPorts[0] | object | {} |  |
| gateway.containerPorts[0].containerPort | int | 18080 | the container port of the SignalFx Listener |
| gateway.containerPorts[0].name | string | "sfx-listener" | the name of the SignalFx Listener containerPort |
| gateway.containerPorts[0].protocol | string | "TCP" | the protocol of the SignalFx Listener |
| gateway.containerPorts[1] | object | {} |  |
| gateway.containerPorts[1].containerPort | int | 2382 | the container port of the SignalFx Forwarder's rebalance server |
| gateway.containerPorts[1].name | string | "sfx-rebalance" | the name of the SignalFx Forwarder's rebalance containerPort |
| gateway.containerPorts[1].protocol | string | "TCP" | the protocol of the SignalFx Forwarder's rebalance server |
| gateway.count | int | 1 | number of gateway pods to deploy |
| gateway.livenessProbe | object | {} | a kubernetes liveness probe to determine the health of the container |
| gateway.livenessProbe.httpGet | object | {} |  |
| gateway.livenessProbe.httpGet.path | string | "/healthz" |  |
| gateway.livenessProbe.httpGet.port | string | "sfx-listener" |  |
| gateway.livenessProbe.initialDelaySeconds | int | 15 |  |
| gateway.nodeSelector | object | {} | kubernetes node selectors to apply to the pod |
| gateway.readinessProbe | object | {} | a kubernetes readiness probe to determine when the container is ready |
| gateway.resources | object | {} | kubernetes deployment resources |
| gateway.tolerations | list | [] | kubernetes deployment tolerations to apply to the pod |
| gateway.volumeMounts | list | [] | kubernetes volume mounts to apply to the container |
| gateway.volumes | list | [] | kubernetes volumes to apply to the pod |
| image | object | {} | configurations for the deployed docker image |
| image.pullPolicy | string | "IfNotPresent" | defines when to pull the image |
| image.pullSecrets | list | [] | the image pull secrets of kubernetes secrets to use when pulling images  (REQUIRED) |
| image.pullSecrets[0] | object | {} |  |
| image.pullSecrets[0].name | string | "" |  |
| image.repository | string | "signalfx-smart-gateway" | the container image to pull |
| image.tag | string | "" | the container image tag to pull (REQUIRED) |
| ingress | object | {} |  |
| ingress.enabled | bool | false |  |
| listeners | list | [] | SignalFx Gateway listener config objects.  These will be merged into the gateway.conf.ListenFrom and distributor.conf.ListenFrom lists. |
| listeners[0] | object | {} |  |
| listeners[0].ListenAddr | string | "0.0.0.0:18080" |  |
| listeners[0].Name | string | "signalfxlistener" |  |
| listeners[0].RemoveSpanTags | list | [] |  |
| listeners[0].RemoveSpanTags[0] | object | {} |  |
| listeners[0].RemoveSpanTags[0].Service | string | "auth*" |  |
| listeners[0].RemoveSpanTags[0].Tags | list | [] |  |
| listeners[0].RemoveSpanTags[0].Tags[0] | string | "password" |  |
| listeners[0].Type | string | "signalfx" |  |
| nameOverride | string | "" | overrides the name of the helm chart |
| service | object | {} | configurations for the service exposeing the gateway's SignalFx listener |
| service.ports | list | [] | the ports to configure on the service |
| service.ports[0] | object | {} |  |
| service.ports[0].nameSuffix | string | "sfx-listener" | a suffix to apply to the port name |
| service.ports[0].port | int | 18080 | the port to use on the service |
| service.ports[0].protocol | string | "TCP" | the protocol of the service |
| service.ports[0].targetPort | string | "sfx-listener" | the name of the container port to send requests to |
| service.type | string | "ClusterIP" | the type of service |
| signalFxAccessToken | string | "" | access token for SignalFx.  (REQUIRED) |
| targetClusterAddresses | list | [] | a list of etcd client addresses to connect to (REQUIRED) |
| targetClusterAddresses[0] | string | "etcd-cluster-client:2379" |  |