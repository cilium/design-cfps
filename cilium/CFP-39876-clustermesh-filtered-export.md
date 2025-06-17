# CFP-39876: Scoped Export Mode for ClusterMesh

**SIG:** SIG-NAME

**Sharing:** Public

**Begin Design Discussion:** 2025-06-02

**End Design Discussion:**

**Cilium Release:** (TBD)

**Authors:** Krunal Jain <krunaljain@microsoft.com>, Vamsi Kalapala <Krishna.Vamsi@microsoft.com>

## Summary

This CFP introduces selective distribution of endpoints and identities within Cilium Clustermesh by introducing a new “scoped-export” mode for the Clustermesh API server, restricting cross-cluster propagation only to resources that are fronted by global services. The modification targets improved scalability while acknowledging that direct inter-cluster endpoint access and network policy enforcement will be discontinued for resources not associated with global services. The addition of new Clustermesh APIserver export mode would ensure the changes are backwards compatible.

## Motivation

The existing Cilium Clustermesh implementation distributes all endpoints and identities across connected clusters, creating scaling constraints as resource volumes grow. This comprehensive propagation model becomes a limiting factor when managing extensive endpoint and identity collections. Performance profiling analysis demonstrates the severity of this scaling challenge: in a 200-cluster Clustermesh deployment with the following configuration parameters, each Cilium agent consumes 15GiB of memory:

- cm-apiserver-replicas=2
- clusters=200
- nodes/cluster=300
- identities/cluster=500
- identities-qps=0.2
- endpoints/cluster=15000
- endpoints-qps=1
- services=0
- services-qps=0

By restricting distribution to only those resources backing global services, we can achieve substantial scalability improvements and operational efficiency gains. The gains in compute from the proposed optimization would also ensure that customers can run multiple replicas of the Clustermesh APIserver which is recommended for production workloads.

## Goals

- Enhance Cilium Clustermesh scalability through selective endpoint and identity distribution
- Maintain cross-cluster visibility for endpoints and identities associated with global services
- Service to Service connection and network policies

## Non-Goals

- Direct inter-cluster endpoint access for resources not backed by global services
- Cross-cluster network policy enforcement
- Pod to Pod or Pod to service connection or Network Policy

## Proposal

### Implementation details

We intend to add a new cache in conjunction to the Service cache that exists both in the cilium agent as well as the operator. We intend to provide a new config in the cilium agent configmap called scoped-export which is by default set to false. Upon setting this config, the agent and the operator will annotate the CiliumEndpoint, CiliumEndpointSlice and CiliumIdentity CRDs with internal annotation allowlisting them for export to etcd. A change in scope–export config would require restart of cilium agent and operator components for the changes to take effect.

### Global Service Backend Cache

#### Implementation Location

The global service backend cache is implemented in both the Cilium Agent and Operator and works in conjunction with the ServiceCache. The cache resides both in the agent and operator maintaining a synchronized copy of global service backend endpoint IPs.
The cache maps backend IP addresses to metadata including service name and namespace

#### Cache Management

The cache is populated via service endpoint updates and receives live updates when backends change. Stale entries are automatically cleaned up, and synchronization mechanisms ensure the global service endpoint cache contains the latest copy of backend endpoint ips. Both Cilium Agent and Cilium Operator already listen to existing Service Events. These events are consumed to populate the global service backend IP cache. 

### Cilium Agent Changes

#### Service Cache Extension

The agent's existing service cache is extended to support lookup for ips against global services.

#### CiliumEndpoint Reconciliation

Every 10 seconds, the agent checks each CiliumEndpoint IP against the global backend cache. If matched, it adds an internal annotation to mark it as a global backend; otherwise, it removes the annotation. This annotation is internal and not user-modifiable.

#### CiliumIdentity Reconciliation

When managing CiliumIdentities, the agent listens for global service IP cache events. On upserts or deletions, it reconciles the relevant identities and updates their annotations accordingly.

### Cilium Operator Changes

#### Service Cache Extension

The agent's existing service cache is extended to support lookup for ips against global services.

#### CiliumIdentity Handling

If the operator manages CiliumIdentities, it responds to global service events from its cache and performs reconciliation to add or remove internal annotation as needed.

#### CiliumEndpointSlice Controller Updates

The controller, which already watches CiliumEndpoint resources, is updated to check for the global backend annotation. If present, it adds a corresponding annotation to the CiliumEndpointSlice. Annotation changes are tracked and updated accordingly.

### ClusterMesh API Server Changes

#### Export Filtering

The ClusterMesh API server filters resources before exporting to etcd. Only those with the internal global backend annotation are shared for cross-cluster visibility.

#### Supported Resource Types

The filter applies to CiliumEndpoint, CiliumIdentity, and CiliumEndpointSlice. Only annotated resources are exported to be shared cross clusters.

## Future Milestones

### Dynamic Export Scope Configuration

Introduce support for dynamic configuration of export scope per namespace or pod label. This would allow operators to fine-tune which endpoints and identities are shared across clusters beyond just global service association.

### Policy-Aware Export Filtering

Integrate network policy awareness into the export decision logic. Endpoints and identities could be exported based on whether they are referenced in cross-cluster network policies, even if not fronted by a global service.
