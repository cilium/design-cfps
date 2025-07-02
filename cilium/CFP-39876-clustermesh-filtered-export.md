# CFP-39876: Scoped Export Mode for ClusterMesh

**SIG:** SIG-ClusterMesh

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
### Cross-Cluster Communication and Network Policy Behavior

Full support for **cross-cluster communication** and **network policy enforcement** is available **only when both the source (client) and destination (server) pods**—which may optionally be accessed via a service—**are part of a _global namespace_**.

### Behavior When Pods Are Not in Global Namespaces

If either the source or the destination pod does **not** belong to a global namespace, cross-cluster communication **may still succeed**, but under certain conditions although we do not support as part of the new mode. The evaluation of network policy for different cases is summarize below:


### Summary of Enforcement Logic

| Source Pod in Global NS | Destination Pod in Global NS | Network Policy Applies | Traffic Allowed? |
|-------------------------|-------------------------------|-------------------------|------------------|
| Yes                     | Yes                           | Yes                     | ✅ Enforced normally |
| No                      | Yes                           | Ingress Policy          | ❌ Not Supported  |
| Yes                     | No                            | Egress Policy           | ❌ Not Supported |
| No                      | No                            | Any Policy              | ❌ Not Supported  |
| Any                     | Any                           | No Policies             | ✅ Allowed (no restrictions) |

## Global Service Support
In the scoped-export mode, all services within global namespaces would automatically be marked as global/local depending on the provided annotation on the namespace. Users are not required to explicitly annotate individual services. Any service which is not inside a global namespace is considered local even if it has an associated global annotation. Such a service will not share backends with remote clusters in any case.  

## MCS support
Similar to global services, MCS support is only available for ServiceExport/ServiceImport CRDs created under global namespaces for both the local and remote clusters.  

## Implementation details

The proposed scoped-export mode has a config that users can pass in to mark all non local annotated namespaces as global. 
- clustermesh-default-global-namespace - true/false Honors only the local namespace annotation and marks all non annotated namespaces as global if set to true. Honors the global annotation and marks all non annotated  namespaces as local if set to true

#### Allow/Deny listing namespace
To allowlist a namespace and mark it as global, users can annotate the namespace with:

```
clustermesh.cilium.io/global: "true"
```

The ClusterMesh API server will watch namespace resources. Adding this annotation to any namespace will result in the following steps:

- A full synchronization is triggered for all resources within the newly annotated namespaces 
- Only entries associated with namespaces explicitly annotated as global will be exported.
- Unannotated namespaces are marked global/local depending on the value of config `clustermesh-default-global-namespace`

To explicitly mark a namespace as local, the annotation can be set as:

```
clustermesh.cilium.io/global: "false"
```

When the annotation is removed or disabled for a namespace:
- A full synchronization is triggered to mark all resources within the annotated namespace as global/local
- The system exports the CiliumEndpoints and CiliumIdentities with the updated list of global namespaces
- If no namespaces remain annotated post removal of the annotations, the behavior defaults to the existing behavior of the `clustermesh-apiserver`


#### Network Policies 
An important part of this implementation is the functioning of network policies. In tunnel mode, the identitiy information is embedded in the network packet itself whereas in native routing mode, the identity information at the destination is derieved through the destination cluster's ipcache. As a result, the network policies would not be enforced due to missing identity definition/security labels in native mode and security labels in the tunnel mode. 


#### MCS Support 
MCS-API ServiceExport and ServiceImport CRs will be only supported from a global Namespace. On a local namespace the following will be happening:
- ServiceExport created by a user will report a Condition to signal that the namespace is local and the Service is not actually exported to the clustermesh-apiserver.
- ServiceImport will still be created by the Cilium Operator but will also report a Condition to signal the user that the Service cannot be imported. Also the derived Service associated to the ServiceImport will not be created and any existing one will be deleted.

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