# CFP-39876: Namespace based Export Control for ClusterMesh

**SIG:** SIG-ClusterMesh

**Sharing:** Public

**Begin Design Discussion:** 2025-06-02

**End Design Discussion:**

**Cilium Release:** 1.19

**Authors:** Krunal Jain <krunaljain@microsoft.com>, Vamsi Kalapala <Krishna.Vamsi@microsoft.com>

**Status:** Implementable

## Summary

This CFP introduces selective distribution of CiliumEndpoint/CiliumEndpointSlice and CiliumIdentity information via Cilium ClusterMesh by adding a new export mechanism for the ClusterMesh API server, which restricts cross-cluster propagation only to resources that reside in a set of allowlisted namespaces, also known as _global namespaces_. Users can mark a namespace as global through a dedicated annotation applied to the namespace itself. This proposal targets improved scalability while acknowledging that cross-cluster endpoint access and network policy enforcement is only enabled for pods inside global namespaces.

## Motivation

The existing Cilium ClusterMesh implementation distributes all CiliumIdentities and CiliumEndpoints across connected clusters, creating scaling constraints as resource volumes grow. This comprehensive propagation model becomes a limiting factor when managing extensive endpoint and identity collections. Performance profiling analysis demonstrates the severity of this scaling challenge: in a 200-cluster ClusterMesh deployment with the following configuration parameters, each Cilium agent consumes 15GiB of memory:

- clusters=200
- nodes/cluster=300
- identities/cluster=500
- endpoints/cluster=15000
- services/cluster=0

By restricting export to only resources under global namespaces, we can achieve substantial scalability improvements and operational efficiency gains. An important consideration for using this new global/local namespace feature is that all resources under the global namespaces are exported to the remote clusters. This implies that the memory and compute optimizations might be suboptimal if the cluster has most of the resources under the global namespaces. Without the proposed optimizations, each clustermesh-apiserver replica would export the entire local state to remote clusters.  

## Goals

- Enhance Cilium ClusterMesh scalability through selective endpoint and identity distribution
- Full network policy support for pods inside global namespaces
- Cross-cluster pod to pod connectivity inside global namespaces
- Global service and MCS-API support for resources under global namespaces

## Non-Goals

- Network policies for non global(local) namespaces
- Cross-cluster pod to pod connectivity for non global(local) namespaces
- Global service and MCS-API support for non global(local) namespaces
- Support Global/Local namespaces for non CRD modes of identity allocation 
 
## Network Policy 
### Cross-Cluster Communication and Network Policy Behavior

Full support for **cross-cluster communication** and **network policy enforcement** is available **only when both the source (client) and destination (server) pods**—which may optionally be accessed via a service—**are part of a _global namespace_**. 

### Behavior When Pods Are Not in Global Namespaces
Network policies in Cilium rely heavily on **identity labels** for policy matching. Cross-cluster traffic from a source pod not part of a global namespace to a destination pod matched by an ingress policy is dropped, unless from-world traffic is explicitly allowed by the policy, in which case it may be allowed, depending on the specific Cilium configuration. Similarly, cross-cluster traffic from a source pod matched by an egress policy to a destination pod not part of a global namespace is dropped, unless to-world traffic is explicitly allowed by the policy (either globally, or via a specific CIDR/FQDN rule), in which case it is allowed.

#### Why Current Policies Can't Work

Cilium `NetworkPolicy` typically uses selectors like:

```yaml
endpointSelector:
  matchLabels:
    app: frontend
ingress:
  - fromEndpoints:
    - matchLabels:
        app: backend
        tier: api
```

Without labels, the policy engine cannot:
- Match fromEndpoints selectors
- Evaluate matchLabels conditions
- Distinguish between different application identities


When using this feature, cross-cluster communication requires both the source and destination namespaces to be global, and both clusters to have an established cluster mesh. The evaluation of network policy for different cases is summarized below:

### Summary of Enforcement Logic

| Source Pod in Global NS | Destination Pod in Global NS | Can you allow traffic based on Labels? |
|-------------------------|------------------------------|------------------|
| Yes                     | Yes                          | ✅ Enforced normally |
| No                      | Yes                          | ❌ Not Supported  |
| Yes                     | No                           | ❌ Not Supported |
| No                      | No                           | ❌ Not Supported  |

In the above table, Not Supported implies that policy enforcement will not take into account labels on Pods in local namespaces in a peer cluster. 

## Global Service Support
For a service to be considered a Global Service, it must satisfy the following conditions 
- It must be annotated with existing global service annotation ***service.cilium.io/global: "true"***
- It must reside within a global namespace 

## Implementation details
The proposed namespace based export mode has a config that users can pass in to mark all non annotated namespaces as global
- clustermesh-default-global-namespace (true/false) - Marks implicitly all non annotated namespaces as global if set to true. Marks implicitly all non annotated namespaces as local if set to false. This option is set to true by default to preserve the current behavior.

#### Allow/Deny listing namespace
To allowlist a namespace and mark it as global, users can annotate the namespace with:

```
clustermesh.cilium.io/global: "true"
```

The ClusterMesh API server will watch namespace resources. Adding, changing or removing this namespace annotation will result in the following steps:

- A full synchronization is triggered for all resources within the newly annotated namespaces 
- Only entries associated with namespaces explicitly annotated as global will be exported.
- Unannotated namespaces are treated as global/local depending on the value of config `clustermesh-default-global-namespace`

To explicitly mark a namespace as local, the annotation can be set as:

```
clustermesh.cilium.io/global: "false"
```

When the annotation is removed or disabled for a namespace:
- A full synchronization is triggered to add/remove the CiliumEndpoints and CiliumIdentities objects in etcd associated with the resources in the unannotated namespace
- The system exports the CiliumEndpoints and CiliumIdentities with the updated list of global namespaces

#### MCS Support 
MCS-API ServiceExport and ServiceImport CRs will be only supported from a global Namespace. On a local namespace the following will be happening:
- ServiceExport created by a user will report a Condition to signal that the namespace is local and the Service is not actually exported to the clustermesh-apiserver.
- ServiceImport will still be created by the Cilium Operator but will also report a Condition to signal the user that the Service cannot be imported. Also the derived Service associated to the ServiceImport will not be created and any existing one will be deleted.

### ClusterMesh API Server Changes

#### Process annotations 
We will add new watch on namespaces to check for global or local annotations and maintain internal sets for the same. Once we have the list, we will trigger a full sync operation to update the export based on the provided annotation. 

#### Export Filtering
Only the CiliumIdentities and CiliumEndpoints under global namespaces will be exported to the remote clusters. 

#### ServiceExport controller 
Provide warning message when a ServiceExport is triggered for a service residing in local namespace 

#### Existing Cilium Resource Types
- CiliumEndpoint(Slice)
- CiliumIdentities
- Global Services
- MCS ServiceExport

## Impacts / Key Questions
---

### Impact: Breakage during upgrade
- The changes are expected to be backward compatible in the sense that the new scoped export logic only takes effect once the user has explicitly annotated namespaces as global or local. As a result, existing clusters will continue to function as before until the user opts into the new behavior. The `clustermesh-default-global-namespace` config option will determine the default behavior for unannotated namespaces.
---

### Impact: Debugging multi-cluster environment connectivity may become more difficult
- Failures due to missing identity or label exports may be non-obvious and require deeper inspection into the global/local namespace configuration.
- Cross-cluster connectivity issues will now depend not only on service presence or routes but also on namespace-level annotations and default behaviors requiring additional verifications on the value of config `clustermesh-default-global-namespace` along with the actual annotations present on the namespaces during debugging.
- If an application is sending traffic to a destination in another cluster and the traffic is dropped due to policy (as seen in hubble observe), users should inspect whether the source and destination namespaces are marked as global.
---
### Alternative Approach 1

### Exporting CiliumEndpoint and CiliumIdentity for Global Services

By default, all endpoints fronted by **global services** are expected to have their `CiliumEndpoint` and `CiliumIdentity` information shared across clusters to support cross-cluster load balancing. If we can determine whether a given `CiliumEndpoint` or `CiliumIdentity` is fronted by a global service, we can use that as the basis for deciding whether to export it.

To enable clients of global services to communicate with their backend service endpoints, we also need a mechanism for users to explicitly mark certain pods as **clients** of global services. This can be achieved through a pod-level annotation.

Since the `clustermesh-apiserver` already watches `CiliumEndpoints` and `CiliumIdentities`, we can maintain in-memory caches to support export decisions:

- **`IPToService`**: a mapping from IP addresses to services that front them.
- **`IdentityToIP`**: a mapping from identities to the set of IPs they represent.
- **`IPToEndpoint`**: to associate IPs back to `CiliumEndpoints`.

These caches allow us to answer the following questions when processing service, endpoint, or identity events:

- Is a given `CiliumEndpoint` fronted by a global service?
- Is a given `CiliumIdentity` associated with any `CiliumEndpoint` that is fronted by a global service?

To join `CiliumIdentity` and `CiliumEndpoint` information, we rely on the IPCache, which maps IP addresses from endpoints to their identities and vice versa. This mapping can be achieved using a cache containing all CiliumEndpoints. 

In addition to exporting resources fronted by global services, we also propose exporting `CiliumEndpoints` and `CiliumIdentities` for pods explicitly annotated as **clients of global services**.


### Alternative Approach 2

#### Exporting CiliumEndpoint and CiliumIdentity associated with annotated pods
To support selective export of `CiliumEndpoints`, `CiliumIdentities`, and `CiliumEndpointSlices`, we propose updating the CRD controllers in the Cilium Agent or Operator to internally annotate resources associated with service endpoints or client pods of a global service. These are identified via a specific user-defined annotation that marks them as a resource that should be exported across cluster boundary.

#### Workflow

1. **User Annotation**: 
   - Users annotate certain pods with a predefined key (e.g., `clustermesh.cilium.io/global-service-client=true`) to indicate that the pod is a client/endpoint of one or more global services.

2. **CRD Controller Processing**:
   - The Cilium Agent and Operator monitors pod changes.
   - When a pod with the client annotation is detected, the associated `CiliumEndpoint`, `CiliumIdentity`, and `CiliumEndpointSlice` are automatically annotated internally.

3. **Purpose of Internal Annotations**:
   - These internal annotations allow downstream components, such as the `clustermesh-apiserver`, to easily identify and filter relevant resources during export.
   - This avoids the overhead of recomputing client-pod status or performing expensive label queries in the API server.

The main complexity of this approach lies in inferring global service endpoints internally along with annotated client pods. The client annotations are processed in the CiliumEndpoint, CiliumIdentity and CiliumEndpointSlice CRD controllers which would lead to changes in all the existing Cilium components (Agent, Operator and ClusterMesh APIserver). 

#### Export Filtering

Once these internal annotations are present:
- The `clustermesh-apiserver` can skip exporting any `CiliumEndpoint`, `CiliumIdentity`, or `CiliumEndpointSlice` that is **not annotated**.
- This significantly reduces the number of exported resources, improving scalability, security, and performance across clusters.

Similar to the above approach, this approach would require changes in the CiliumEndpoint, CiliumIdentity and CiliumEndpointSlice CRD controllers which would lead to changes in all the existing Cilium components (Agent, Operator and ClusterMesh APIserver). 

### Additional Details for Alternate approaches 
A detailed breakdown of alternate approaches can be found in [this](https://docs.google.com/document/d/1GN5PYM40upeYpuriBZheSv2VWrAwA7WpBEqDb7EAv7w/edit?tab=t.0) document

## Future Milestones

### Pod level allowlisting 
The current implementation enables users to do namespace grain allowlisting. In future, we would like to move to a more finer Pod level annotations
