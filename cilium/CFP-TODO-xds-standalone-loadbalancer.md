# CFP: xDS-controlled Standalone L4 LoadBalancer

**SIG:** SIG-DATAPATH

**Begin Design Discussion:** 2025-08-04

**Cilium Release:** 1.19

**Authors:** Tsotne Chakhvadze, <tsotne@google.com>

**Status:** Provisional

## Summary

This proposal introduces a powerful, API-driven way to manage **L4 load balancers** in Cilium using the standard xDS protocol. This allows users to dynamically configure L4 VIPs and backends from any xDS-compatible control plane, decoupling load balancing from Kubernetes Services for greater flexibility. This proposal is **exclusively focused on L4 load balancing** and does not include L7 features. This empowers platform builders and users with custom control planes to leverage Cilium's high-performance datapath more effectively for TCP/UDP services.

## Motivation

Cilium's original L4 load balancer (introduced in v1.10) was a powerful feature, but it had design limitations around management and persistence that led to its deprecation. However, the need for a robust, dynamically-configured L4 load balancer that is not tied to Kubernetes Services remains strong within the community. The exploration of a filesystem-based LB configuration ([cilium/cilium#39117](https://github.com/cilium/cilium/pull/39117)) is a clear indicator of this demand.

By integrating a native xDS client focused specifically on L4 constructs, we can provide a standard, industry-recognized API for this functionality. Previous initiatives introduced an experimental xDS client into Cilium; this proposal aims to build on that work to deliver a complete, stable, and well-supported feature for **L4 load balancing**. This aligns Cilium with modern, API-driven infrastructure practices and opens the door for deeper integration with a wide ecosystem of tools for service discovery and load balancing.

Furthermore, this provides a path to unify different L4 service discovery mechanisms in the future, such as the MCS (Multi-Cluster Services) service importer, leading to a more maintainable and streamlined codebase.

## Goals

*   Enable the programming of an **L4 LoadBalancer**, including a VIP and its backend endpoints, using xDS as the control plane signal.
*   Provide a stable, maintainable, and well-documented **L4 LB feature** that is decoupled from Kubernetes services.
*   Implement the feature in a way that is compatible with and can eventually support the MCS Service Importer, unifying L4 control paths.
*   Complete and stabilize the xDS client integration in Cilium for **L4 load-balancing use cases**.

## Non-Goals

*   This CFP is not proposing to build an xDS management server into Cilium. The xDS server is assumed to be an external component.
*   **Support for any L7 xDS features** (e.g., HTTP routing via RDS, traffic splitting) is out of scope. This proposal is **strictly for L4 load-balancing** (TCP/UDP).
*   This feature is not intended to replace Gateway API, GAMMA, or standard Kubernetes Service discovery, but rather to provide an alternative for standalone L4 LB use cases.

## Proposal

### Overview

The Cilium agent will be enhanced to run an xDS client that connects to an external xDS management server. This client will be responsible for fetching **L4 load-balancing configuration** and programming it into Cilium's datapath via StateDB.

The feature will be disabled by default and can be enabled via a new configuration flag in the Cilium Helm chart / ConfigMap.

### Configuration

A new configuration block will be added to the Cilium configuration:

```yaml
xds-control-plane:
  # Enables the xDS client in the Cilium agent for L4 LB
  enabled: false
  # Address of the external xDS management server
  server-address: "xds.my-company.com:9000"
  # Optional: Path to a client certificate for mTLS authentication
  client-cert-path: "/var/lib/cilium/tls/xds-client.crt"
  # Optional: Path to a client key for mTLS authentication
  client-key-path: "/var/lib/cilium/tls/xds-client.key"
  # Optional: Path to the CA certificate for verifying the xDS server
  ca-cert-path: "/var/lib/cilium/tls/xds-ca.crt"
```

### xDS Resource Mapping for L4 Load Balancing

The Cilium xDS client will subscribe to `Cluster` and `ClusterLoadAssignment` (CLA) resources to configure L4 load balancers. The agent will specifically look for `Cluster` resources containing Cilium-specific metadata to identify them as L4 LB services.

Here is a concrete example of how a user would define a simple TCP load balancer for a database service:

```yaml
# These resources would be served by an external xDS management server.

---
# Resource 1: The Cluster, defining the L4 service front-end (the VIP).
# The Cilium agent will watch for Clusters with the 'io.cilium.l4lb' metadata.
resource:
  '@type': type.googleapis.com/envoy.config.cluster.v3.Cluster
  name: my-database-service
  # This metadata block is the key part for Cilium. It defines the L4 LB properties.
  metadata:
    filter_metadata:
      io.cilium.l4lb:
        vip: "10.1.2.3"
        port: 3306
        protocol: TCP
  # Tells the xDS client to fetch endpoints via EDS for this cluster.
  type: EDS
  eds_cluster_config:
    service_name: my-database-service # Links this Cluster to its endpoints

---
# Resource 2: The ClusterLoadAssignment, defining the service backends.
resource:
  '@type': type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
  cluster_name: my-database-service # Must match the service_name/name above
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address: { address: 192.168.1.10, port_value: 3306 }
    - endpoint:
        address:
          socket_address: { address: 192.168.1.11, port_value: 3306 }
```

This information will be used to populate the `statedb.DB` with `tables.Services` and `tables.Backends`, which the datapath already uses for service translation.

### User Experience

The user experience will be analogous to the file-based L4 LB proposal, but far more powerful and scalable. Instead of managing a local JSON file on each node, users will manage a centralized set of standard `Cluster` and `ClusterLoadAssignment` resources in their xDS management server. This provides a centralized and dynamic control plane for managing potentially thousands of L4 services and backends across a fleet of Cilium-managed nodes.

## Impacts / Key Questions

### Impact: New Agent Component

This introduces a new, long-running gRPC client to the Cilium agent.
*   **Resource Usage**: This will add a baseline CPU and memory overhead to the agent for maintaining the gRPC connection and processing xDS updates.
*   **Complexity**: It adds a new component that needs to be maintained, including its connection logic, error handling, and security.

### Key Question: Connection Management and Security

*   How should the agent handle startup if the xDS server is unavailable? Should it fail, or start and retry in the background? (Proposal: Start and retry, to avoid blocking agent startup).
*   What is the strategy for securing the connection to the xDS server? (Proposal: Support mTLS, configurable via the agent settings).

### Key Question: xDS Schema and Mapping

*   The exact mapping from xDS `Cluster` and `ClusterLoadAssignment` fields to Cilium's service/backend model needs to be precisely defined. For example, how do we represent a VIP, which is not a standard field in the `Cluster` resource?
*   **Option 1: Use `metadata`:** We could use the `metadata` field on the `Cluster` resource to carry the VIP address and other Cilium-specific configuration. This is a common xDS pattern for extensibility and is flexible, but requires the user to structure the metadata correctly.
*   **Option 2: Define a custom xDS resource:** This is a much heavier lift and would require upstream xDS changes, so it is not preferred.

We will proceed with **Option 1** and clearly document the expected `metadata` structure, as shown in the example above.

## Future Milestones

### MCS Service Importer Integration

Once this feature is stable, the MCS (Multi-Cluster Services) service importer can be refactored to use this xDS client as its backend for L4 service discovery, rather than directly programming services itself. This would unify the standalone and multi-cluster L4 LB implementation paths, reducing complexity and improving maintainability.