# CFP-30984: toFQDN DNS proxy HA

**SIG: SIG-POLICY**

**Begin Design Discussion:** 2024-08-19

**Cilium Release:** 1.17

**Authors:** Hemanth Malla <hemanth.malla@datadoghq.com>, Vipul Singh <singhvipul@microsoft.com>, Tamilmani Manoharan <tamanoha@microsoft.com>

## Summary

Cilium agent uses a proxy to intercept all DNS queries and obtain necessary information to enforce FQDN network policies. However, the lifecycle of this proxy is coupled with the cilium agent. When an endpoint has a toFQDN network policy in place, cilium installs a redirect to capture all DNS traffic. So, when the agent is unavailable, all DNS requests time out, including when DNS name to IP address mappings are already in place for this name. DNS policy unload on shutdown can be enabled on the agent, but it works only when the L7 policy is set to * and the agent is shutdown gracefully.

This CFP introduces a standalone DNS proxy that can run alongside the Cilium agent, which should eliminate hard dependency for names that already have policy map entries in place.

## Motivation

Users rely on toFQDN policies to enforce network policies against traffic to destinations outside the cluster, typically to blob storage / other services on the internet. Rolling out the Cilium agent should not result in packet drops. Introducing a high availability (HA) mode will allow for increased adoption of toFQDN network policies in critical environments.

## Goals

* Introduce a streaming gRPC API for exchanging FQDN policy related information.
* Introduce a standalone DNS proxy (SDP) that binds on the same port as built-in proxy with SO_REUSEPORT.
* Enforce L7 DNS policy via SDP.

## Non-Goals

* Updating new DNS <> IP mappings when the agent is down.

## Proposal

### Overview

There are two parts to enforcing toFQDN network policy. L3/L4 policy enforcement against IP addresses resolved from an FQDN and policy enforcement on DNS requests (L7 DNS policy). To enforce L3/L4 policy, per endpoint policy bpf maps need to be updated. We'd like to avoid multiple processes writing entries to policy maps, so the standalone DNS proxy (SDP) needs a mechanism to notify agent of newly resolved FQDN <> IP address mappings. This CFP proposes exposing a new gRPC streaming API from the cilium agent. Since the connection is bi-directional, the cilium agent can reuse the same connection to notify the SDP of L7 DNS policy changes.

Additionally, SDP needs to translate the IP address to cilium identity to enforce the policy. This CFP proposes to retrieve the identity mapping from the cilium_ipcache BPF map. If / when other another abstraction is introduced for getting IP<->Identity mappings (for ex, in L7 proxy), this implementation can use that abstraction. This CFP will focus on the contract between SDP and Cilium agent to exchange minimum information for implementing the high availability mode.

In addition to existing unix domain socket (UDS) opened by the agent to host HTTP APIs, we'll need a new UDS for the gRPC streaming service with similar permissions. 

### RPC Methods

Method : UpdateMappings (Invoked from SDP to agent)

_rpc UpdatesMappings(steam FQDNMapping) returns (Result){}_
Request :
```
message FQDNMapping {
    string FQDN = 1;    // DNS Name of the request made by the client
    repeated bytes IPS = 2; // Resolved IP addresses
    uint32 TTL = 3;
    uint64 source_identity = 4; // Identity of the client making the DNS request
    bytes source_ip = 5; // IP address of the client making the DNS request
    int dns_response_code = 6;
}
```
Response :
```
message Result {
    bool success = 1;
}
```

Method : UpdatesDNSRules ( Invoked from agent to SDP via bi-directional stream )

_rpc UpdatesDNSRules(stream DNSPolicies) returns (Result){}_
Request :
```
message DNSServer {
  uint64 dns_server_identity = 1;  // Identity of destination DNS server
  uint32 dns_server_port = 2;
  uint32 dns_server_proto = 3;
}

message DNSPolicy {
  uint64 source_identity = 1;  // Identity of the workload this L7 DNS policy should apply to
  repeated string dns_pattern = 2;  // Allowed DNS pattern this identity is allowed to resolve.
  repeated DNSServer dns_servers = 3;
}

message DNSPolicies {
  repeated DNSPolicy egress_l7_dns_policy = 1;
}

```

*Note: `dns_pattern` follows the same convention used in CNPs. See https://docs.cilium.io/en/stable/security/policy/language/#dns-based for more details*

Response :
```
message Result  {
     bool success = 1;
}
```

### Load balancing

SDP and agent's DNS proxy will run on the same port using SO_REUSEPORT. By default, kernel will use round robin algorithm to distribute load evenly between all sockets in the reuseport group. If cilium agent's DNS proxy goes down, kernel will automatically switch all traffic to SDP and vice versa. In the future, we can consider using a custom bpf program to make SDP only act as a hot standby. See (PoC)[https://github.com/hemanthmalla/reuseport_ebpf/blob/main/bpf/reuseport_select.c] / eBPF summit 2023 talk for more details.


### High Level Information Flow

* Agent starts up with gRPC streaming service (only after resources are synced from k8s and ipcache bpf map is populated)
* SDP starts up.
* Connects to gRPC service, retrying periodically until success.
* Agent sends current snapshot for L7 DNS Policy enforcement via UpdatesDNSRules to SDP.
* On policy recomputation,  agent invokes UpdatesDNSRules.
* On DNS request from the client, DNS request redirects to DNS proxy port.
* Kernel round robin load balances between SDP and built in proxy.
* Assuming SDP gets the request, SDP enforces L7 DNS policy.
  * Lookup identity based on IP address via bpf map.
  * Check against policy snapshot if this identity is allowed to resolve the current DNS name and is allowed to talk to DNS server target identity (also needs lookup).
* Make upstream DNS request from SDP.
* On response, SDP invokes UpdatesMappings() to notify agent of new mappings.
* Release DNS response after success from  UpdatesMappings() / timeout.

### Handling SDP <> Agent re-connections

* When the agent is unavailable, SDP will periodically attempt to re-connect to the streaming service. Any FQDN<>IP mappings resolved when the agent is down will be cached in SDP and `UpdatesMappings` will be retried after establishing the connection.
  * A new bpf map for ipcache is populated on agent startup, so SDP needs to re-open the ipcache bpf map when the connection is re-established. See https://github.com/cilium/cilium/pull/32864 for similar handling in envoy.
  * On a new connection from SDP, the agent will invoke `UpdatesDNSRules` to notify SDP of all L7 DNS policy rules.

* SDP will not listen on the DNS proxy port until a connection is established with cilium agent and initial L7 DNS policy rules are received. Meanwhile, built-in DNS proxy will continue to serve requests. SDP relies on cilium agent for initial bootstrap. In future, we could make SDP retrieve initial policy information from other sources, but this is not in scope for this CFP.

### Handling Upgrades

Other than the streaming API from the agent, this CFP introduces a dependency on the ipcache bpf map which isn't a stable API exposed to components beyond the agent. Sufficent tests will be added to catch such datapath changes impacting SDP. In order to support a safe upgrade path, SDP would need to support reading from the current and future formats of the map (including possibly reading from an entire new map).
