---
reviewers:
- johnbelamaric
- imroc
title: Topology-aware traffic routing with topology keys
content_type: concept
weight: 150
---


<!-- overview -->

{{< feature-state for_k8s_version="v1.21" state="deprecated" >}}

{{< note >}}

This feature, specifically the alpha `topologyKeys` API, is deprecated since
Kubernetes v1.21.
[Topology Aware Hints](/docs/concepts/services-networking/topology-aware-hints/),
introduced in Kubernetes v1.21, provide similar functionality.

{{</ note >}}

_Service Topology_ enables a service to route traffic based upon the Node
topology of the cluster. For example, a service can specify that traffic be
preferentially routed to endpoints that are on the same Node as the client, or
in the same availability zone.


<!-- body -->

## Topology-aware traffic routing

By default, traffic sent to a `ClusterIP` or `NodePort` Service may be routed to
any backend address for the Service. Kubernetes 1.7 made it possible to
route "external" traffic to the Pods running on the same Node that received the
traffic. For `ClusterIP` Services, the equivalent same-node preference for
routing wasn't possible; nor could you configure your cluster to favor routing
to endpoints within the same zone.
By setting `topologyKeys` on a Service, you're able to define a policy for routing
traffic based upon the Node labels for the originating and destination Nodes.

The label matching between the source and destination lets you, as a cluster
operator, designate sets of Nodes that are "closer" and "farther" from one another.
You can define labels to represent whatever metric makes sense for your own
requirements.
In public clouds, for example, you might prefer to keep network traffic within the
same zone, because interzonal traffic has a cost associated with it (and intrazonal
traffic typically does not). Other common needs include being able to route traffic
to a local Pod managed by a DaemonSet, or directing traffic to Nodes connected to the
same top-of-rack switch for the lowest latency.

## Using Service Topology

If your cluster has the `ServiceTopology` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) enabled, you can control Service traffic
routing by specifying the `topologyKeys` field on the Service spec. This field
is a preference-order list of Node labels which will be used to sort endpoints
when accessing this Service. Traffic will be directed to a Node whose value for
the first label matches the originating Node's value for that label. If there is
no backend for the Service on a matching Node, then the second label will be
considered, and so forth, until no labels remain.

If no match is found, the traffic will be rejected, as if there were no
backends for the Service at all. That is, endpoints are chosen based on the first
topology key with available backends. If this field is specified and all entries
have no backends that match the topology of the client, the service has no
backends for that client and connections should fail. The special value `"*"` may
be used to mean "any topology". This catch-all value, if used, only makes sense
as the last value in the list.

If `topologyKeys` is not specified or empty, no topology constraints will be applied.

Consider a cluster with Nodes that are labeled with their hostname, zone name,
and region name. Then you can set the `topologyKeys` values of a service to direct
traffic as follows.

* Only to endpoints on the same node, failing if no endpoint exists on the node:
  `["kubernetes.io/hostname"]`.
* Preferentially to endpoints on the same node, falling back to endpoints in the
  same zone, followed by the same region, and failing otherwise: `["kubernetes.io/hostname",
  "topology.kubernetes.io/zone", "topology.kubernetes.io/region"]`.
  This may be useful, for example, in cases where data locality is critical.
* Preferentially to the same zone, but fallback on any available endpoint if
  none are available within this zone:
  `["topology.kubernetes.io/zone", "*"]`.



## Constraints

* Service topology is not compatible with `externalTrafficPolicy=Local`, and
  therefore a Service cannot use both of these features. It is possible to use
  both features in the same cluster on different Services, only not on the same
  Service.

* Valid topology keys are currently limited to `kubernetes.io/hostname`,
  `topology.kubernetes.io/zone`, and `topology.kubernetes.io/region`, but will
  be generalized to other node labels in the future.

* Topology keys must be valid label keys and at most 16 keys may be specified.

* The catch-all value, `"*"`, must be the last value in the topology keys, if
  it is used.


## Examples

The following are common examples of using the Service Topology feature.

### Only Node Local Endpoints

A Service that only routes to node local endpoints. If no endpoints exist on the node, traffic is dropped:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
```

### Prefer Node Local Endpoints

A Service that prefers node local Endpoints but falls back to cluster wide endpoints if node local endpoints do not exist:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "*"
```


### Only Zonal or Regional Endpoints

A Service that prefers zonal then regional endpoints. If no endpoints exist in either, traffic is dropped.


```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
```

### Prefer Node Local, Zonal, then Regional Endpoints

A Service that prefers node local, zonal, then regional endpoints but falls back to cluster wide endpoints.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
    - "*"
```




## {{% heading "whatsnext" %}}


* Read about [enabling Service Topology](/docs/tasks/administer-cluster/enabling-service-topology)
* Read [Connecting Applications with Services](/docs/tutorials/services/connect-applications-service/)

