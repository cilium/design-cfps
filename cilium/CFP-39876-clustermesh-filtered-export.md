# CFP-39876: Scoped Export Mode for ClusterMesh

**SIG:** SIG-NAME

**Sharing:** Public

**Begin Design Discussion:** 2025-06-02

**End Design Discussion:**

**Cilium Release:** 1.19

**Authors:** Krunal Jain <krunaljain@microsoft.com>, Vamsi Kalapala <Krishna.Vamsi@microsoft.com>

## Summary

This CFP introduces selective distribution of CiliumEndpoint/CiliumEndpointSlice and CiliumIdentity information via Cilium ClusterMesh by adding a new “scoped-export” mode for the ClusterMesh API server, which restricts cross-cluster propagation only to resources that reside in a set of allowlisted namespaces, also known as _global namespaces_. Users can mark a namespace as global through a dedicated annotation applied to the namespace itself. This proposal targets improved scalability while acknowledging that cross-cluster endpoint access and network policy enforcement is not supported for pods outside global namespaces.

## Motivation

The existing Cilium Clustermesh implementation distributes all CiliumIdentities and CiliumEndpoints across connected clusters, creating scaling constraints as resource volumes grow. This comprehensive propagation model becomes a limiting factor when managing extensive endpoint and identity collections. Performance profiling analysis demonstrates the severity of this scaling challenge: in a 200-cluster Clustermesh deployment with the following configuration parameters, each Cilium agent consumes 15GiB of memory:

- clusters=200
- nodes/cluster=300
- identities/cluster=500
- endpoints/cluster=15000
- services=0

By restricting export to only resources under global namespaces, we can achieve substantial scalability improvements and operational efficiency gains. An important consideration for using this new "scoped-export" mode is that all resources under the global namespaces are exported to clusters backend. This implies that the memory and compute optimizations might be suboptimal if the cluster has most of the resources under the global namespaces. Without the proposed optimizations, it wouldn't be possible to mesh such clusters together.

## Goals

- Enhance Cilium Clustermesh scalability through selective endpoint and identity distribution
- Full network policy support for pods inside global namespaces
- Endpoint to Endpoint connectivity for resources under global namespaces
- Global service functionality for resources under global namespaces
- Support scoped export for CRD mode Ciliumidentities

## Non-Goals

- Network policies for non global(local) namespaces
- Endpoint to Endpoint connectivity for non global(local) namespaces
- Global service functionality for non global(local) namespaces
- Scoped export for non CRD modes of identity allocation 
 
## Network Policy Support 
# Cross-Cluster Communication and Network Policy Behavior

Full support for **cross-cluster communication** and **network policy enforcement** is available **only when both the source (client) and destination (server) pods**—which may optionally be accessed via a service—**are part of a _global namespace_**.

### Behavior When Pods Are Not in Global Namespaces

If either the source or the destination pod does **not** belong to a global namespace, cross-cluster communication **may still succeed**, but under certain conditions although we do not support as part of the new mode. The evaluation of network policy for different cases is summarize below:


### Summary of Enforcement Logic

| Source Pod in Global NS | Destination Pod in Global NS | Network Policy Applies | Traffic Allowed? |
|-------------------------|-------------------------------|-------------------------|------------------|
| Yes                     | Yes                           | Yes                     | ✅ Enforced normally |
| No                      | Yes                           | Ingress Policy          | ❌ Works for tunnel mode. Does not work for native routing mode  |
| Yes                     | No                            | Egress Policy           | ❌ Not Supported |
| No                      | No                            | Any Policy              | ❌ Not Supported  |
| Any                     | Any                           | No Policies             | ✅ Allowed (no restrictions) |

## Global Service Support
In the scoped-export mode, all services within global namespaces would automatically be marked as "global". Users are not required to explicitly annotate individual services. Also, a service which is annotated as global but does not reside in a global namespaces would no longer be global.  

## MCS support
Similar to global services, MCS support is only available for services created under global namespaces. 

### Implementation details

The proposed scoped-export mode has a config that users can pass in to mark all non local annotated namespaces as global. 
- scoped-export-default-enable - true/false Honors only the local namespace annotation and marks all non annotated namespaces as global if set to true. Honors the global annotation and marks all non annotated  namespaces as local if set to true

#### Allow/Deny listing namespace
To allowlist a namespace and mark it as global, users can annotate the namespace with:

```
clustermesh.cilium.io/global: "true"
```

The ClusterMesh API server will watch namespace resources. Adding this annotation to any namespace will result in the following steps:

- A full synchronization is triggered for all resources within the newly annotated namespaces 
- Only entries associated with namespaces explicitly annotated as global will be exported.
- Entries in etcd that do not belong to any global (allowlisted) namespace will be deleted.
- All unannotated namespaces or entries with denied annotations are implicitly treated as non-global and excluded from export.

To explicitly denylist a namespace, users can annotate it with:

```
clustermesh.cilium.io/global: "false"
```

This explicitly marks all unannotated namespaces as global and marks the annotated services as local. Note that if both "true" and "false" annotations are present on different namespaces, the behaviour is to mark all unannotated namespaces as local(The "true" annotation has higher priority). Hence, to explicitly mark the unannotated services as global, users need to ensure that no namespace is explictly marked global in their cluster. Logically, since marking a namespace as global marks all other namespaces as local, users should not be using both the annotations for any cluster. To summarize behavior with deny annotation

- A full synchronization is triggered to mark all resources within the annotated namespace as global/local.
- Only entries associated with namespaces explicitly annotated with the value `false` will be skipped from the export provided no other namespace has the global annotation.
- Entries will be added in etcd for unannotated namespaces provided no other namespaces has the global annotation.
- All unannotated namespaces are exported unless no other namespace has the global annotation. 

When the annotation is removed or disabled for a namespace:
- A full synchronization is triggered to mark all resources within the annotated namespace as global/local.
- The system exports the CiliumEndpoints and CiliumIdentities with the updated list of global namespaces
- If no namespaces remain annotated post removal of the annotations, the behavior defaults to the existing behavior of the `clustermesh-apiserver`.

#### Network Policies 
An important part of this implementation is the functioning of network policies. In tunnel mode, the identitiy information is embedded in the network packet itself whereas in native routing mode, the identity information at the destination is derieved through the destination cluster's ipcache. Due to this descrepancy, if an identity is not exported and has associated network policies, the policies will be enforced in tunnel mode whereas they will not be honored in native mode if the identity is not exported since the identity is inferred as world at the destination due to the entry missing in the ipacache. As a result, for the network policies to work as expected, in the scoped export mode, network policies would only be honored for CiliumIdentities and CiliumEndpoints created under global namespaces.  Theoretically, the network policy descrepeancy would only occur for native routing mode but we intend to provide a consistent product expreience and communicate non support for all routing modes in non global(local) namespaces. We can provide updated documentation providing destination identity values for different routing modes but the network policy enforcement is only supported for workloads under scoped-export namespaces. 


#### MCS API 
We would not crete ServiceExport CRDs for local namespaces. This would inturn prevent the corresponding ServiceImport CRDs from being created in the remote clusters. Since we support transition to scoped-export mode without restarting the pods, it is possible that the CRDs might have been created before the users annotate their namespaces. We would not provide support for the functionality in local namespaces even if the CRDs exist.  


### ClusterMesh API Server Changes

#### Process annotations 
We will add new watch on namespaces to check for global or local annotations and maintain internal sets for the same. Once we have the list, we will trigger a full sync operation to update the export based on the provided annotation. 

#### Export Filtering
If the scoped export mode is enabled, only the CiliumIdentities and CiliumEndpoints under global namespaces will be exported to the remote clusters. 

#### ServiceExport controller 
Provide warning message when a ServiceExport is triggered for a service residing in local namespace 

#### Supported Resource Types
- CiliumEndpoints 
- CiliumIdentities 

### Alternative Approach1

#### Add IPtoSvc and IdentityToIP cache in Clustermesh APIserver 

Since clustermesh-apiserver already has watches setup on CiliumIdentities and CiliumEndpoints, we can maintain mappings of IPtoSvc and IdentityToIP caches which can provide answers to the following questions while processing each service event, ciliumendpoint event or ciliumidentityevent. To join ciliumidentity and ciliumendpoint information, we also need to maintain ipcache of ciliumendpoints  

- Is a CiliumEndpoint fronted by global service?
- Is a CiliumIdentity referenced by a ciliumendpoint fronted by global service?

The main challenge in this method is requirement to store all ciliumendpoints in memory 
 
 ### Alternative Approach2

#### Update CiliumEndpoint, CiliumIdentity and CiliumEndpointSlice controllers 
Update the CRD controllers to internally annotate CiliumEndpoints, CiliumIdentities and CiliumEndpointSlices associated with 
- Pods that are annotated by the user 
- Endpoints behind global services 

Once the annotations are added in the CRD controllers in Cilium Agent/Operator, the Clustermesh APIserver can filter out the non annotated CRDs and prevent them from being exported

### Alternate approaches 
A detailed breakdown of alternate approaches can be found in [this](https://docs.google.com/document/d/1GN5PYM40upeYpuriBZheSv2VWrAwA7WpBEqDb7EAv7w/edit?tab=t.0) document

## Future Milestones

### Pod level allowlisting 
The current implementation supports namespace grain allowlisting. In future, we would like to move to a more finer Pod level annotations

### Network policy support for non allowlisted namespaces 
Support network policies for non allowlisted namespaces