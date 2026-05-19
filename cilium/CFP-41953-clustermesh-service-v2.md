# CFP-41953: ClusterMesh Service v2

**SIG: SIG-clustermesh**

**Begin Design Discussion:** 2025-10-01

**Cilium Release:** 1.20

**Authors:** Arthur Outhenin-Chalandre <git@mrfreezeex.fr>

**Status:** Implementable

## Summary

This CFP proposes introducing v2 of the clustermesh global service data
format stored in etcd. It transitions from `cilium/state/services/v1/`
to `cilium/state/endpointslices/v1/` and harmonizes backend data insertion
techniques between clustermesh services and Kubernetes services.

## Motivation

The current clustermesh global service data is handled with the
[`ClusterService` struct](https://github.com/cilium/cilium/blob/d83cf8ab5e20f8ef6031d9e0f66f577cd095ef89/pkg/clustermesh/store/store.go#L52).
This struct is encoded in JSON format and stored in etcd. While this format
has served well initially, it now faces several limitations that prevent
clustermesh from scaling efficiently and supporting new features. These
limitations fall into three main areas: missing backend conditions, suboptimal
performance for service updates, and inefficient data format and encoding.

### Missing backend conditions

The current format omits all backend conditions. It directly removes
backends that are not ready and not serving. This means that we cannot
properly perform graceful termination as described in the
[KPR documentation](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#graceful-termination),
most likely resulting in some traffic loss during rolling updates.

### Performance gap with the loadbalancer k8s reflector

There is a large performance gap between clustermesh and the
standard loadbalancer k8s reflector. Their update and ingestion behavior
is not currently on par. The table below shows how large this gap is:

| Backends | clustermesh (µs) | clustermesh w/o JSON decoding (µs) | loadbalancer k8s (µs) | Ratio (clustermesh w/o JSON/k8s) |
|----------|------------------|------------------------------------|-----------------------|----------------------------------|
| 1        | 44               | 35                                 | 4                     | 9x slower                        |
| 100      | 629              | 271                                | 4                     | 68x slower                       |
| 1 000    | 6 349            | 2 626                              | 7                     | 375x slower                      |
| 5 000    | 36 861           | 15 831                             | 30                    | 528x slower                      |
| 10 000   | 78 810           | 44 289                             | 70                    | 633x slower                      |

Note that the standard k8s reflector does not include JSON unmarshalling,
which is why we compare it with the loadbalancer k8s reflector without JSON
decoding above. These benchmarks are also not strictly equivalent, but they
give a good idea of the performance gap, how much JSON decoding contributes
to it, and how clustermesh degrades as the number of backends grows.

### Inefficient data format and network traffic

The `ClusterService` struct/format was designed to fit the loadbalancer internals
when introduced in 2018. In 2026, after several iterations and refactors, those
two formats have diverged. For instance, one recent example was that
`ClusterService` encoded similar ports with different names as multiple entries,
while the loadbalancer backends encoded them as one entry with multiple port
names attached. This divergence led to [a bug](https://github.com/cilium/cilium/issues/45109)
with one port shadowing others, which was recently fixed and will be released
in 1.19.4 / 1.18.10.

In terms of wire size, the backend map duplicates port information for each IP,
so the total size of the object tends to grow almost in proportion to the number
of ports and IPs. In addition, JSON encoding adds extra overhead
for field names and number formatting compared to a binary representation.

All of this data must be replicated to every node in the mesh. A mesh often
has many more nodes than a single cluster, which results in a high volume
of control plane network traffic.

Additionally, etcd imposes a hard limit of 1.5 MiB per object. Without a
breaking change to a more efficient format, adding the missing fields to
the current `ClusterService` struct would restrict scalability to fewer
than 10,000 backends per service per cluster. While this is a high limit,
having headroom beyond this point is useful for future growth.

Even below the limit, keeping objects small is important to reduce network
traffic when backends change. This situation is similar to the Kubernetes
community's move from `Endpoints` to `EndpointSlice`, but the problem is
even stronger in Cluster Mesh because a mesh can contain many more nodes
than a single Kubernetes cluster.

For example, the current `ClusterService` format for a 5 000 backends service
and 2 ports is 786.89 KiB. In an 11 clusters mesh where each cluster has 1 000
nodes, any update from any endpoint in this single service would result in about
7.5 GiB of data propagated globally across the mesh. From the perspective of a
single cluster control plane, this corresponds to about 750 MiB per update.
Assuming 10 updates per second for convenience, this would result in 7.5 GiB/s
of control plane traffic within that single cluster!

## Goals

* Reduce network bandwidth needed for control plane operations on large services
* Improve clustermesh service ingestion performance in the agent and in particular
  related to service churn scenarios
* Allow scaling on the clustermesh level to a larger number of backends per
  service per cluster
* Add backend conditions to clustermesh services to allow correct backend
  state handling in the loadbalancer
* Add the source `EndpointSlice` name to the clustermesh service data to simplify
  the EndpointSliceSync logic

## Non-Goals

* Changes not specific to clustermesh global services (for example
  MCS-API handling)
* Large changes to non-clustermesh loadbalancer logic

## Proposal

### Overview

This CFP proposes replacing the single `ClusterService` object, which currently
contains all backends for a global service, with individual
`ClusterEndpointSlice` objects. These objects are the Cluster Mesh equivalent
of Kubernetes `EndpointSlice` objects. We are not using any Service level fields
from the existing `ClusterService` struct, so we do not currently need to
maintain a Service level object. If future features require one, we may
reintroduce a Service level object, but it would most likely contain different
fields and no backends field or equivalent.

This will allow more code reuse between the Kubernetes reflector in the
loadbalancer packages, which already ingests individual Kubernetes
`EndpointSlice` objects, and the clustermesh code path.

The new format will include every relevant field from the `EndpointSlice`
struct, including endpoint conditions. The clustermesh code will preserve these
conditions and let the existing loadbalancer code apply the same backend state
handling as it does for local services, instead of only receiving backends in
an active state.

This will also significantly simplify the EndpointSliceSync codebase because it
can mirror remote `ClusterEndpointSlice` objects into local `EndpointSlice`
objects without the complex sharding logic. The inclusion of the conditions
should also benefit consumers watching those `EndpointSlice` objects, for
instance third-party GW-API implementations.

### Using `EndpointSlice` fields directly

The Kubernetes `EndpointSlice` API must remain backward compatible across
Kubernetes versions. This aligns well with Cilium Cluster Mesh's upgrade
requirement to support at least two consecutive minor versions.

We will directly embed various `EndpointSlice` fields into a new
`ClusterEndpointSlice` struct (the JSON and protobuf tags are omitted for
simplicity, but they will be added in the actual implementation):

```go
type ClusterEndpointSlice struct {
	Cluster string
	ClusterID uint32

	Namespace string
	Name string

	Labels map[string]string
	Annotations map[string]string

	AddressType slim_discovery_v1.AddressType
	Endpoints []slim_discovery_v1.Endpoint
	Ports []slim_discovery_v1.EndpointPort
}
```

KCM can update all `EndpointSlice` objects up to 20 times per second, with
bursts up to 30 updates per second. Kubernetes scalability tests also test
Services, and in those tests, despite significantly boosting KCM QPS to 100 or
500, `EndpointSlice` updates remain around 10 or fewer updates per second. At
very large scale (5k nodes), they can go up to ~45 updates per second in those
same tests.

This QPS is manageable for clustermesh despite the likely increase from managing
individual `ClusterEndpointSlice` objects and adding endpoint conditions. We
currently only have 20 QPS for clustermesh-apiserver, but we could likely boost
it to at least 50 QPS, similarly to how the Cilium Operator Kubernetes client
QPS was raised from 10 QPS to 100 QPS in 2024 in
[cilium/cilium#33821](https://github.com/cilium/cilium/pull/33821). In practice,
kvstoremesh already has a 100 QPS limit, so it would remain the shared upper
bound for events from all remote clusters. If there is more QPS across all
clusters and object types, each will essentially compete for that QPS budget.

### Unifying datapath ingestion pipelines

The clustermesh and loadbalancer k8s reflector currently diverge in both
their data structures and ingestion logic. This divergence has created the
performance gap described in the Motivation section and makes it harder to
maintain performance (and feature) parity between the two code paths.

As the new format mirrors the Kubernetes `EndpointSlice` model, we should be
able to unify both code paths and allow the Kubernetes reflector to receive
events from remote clusters.

Given the 10-600x ingestion gap shown in the Motivation benchmarks, the added
churn from handling individual `ClusterEndpointSlice` objects and conditions
should remain manageable while still making this path significantly more
performant than the current clustermesh v1 path.

### ClusterEndpointSlice encoding

We have conducted preliminary benchmarks comparing JSON, CBOR, protobuf, and
zstd compression for a full Kubernetes `EndpointSlice` with 100 endpoints as a
representative `ClusterEndpointSlice` payload. These benchmarks measure both
size and decoding speed. The size benchmarks are relevant for minimizing control
plane network bandwidth, while the decoding speed benchmarks measure the CPU
impact on Cilium agents across Cluster Mesh nodes.

JSON is what we currently use for all clustermesh data. Protobuf is used for
Kubernetes native types both at rest (in etcd) and to communicate with clients.
CBOR is a schemaless format, also used to some extent in Kubernetes, and is
essentially a more optimized alternative to JSON but is not human-readable.

Size:

| Format   | Raw      | zstd  |
| -------- | -------- | ----- |
| JSON     | 10.59KiB | 435B  |
| CBOR     |  7.42KiB | 404B  |
| Protobuf |  2.88KiB | 283B  |

Decode speed:

| Format   | Raw      | zstd  |
| -------- | -------- | ----- |
| JSON     | 420µs    | 449µs |
| CBOR     | 222µs    | 252µs |
| Protobuf |  62µs    |  74µs |

Based on these preliminary benchmarks, we will use zstd-compressed protobuf for
the new `ClusterEndpointSlice` format.

Compared to the JSON baseline currently used by Cluster Mesh, raw protobuf
is ~3.7x smaller and ~6.8x faster to decode. While using protobuf aligns with
the Kubernetes baseline for native types, we are also adding zstd compression.
Zstd makes each object another ~10x smaller, for a total of ~38x smaller than
the raw JSON baseline. It adds ~19% decoding overhead compared to raw protobuf,
but zstd-compressed protobuf still decodes ~5.7x faster than raw JSON.
The bottom line is that it should significantly reduce bandwidth and Cilium
agent CPU usage while ingesting remote Service data compared to what we have
today, even before accounting for the benefit from adopting slices.

The protobuf schema will be generated from the Go struct using
`k8s.io/code-generator/cmd/go-to-protobuf`. This process is already used to
generate the protobuf schema for the slim Kubernetes types, so this will only
require a minimal Makefile change.

The `kvstore/list` and `kvstore/update` shell commands will also support
converting this data to and from JSON to ease testing and debugging. As with
Kubernetes types, the Go struct will have JSON tags, so we can easily marshal
and unmarshal the data to and from JSON.

### Rollout strategy

As the Cluster Mesh area is currently maintained by a small team, we are aiming
for a straightforward rollout strategy that minimizes code changes.
The end goal is to not break the one minor version skew guarantee
with default settings during the transition.

The initial choice, which may vary during implementation, is to use a new
configuration option `clustermesh-service-v2`, with the following possible
options, which will evolve over minor releases:
- `prefer-legacy` (default in 1.20): Export `ClusterEndpointSlice` while still
  exporting and consuming `ClusterService`.
- `prefer-endpointslice` (default in 1.21): Export and consume
  `ClusterEndpointSlice` while still exporting `ClusterService` for backwards
  compatibility.
- `only-endpointslice`: Only export and consume `ClusterEndpointSlice` and stop
  exporting `ClusterService`.

With that initial choice, we would remove all `ClusterService` related code and
the `clustermesh-service-v2` option in Cilium 1.22, while allowing more advanced
users to either speed up or slow down the transition by changing the default
option in earlier releases.

## Impacts / Key Questions

### Key Questions: Chunking multiple `ClusterEndpointSlice` objects vs a single `ClusterService` object

The question of grouping the slices was quite debated. The existing
`ClusterService` format groups all backends from a Service in a single object.
Ultimately, we favored individual `ClusterEndpointSlice` objects because:
- KCM `EndpointSlice` QPS is manageable in our case
- It matches the Kubernetes model and is simple to reason about
- Kubernetes may provide alternative Service APIs in the future, and not relying
  explicitly on Services may help future-proof the format
- Handling individual `ClusterEndpointSlice` updates should be natural and
  simple for the loadbalancer and EndpointSliceSync code

### Key Questions: Encoding format

The encoding question was also quite debated. The existing `ClusterService`
format uses JSON, Kubernetes uses protobuf, and CBOR was also considered.
Ultimately, we chose the most efficient format that we considered:
zstd-compressed protobuf. See [ClusterEndpointSlice encoding](#clusterendpointslice-encoding)
for more details.
