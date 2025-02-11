# CFP-TBD: Egress Connection Timeouts

**SIG: SIG-DATAPATH, EGRESS-GATEWAY**

**Begin Design Discussion:** 2024-01-16

**Cilium Release:** X.XX

**Authors:** Murari Bhimavarapu <murarib@google.com>

**Contributors:** Carlos Abad <carlosab@google.com>

**Status:** Implementable


# Summary

We propose enabling users to adjust connection timeout settings for specific egress gateways via the addition of a new field in Cilium Egress Gateway Policy (CEGP).


# Motivation

Currently, Cilium relies heavily on default operating system connection timeout settings and offers control over connection timeouts at cluster or node level. It is not optimal for all workloads to be bound to these node level timeouts, especially with respect to egress gateways where prolonged idle connections can contribute to port exhaustion on the NAT gateway.

Modifying CiliumEgressPolicy to include an optional timeout field would allow us to ingest custom timeouts and give users additional control over egress connections. 


# Goals



*   Allow users to specify egress IPv4 connection timeouts via CiliumEgressGatewayPolicy.
*    Allow for back-compatibility with cilium-config default timeouts.


# Non-Goals



*   Modify connection timeouts for non-egress traffic.
*   Modify connection timeouts at a more granular level than CiliumEgressGatewayPolicy, i.e. per egress connection.
*   Modify egress connection timeouts for IPv6 traffic.


# Proposal


<table>
  <tr>
   <td style="background-color: #efefef">
   </td>
   <td style="background-color: #efefef">Option 1. Custom Conntrack Timeout
   </td>
   <td style="background-color: #efefef">Option 2. Timeout at Node Level
   </td>
   <td style="background-color: #efefef">Option 3. Timeout at Socket Level
   </td>
  </tr>
  <tr>
   <td style="background-color: null">Engineering Investment
   </td>
   <td style="background-color: #ffe599">Modify SNAT Datapath and conntrack creation to ingest and write custom timeouts to egress nat conntrack entries. 
   </td>
   <td style="background-color: #b6d7a8">Watch CEGP and modify CiliumNodeConfig for the respective gateway node
   </td>
   <td style="background-color: #ea9999"> New bpf program to hook into sockopt syscall event
   </td>
  </tr>
  <tr>
   <td style="background-color: null">Flexibility
   </td>
   <td style="background-color: #b6d7a8">Granularity can be achieved at the CEGP level
   </td>
   <td style="background-color: #ea9999">Can only achieve granularity at the node or cluster level, not by CEGP
<p>
   </td>
   <td style="background-color: #b6d7a8">Granularity can be achieved at the CEGP level
   </td>
  </tr>
  <tr>
   <td style="background-color: null">Performance/
<p>
Maintainability
   </td>
   <td style="background-color: #b6d7a8">Add minimal latency to the SNAT Datapath and space to BPF Maps. Cilium agent can continue to run without restart.
   </td>
   <td style="background-color: #ffe599">Write to CiliumNodeConfig is cheap but cilium agent will need to be restarted
   </td>
   <td style="background-color: #ea9999">Cilium agent pods can continue to run unchanged but the bpf program and map lookup will be called on every new socket, not just egress nat sockets.
   </td>
  </tr>
</table>


**Option 1. Custom Conntrack Timeout** was chosen as it minimizes impact on existing datapath while offering the best blend of flexibility and performance. 


# **[Preferred] Option 1: Add Timeouts Field to Existing BPF EGRESS_POLICY_MAP**


## Overview

**Steps to take are as follows:**



*   Add an optional timeout field to CEGP.
*   Modify CEGP ingestion logic to add new timeout fields to EGRESS_POLICY_MAP
*   Modify conntrack creation ([ct_create4](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L1015)) and conntrack timeout update  ([ct_update_timeout](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L94)) to take custom timeout values as parameters.
*    In bpf_host.c, modify the SNAT datapath to perform a lookup of the timeout. Pass timeout into the conntrack creation function to assign custom timeout to specific egress conntrack entries. 

<img width="392" alt="Program Overview" src="https://github.com/user-attachments/assets/1759346e-88ff-42ad-86df-fc6c40f1d917" />


## Add Optional Timeout Field to CEGP

Add an optional timeout field to CEGP:


```
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-sample
spec:
  bpf-ct-timeout-regular-any: 60
  bpf-ct-timeout-regular-tcp: 21600
  bpf-ct-timeout-regular-tcp-fin: 10
  bpf-ct-timeout-regular-tcp-syn: 60
  bpf-ct-timeout-service-any: 60
  bpf-ct-timeout-service-tcp: 21600
  bpf-ct-timeout-service-tcp-grace: 60
  ...      
```


This will need to be mirrored in Cilium’s internal representation of CEGP which is [PolicyMap](https://source.corp.google.com/h/gke-internal/third_party/cilium/+/master:pkg/maps/egressmap/policy.go;drc=2c8979e71bcc00b0135735334b71cce832554a3d;bpv=1;bpt=1;l=59?q=PolicyConfig&sq=repo:gke-internal%2Fthird_party%2Fcilium%20branch:master). 


## Add Timeouts Field to Existing BPF EGRESS_POLICY_MAP


```
struct egress_gw_policy_entry {
	__u32 egress_ip;
	__u32 gateway_ip;
	struct connection_timeouts;
};

struct connection_timeouts {
	__u32 bpf_ct_timeout_regular_any;
	__u32 bpf_ct_timeout_regular_tcp;
	__u32 bpf_ct_timeout_regular_tcp_fin;
	__u32 bpf_ct_timeout_regular_tcp_syn;
	__u32 bpf_ct_timeout_service_any;
	__u32 bpf_ct_timeout_service_tcp;
	__u32 bpf_ct_timeout_service_tcp_grace;
};
```
 

In the case where `connection-timeout` is not specified in CiliumEgressGatewayPolicy, we will populate the struct values for connection timeouts to be 0. This will result in the cilium-config defaults being used. 


## Modify the SNAT Datapath to Ingest and Update Custom Timeouts in Conntrack

In the [snat_v4_nat_handle_mapping](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/nat.h#L269) function in bpf/lib/nat.h, we need to perform 1 lookup.



1. Perform a lookup in the EGRESS_POLICY_MAP using the src ip address to get the Egress IP of the conntrack entry to be created. 

Pass this struct into the now modified [ct_create4](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L1015) function to create conntrack entries with custom timeouts.

<img width="476" alt="image" src="https://github.com/user-attachments/assets/4c9fb8e4-f42e-456c-9487-961013389dce" />


## Add Timeout Parameters to Conntrack Creation and Conntrack Timeout Update

Conntrack timeouts are updated on calls to [ct_update_timeout](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L94). [ct_update_timeout](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L94) is called by [ct_create4](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L1015) when new conntrack entries are created. 

We need to modify [ct_create4](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L1015) and [ct_update_timeout](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L94) to take an additional parameter of connection_timeouts struct. 

Then we will modify the logic of [ct_update_timeout](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L94) so if the connection_timeouts struct is nil, then the behavior is default to the node-level timeout config (i.e. Cilium config flags). Else, use the timeouts specified in the connection_timeouts struct when updating conntrack entries. 

<img width="483" alt="image" src="https://github.com/user-attachments/assets/29bf47ce-39ae-4ebb-9bbc-97ec81c7d199" />


## Option 1 Impacts:


### Performance of  SNAT Datapath 

**Latency:**

We will be performing one lookup in the EGRESS_POLICY_MAP for all egress nat cases. 

With EGRESS_POLICY_MAP LPM Trie search being O(log N), Big O complexity in the average case is **_O(log N)_**

**Space:**

We will be adding 4*7=28 bytes of memory to each entry in EGRESS_POLICY_MAP.

This will **increase** the size of all existing EGRESS_POLICY_MAP by **140%**.

With a default size of 16k entries, this will **add 448 KB of memory** overhead to each node in the default case. 

<span style="text-decoration:underline;">This space overhead is considered acceptable in favor of the speed we gain with just doing one map lookup.</span>


### Modification of helper functions ct_create4 and ct_update_timeout

We will be modifying the function signatures of these helpers, but default functionality will be back-compatible. For example, passing in a nil connection_timeouts struct should result in default timeout behavior. 


# **Option 2: Create a new BPF map to store Egress Connection Timeout Values**


## Overview 

<img width="490" alt="image" src="https://github.com/user-attachments/assets/d74a1e65-0920-4e99-afd2-496e5affb058" />

**Steps to take are as follows:**



*   Add an optional timeout field to CEGP.
*   Modify CEGP ingestion logic to create a new bpf map to store timeout values.
*   Modify conntrack creation ([ct_create4](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L1015)) and conntrack timeout update  ([ct_update_timeout](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L94)) to take custom timeout values as parameters.
*    In bpf_host.c, modify the SNAT datapath to perform a lookup of the timeout. Pass timeout into the conntrack creation function to assign custom timeout to specific egress conntrack entries. 


## Create a new BPF map to store Egress Connection Timeout Values


```
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, __u32 policy_config_id);
	__type(value, struct egress_connection_timeouts_val);
} EGRESS_CONNECTION_TIMEOUT_MAP __section_maps_btf;

struct egress_connection_timeouts_val {
	__u32 bpf_ct_timeout_regular_any; 
	__u32 bpf_ct_timeout_regular_tcp; 
	__u32 bpf_ct_timeout_regular_tcp_fin; 
	__u32 bpf_ct_timeout_regular_tcp_syn; 
	__u32 bpf_ct_timeout_service_any; 
	__u32 bpf_ct_timeout_service_tcp; 
	__u32 bpf_ct_timeout_service_tcp_grace;
};
```


Add policy_id field to Existing BPF EGRESS_POLICY_MAP value as well. 

**Key Hashing Methods:**



*   Take a 64 bit truncation of the SHA256 hash of CiliumEgressGatewayPolicy Name and Namespace.
    *   To correct  hash collisions we failover to a new hash when a collision occurs, either with a new algorithm or taking the next 64 bits of the SHA256 hash.
        *   1\*10<sup>-28</sup> chance of collision with 1 million entries 
*   Use a concatenation of Name and Namespace, max size of [506 chars](https://pwittrock.github.io/docs/concepts/overview/working-with-objects/names/#:~:text=By%20convention%2C%20the%20names%20of,precise%20syntax%20rules%20for%20names.)
*   Use key of saddr and daddr and make unique entry for each src endpoint

Include a boolean field `custom_timeouts_specified `to specify whether custom connection timeouts were specified in the CEGP. This will prevent additional map lookups in the default case.


```
struct egress_gw_policy_entry {
	__u32 egress_ip;
	__u32 gateway_ip;
	__u8 custom_timeouts_specified;
	__u32 policy_config_id;
};
```



## Modify the SNAT Datapath to Ingest and Update Custom Timeouts in Conntrack

In the SNAT datapath, the creation of conntrack entries for egress connections is made in the function [snat_v4_nat_handle_mapping](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/nat.h#L269).

In the [snat_v4_nat_handle_mapping](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/nat.h#L269) function in bpf/lib/nat.h, we need to perform 2 lookups.



1. Perform a lookup in the EGRESS_POLICY_MAP using the src ip address to get the Egress IP of the conntrack entry to be created.
2. Using the Egress IP, perform a lookup in the newly created EGRESS_CONNECTION_TIMEOUT_MAP to get the struct connection_timeouts.

Pass this struct into a modified [ct_create4](https://github.com/cilium/cilium/blob/2b61fd7a3e6098f791b58d3743a0e26d16b6c524/bpf/lib/conntrack.h#L1015) function to create conntrack entries with custom timeouts.

<img width="474" alt="image" src="https://github.com/user-attachments/assets/aca121e4-6c94-4021-ab9c-3431af76635c" />


## Option 2 Impacts:


### Performance of  SNAT Datapath 

**Latency:**

We will be performing 2 additional BPF Map lookups for conntrack entry creation in the worst case. 

We expect the majority of CiliumEgressGatewayPolicies to use default timeouts, in which case we will only perform one BPF Map lookup. 

With EGRESS_POLICY_MAP LPM Trie search being O(log N) and EGRESS_CONNECTION_TIMEOUT_MAP Hash search being O(1), Big O complexity in the average case is **_O(log N)_**

** **

**Space:**

EGRESS_POLICY_MAP: 

2 additional fields, 5 additional bytes per entry. 

`EGRESS_CONNECTION_TIMEOUT_MAP`: New Map. 

 4 bytes(map_type) + 8 bytes(key) + 4*7 bytes(connection timeout values) = 40 bytes per entry

