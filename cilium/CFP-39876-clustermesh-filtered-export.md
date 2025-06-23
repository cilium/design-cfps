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

By restricting distribution to only those resources backing global services, we can achieve substantial scalability improvements and operational efficiency gains. The gains in compute from the proposed optimization would also ensure that customers can run multiple replicas of the Clustermesh APIserver which is recommended for production workloads. An important consideration for using this new "scoped-export" mode is that global service backends are exported to clusters backend. This implies that the memory and compute optimizations might be suboptimal if the cluster has most of the endpoints fronted by global services. 

## Goals

- Enhance Cilium Clustermesh scalability through selective endpoint and identity distribution
- Network policies only work for "allowlisted" namespaces
- Endpoint to Endpoint connectivity only work for "allowlisted" namespaces
- Global service functionality only work for "allowlisted" namespaces
- Support scoped export for CRD mode identities

## Non-Goals

- Network policies for non "allowlisted" namespaces
- Endpoint to Endpoint connectivity for "allowlisted" namespaces
- Global service functionality for "allowlisted" namespaces
- Scoped export for non CRD modes of identity allocation 
- 
## Proposal

### Implementation details

We intend to only export ciliumendpoints and ciliumidentities for "allowlisted" namespaces when the clustermesh-apiserver runs in "scoped-export" mode. This new mode has three configs which the clustermesh-apiserver imports from cilium configmap namely 
- scoped-export-enabled - Enable the scoped export mode 
- scoped-export-namespaces - List of namespaces that are "allowlisted for export". The ciliumendpoints and ciliumidentities inside the allowlisted namespaces are exported to remote clusters. 
- allow-by-default - Allowlisting method. If set to true, all namespaces are disabled for export until explicitly exported using the allowlisted-namespaces configs. If set to false,  ciliumendpoints and identities under all namespaces are enabled for export by default until mentioned under the scoped-export-namespaces. 

An important part of this implementation is the functioning of network policies. In tunnel mode, the identitiy information is embedded in the network packet itself whereas in native routing mode, the identity information at the destination is derieved through the destination cluster's ipcache. Due to this descrepancy, if an identity is not exported and has associated network policies, the policies will be enforced in tunnel mode whereas they will not be honored in native mode if the identity is not exported since the identity is inferred as world at the destination due to the entry missing in the ipacache. As a result, for the network policies to work as expected, in the scoped export mode, network policies would only be honored for identities and endpoints created under `scoped-export-namespaces`.  Theoretically, the network policy descrepeancy would only occur for native routing mode but we intend to provide a consistent product expreience and communicate non support for all routing modes in non allowlisted namespaces. We can provide updated documentation providing destination identity values for different routing modes but the network policy enforcement is only supported for workloads under scoped-export namespaces. 
Similarly for Global Service implementation, since only the clients under allowlisted namespaces are enabled for cross cluster export, all services under `scoped-export-namespaces` are global by default and customers cannot create global services under non allowlisted namespaces.  


### ClusterMesh API Server Changes

#### Export Filtering
If the scoped export mode is enabled, only the ciliumendpoints and ciliumidentities under the allowlisted namespaces will be exported to the remote clusters. 

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
- Pods that are annotated by the customer 
- Endpoints behind global services 

Once the annotations are added in the CRD controllers in Cilium Agent/Operator, the Clustermesh APIserver can filter out the non annotated CRDs and prevent them from being exported

### Alternate approaches 
A detailed breakdown of alternate approaches can be found in [this](https://docs.google.com/document/d/1GN5PYM40upeYpuriBZheSv2VWrAwA7WpBEqDb7EAv7w/edit?tab=t.0) document

## Future Milestones

### Pod level allowlisting 
The current implementation supports namespace grain allowlisting. In future, we would like to move to a more finer Pod level annotations

