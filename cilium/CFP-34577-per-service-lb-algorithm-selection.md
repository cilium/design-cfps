# CFP-TODO: Per-service Load Balancing Algorithm Selection

**SIG: SIG-DATAPATH**  
**Begin Design Discussion: 2024-08-28**  
**Cilium Release:** 1.17  
**Authors:** Aleksander Mistewicz \<amistewicz@google.com\>

## Summary

Add annotation `service.cilium.io/scheduler` support to Cilium which will allow a user to select the load balancing algorithm on a per-service basis.

## Motivation

Before this CFP, a user needs to make a choice of load-balancing algorithm to  
use in their cluster. There is no one-size-fits-all choice.

`cilium-agent` can be configured by setting `bpf-lb-algorithm` to use one of the following load balancing algorithms:

1. Maglev \- compute [a hash](https://github.com/cilium/cilium/blob/v1.16.0-rc.2/bpf/lib/hash.h\#L9-L17) based on Source Address, Source Port, Destination Port and Protocol  
2. Random

| Algorithm | Static backend selection | Memory use (per service[^1]) | Memory use (total[^2]) |
| :---- | :---- | :---- | :---- |
| Maglev | \+ (as long as backends list is the same) | [65.5 KB \= 4B x 16381](https://github.com/cilium/cilium/blob/v1.16.0-rc.2/bpf/lib/lb.h\#L168) (constant) | 655 MB (10 000 services) |
| Random | \- | 120 KB \= [12 B](https://github.com/cilium/cilium/blob/v1.16.0-rc.2/bpf/lib/common.h\#L1053-L1067) x backends | 3 MB (250 000 endpoints) |

Similar requests that failed to gain traction:

1. [cilium\#8723](https://github.com/cilium/cilium/issues/8723) \- algorithm selection in service annotation  
2. [cilium\#26264](https://github.com/cilium/cilium/issues/26264) \- more load-balancing algorithms

`kube-proxy` lets users [select a load-balancing algorithm based on service annotation](https://github.com/cloudnativelabs/kube-router/blob/v2.1.3/docs/user-guide.md\#load-balancing-scheduling-algorithms).  
At the time of this writing, there are [14 algorithms](https://keepalived-pqa.readthedocs.io/en/latest/scheduling\_algorithms.html) to choose from.

## Goals

1. Support multiple load-balancing algorithms on a single node

## Non-Goals

1. Add support for new (other than `maglev`/`uniform`) load-balancing algorithms

## Proposal

Add a new value for `bpf-lb-algorithm` and a new `service.cilium.io/scheduler` annotation which will initially accept `maglev` or `uniform` values.

### Overview

1. Rename `lb{4,6}_select_backend_id` functions to include algorithm name  
   [code](https://github.com/cilium/cilium/blob/v1.16.0/bpf/lib/lb.h\#L1310-L1332)  
2. Compile eBPF with `LB{4,6}_MAGLEV_MAP_OUTER`  
   (default or behind a new value of `bpf-lb-algorithm`)  
   [code](https://github.com/cilium/cilium/blob/v1.16.0/bpf/lib/lb.h\#L78-L94)  
3. Add new `flag` to `lb{4,6}_service`  
   [code](https://github.com/cilium/cilium/blob/v1.16.0/bpf/lib/common.h\#L1064-L1065)  
4. Pass the flag based on `service.cilium.io/scheduler` annotation to `lb{4,6}_service`  
5. Diverge load-balancing code paths based on the flag set in `lb{4,6}_service`  
6. Add a new value for `bpf-lb-algorithm=all` or enable by default (i.e. `bpf-lb-algorithm` will set the default load-balancing algorithm to use for a service without `service.cilium.io/scheduler` annotation)

#### Pros

1. Slow rollouts \- a different algorithm can be selected/tested without affecting the entire cluster  
2. Extensible:  
   1. A new algorithms can be introduced without affecting current behavior (opt-in)  
   2. It is possible to achieve a feature-parity with `envoy`/`kube-proxy` in the future

#### Cons

1. Adding/Removing annotation to a service may be hard to handle  
2. Agent would need to support all LB algorithms at the same time  
   When compared to uniform-only clusters. It will have:  
   1. Slower eBPF compilation  
   2. Slower startup due to map precomputation  
   3. More BPF maps need to be allocated

[^1]:  With 10 000 endpoints

[^2]:  With 10 000 services across 250 000 endpoints (set by `bpf-lb-service-map-max`)