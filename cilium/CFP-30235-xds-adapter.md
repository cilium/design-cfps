# CFP-30235: xDS Adapter for ClusterIP Routing

**SIG:** Agent

**Begin Design Discussion:** 2023-01-03

**Cilium Release:** 1.16

**Authors:** Rob Scott <robertjscott@google.com>

## Summary

Add a new xDS adapter in Cilium that could take advantage of some of the
strengths of xDS, particularly the feedback loop via LRS and the overall
potential for scalability improvements when adjustments to routing
configurations don't need to round trip through the Kubernetes API Server.
This adapter would be an alternative source of endpoints and would not replace
the existing default behavior of reading directly from Kubernetes APIs.

## Motivation

There are some capabilities that the Kubernetes API Server is not well equipped
to handle. This includes topology aware routing (or any form of routing that
requires a feedback loop) and routing to endpoints outside of the cluster.

## Goals

* Describe how an xDS adapter could work in Cilium
* Highlight new capabilities that xDS could enable

## Non-Goals

* Build anything additional on top of this xDS adapter. Although this could be a
  foundational technology for several new features, those are out of scope for
  this proposal.

## Background

Several years ago, Matt Klein published a blog post charting a vision for xDS as
a [universal data plane
API](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a). Although
these APIs are primarily used by Envoy and gRPC today, there has been a broader
goal in the community to enable more widespread usage as part of a truly
universal data plane API. xDS is CNCF-governed and is gradually transitioning
content from the
[envoyproxy/data-plane-api](https://github.com/envoyproxy/data-plane-api) repo
to [cncf/xds](https://github.com/cncf/xds) repo as part of that overall vision.

In parallel to these efforts by the xDS community, GKE is planning to introduce
xDS as an additional data source for DPv2 configuration. This feels sufficiently
generic and helpful that it could be something that could be contributed to
upstream Cilium. This could be particularly useful for at least two common use
cases:

1. Supporting Services and Endpoints from outside of the local cluster.
1. Supporting advanced routing techniques, such as topology aware routing.

Although it is out of scope for this specific CFP to provide complete solutions
for either of these use cases, it will demonstrate the benefits of having an
xDS adapter when developing a solution either of these use cases.

### Use Cases

#### Topology Aware Routing

As the author of both EndpointSlices and the existing Topology Aware Routing
approach in Kubernetes, I feel like I’m well equipped to discuss the limitations
of both. What most people want from Topology Aware Routing is essentially to
fill the local zone to capacity and then spillover to the next closest set of
endpoints with capacity. Unfortunately that is very difficult to achieve with
the existing Kubernetes tooling. There are a few key problems here:

1. Kubernetes has no feedback loop to its dataplane and therefore no reliable
   way to understand if endpoints have reached capacity.
2. Even if there were a feedback loop, spillover to other zones benefits from
   some form of centralized orchestration to avoid a thundering herd problem.
3. All changes to endpoint routing currently have to go through Services or
   EndpointSlices, and by extension, the Kubernetes API Server. It would be very
   expensive to implement incremental weighting/spillover adjustments with that
   overhead. For example, adjusting the weight of any individual endpoint would
   require a write to the EndpointSlice API which would then need to be
   distributed to all consumers of that API. That can be problematic from a
   scalability perspective when there are frequent updates paired with a large
   number of endpoints and/or nodes.

I believe that xDS is uniquely positioned to address these limitations.
[LRS](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/load_stats/v3/lrs.proto)
(Load Reporting Service) provides a straightforward way to provide load reports
to a centralized control plane. Similarly, [Delta
xDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/xds_api#delta-endpoints)
enables xDS to distribute incremental updates that can be smaller in scope than
comparable Kubernetes APIs. All of this could be combined to avoid adding
additional load on the Kubernetes API Server.

#### Endpoints From Other Sources

In some cases, it’s useful to be able to route to endpoints from other sources,
such as other Kubernetes clusters or outside of Kubernetes altogether. This can
be particularly relevant when implementing Multi-Cluster Service routing.
Implementations today often rely on one of the following approaches:

1. Mirror endpoints from other clusters into Kubernetes by creating custom
   Services and EndpointSlices.
2. Introduce Gateways on the edges of clusters that can be used to route
   requests from other clusters.

With xDS support, it could be possible to have a centralized xDS control plane
that was familiar with endpoints in multiple sources, providing a potentially
more efficient alternative to either of the approaches commonly used today.

## Proposal

### Logic

The introduction of this adapter would mean that Cilium could accept endpoints
from more than one source. Cilium could be configured to read exclusively from
Kubernetes (default), xDS, or both, depending on the use case.

When both data sources were in use, precedence in any conflicts would be given
to data received directly from Kubernetes. As a concrete example, it's possible
that the same Service could be represented in both Kubernetes and xDS. If that
were the case, we'd give precedence to the Service from Kubernetes APIs and drop
the one from xDS, likely surfacing some kind of warning along the way.

### Architecture

This xDS adapter would be similar conceptually to KVStore. Essentially this
would be another backend that Cilium could get data from. Similar to how Cilium
can read from Kubernetes APIs and/or KVStore today, we would add a third option
for xDS as a data source, and develop a shared interface that worked across all
data sources.

### API Mapping

To keep the initial work as focused as possible, I propose using existing xDS
APIs to model ClusterIP routing. In the future, we may want to consider either
additions to those existing xDS APIs or new xDS APIs to cover the entirety of
Cilium functionality. To start, it seems highly valuable to focus on the areas
where existing xDS APIs overlap with existing Cilium capabilities.

This shows how we could use existing xDS APIs to represent Cilium capabilities
for ClusterIP routing:

| Cilium Field | Origin xDS | Comments |
| - | - | - |
| Frontend.Address | `filter_chain_match.prefix_ranges` | |
| Frontend.Port | `filter_chain_match.destination_port` | |
| Frontend.Protocol | `cluster.metadata.protocol` | |
| Backend[*].FEPortName | `cluster.metadata.port_name` | |
| Backend[*].NodeName | ? | TBD - This is only used for logic to determine if an endpoint is local. Instead of sending this through xDS for every endpoint we could just look at the CIDR configured for each Cilium instance. |
| Backend[*].Addr, Backend[*].Port | `lb_endpoints.endpoint.address.socket_address` | |
| Backend[*].State | `lb_endpoints.health_status` | |
| Backend[*].Preferred | N/A | Not used in Kubernetes |
| Backend[*].Weight | `lb_endpoints.endpoint.load_balancing_weight` | |
| Type | N/A | Out of scope - only focusing on ClusterIP Services |
| ExternalTrafficPolicy, InternalTrafficPolicy | `cluster.load_balancing_policy` | Defined in the typed struct `type.googleapis.com/cilium_io_traffic_policy` |
| NatPolicy | N/A | Not used in Kubernetes |
| SessionAffinity | `tcpproxy.hash_policy` | |
| SessionAffinityTimeout | ? | TBD - This represents the maximum session stickiness time. There is no natural place for this in existing xDS APIs. |
| HealthCheckNodePort | N/A | Out of scope - only focusing on ClusterIP Services |
| Name | `cluster.metadata.service_name` | |
| Namespace | `cluster.metadata.service_namespace` | |

## Impacts / Key Questions

_List crucial impacts and key questions. They likely require discussion and are required to understand the trade-offs of the CFP. During the lifecycle of a CFP, discussion on design aspects can be moved into this section. After reading through this section, it should be possible to understand any potentially negative or controversial impact of this CFP. It should also be possible to derive the key design questions: X vs Y._

### Impact: ... 1

_Describe crucial impacts and key questions that likely require discussion and debate._

### Key Question: ... 2

_Describe a key question_

### Option 1:

#### Pros

* ...

#### Cons

* ...

### Option 2:

#### Pros

* ...

#### Cons

* ...

## Future Milestones

_List things that this CFP will enable but that are out of scope for now. This can help understand the greater impact of a proposal without requiring to extend the scope of a CFP unnecessarily._

### Deferred Milestone 1

_Description of deferred milestone_

### Deferred Milestone 2
