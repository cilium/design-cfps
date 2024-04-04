# CFP-31752: Active Connection Tracking

**SIG: SIG-DATAPATH**

**Begin Design Discussion:** 2024-03-13

**Cilium Release:** 1.16

**Authors:** Aleksander Mistewicz <amistewicz@google.com>, Rob Scott <robertjscott@google.com>

## Summary

Count number of opened and closed connections, grouped by service and zone.

This information can be used to close the feedback loop with a control plane to
implement the behavior people expect - fill the local zone to capacity and then
spillover to other zones. Without a control plane, this is still useful as
another observability point.

Discussion kicked off over [the design
doc](https://docs.google.com/document/d/1EdhtDavDzpEk7jamWhNV7uM5h6MhHbVmy7bI8U8nitM/edit#heading=h.dkumlwuhiwuf)
and later presented during Cilium Developer Summit on 2024-03-18.

## Motivation

One of the takeaways from the [broader Cilium xDS
discussion](https://github.com/cilium/design-cfps/pull/14) is that we could
really benefit from some kind of feedback loop with a control plane. This is
particularly important for the kinds of behavior people expect for Topology
Aware Routing - fill the local zone to capacity and then spillover to other
zones. 

One of the most useful metrics for determining how "full" an endpoint is would
be how many active and new connections an endpoint is handling. This information
can be paired with overall resource utilization information for each endpoint to
make more effective routing decisions.

Although this use case initially surfaced in the xDS discussion, we believe it's
useful beyond just xDS integration. If we can track the number of outgoing
connections per zone and Service, this data will likely be useful far beyond
xDS. 

We propose adding this feature independently from the xDS proposal, and reusing
it for xDS only if/when that broader CFP is accepted.


## Goals

* Add metrics to make it easier to observe traffic distribution across zones
* Surface information needed by control plane to better distribute traffic


## Non-Goals

* Change the behavior of load balancers
* Supplement or replace data provided by Hubble


## Proposal 1: Introduce a new BPF map: cilium_lb_lrs

### Overview

Name: `cilium_lb_lrs`

Key: (`svc_id` - 2 bytes, `zone` - 1 byte, `pad` - 1 byte)

Value: (`opened` - 4 bytes, `closed` - 4 bytes)

When connection to a service is opened
[[ct\_create4](https://github.com/cilium/cilium/blob/v1.15.3/bpf/lib/conntrack.h#L980),
[ct\_create6](https://github.com/cilium/cilium/blob/v1.15.3/bpf/lib/conntrack.h#L907)]
we will fetch zone ID from the selected backend. Then we will fetch the entry
and increment `opened` counter.

When connection to a service is closed
[[\_\_ct\_lookup](https://github.com/cilium/cilium/blob/v1.15.3/bpf/lib/conntrack.h#L312)]
we will use `rev_nat_id` (which corresponds to `svc_id`) and `backend_id` (which
is stored in `rx_bytes` for `CT_SERVICE` entries). We will resolve zone ID by
fetching the corresponding backend entry and then increment the `closed`
counter.

Unfortunately it doesn't catch all cases. One example is "connection refused".
These entries are removed by GC process in the agent. Active connection tracking
process should combine data from both sources (GC and BPF map) in order to
provide accurate gauges for new and ongoing connections.

Alternative solution would count only the connections that have already seen a
reply SYN packet.However, in this approach we won't be able to extend this
work to UDP.

### Pros

* Fast: additional code runs only on connection open and close
* `uint32` for counters guarantees atomic operations

### Cons

* Some events may be missed by datapath so the information must be combined with
  Agent's (i.e. counting service entries that were GCed)
* `uint32` will overflow, so metrics reader will need to adjust for that

### Alternatives

* Use event-based mechanism (for example: trace monitoring / Hubble) -
  unreliable, too slow
* Dump conntrack table - too slow


## Proposal 2: Pass zone information from EndpointSlices to lb{4,6}_backend

### Overview

Storing zone information alongside backend entry makes it easy to make (service,
zone) tuples.

[Endpoint](https://pkg.go.dev/k8s.io/kubernetes/pkg/apis/discovery#Endpoint) in
[EndpointSlice](https://pkg.go.dev/k8s.io/kubernetes/pkg/apis/discovery#EndpointSlice)
already contains field `Zone`. We can copy it to
[Backend](https://pkg.go.dev/github.com/cilium/cilium@v1.15.1/pkg/k8s#Backend)
and later to
[lb4_backend](https://github.com/cilium/cilium/blob/v1.15.3/bpf/lib/common.h#L1058)
and
[lb6_backend](https://github.com/cilium/cilium/blob/v1.15.3/bpf/lib/common.h#L999)
in BPF map.

As zone (string) from `EndpointSlice` needs to be converted to zone (uint8) in
`lb{4,6}_backend`, we will introduce a new configuration option to provide a
list of supported/tracked zones. This way the mapping could be preserved across
agent restarts.

### Pros

* Producing (service, zone) tuples in datapath becomes easy
* Backwards-compatible as both `lb4_backend` and `lb6_backend` have padding that
  can be repurposed to store a single byte of zone ID

### Cons

* Relies on using EndpointSlices (which any reasonably-sized cluster should use
  anyway)

### Alternatives

* Use information already available in eBPF: backend->node->label->zone
* Store counters alongside `lb{4,6}_backend` - backends are shared between
  services


## Proposal 3: Add metrics to surface the data

As xDS integration may take some time to implement and there are many users who
don't need it, we can expose this data as metrics.

```
# HELP cilium_opened_connections_count Number of opened connections to service
# TYPE cilium_opened_connections_count counter
cilium_opened_connections_count{src_zone,dst_zone,svc_ip,svc_port,svc_proto} 456

# HELP cilium_closed_connections_count Number of closed connections to service
# TYPE cilium_closed_connections_count counter
cilium_closed_connections_count{src_zone,dst_zone,svc_ip,svc_port,svc_proto} 123
```

### Pros

* non-xDS users can still benefit from this work by enabling another
  observability datapoint

### Cons

* Cardinality can be too high in some cases. As a remediation, we will introduce
  a hard limit in addition to scope definition by zone (see Proposal 2)

### Alternatives

* Expose metrics inferred from open/close connection counters as new/active
  gauges


## Impacts / Key Questions

### Metric cardinality

Metric cardinality is O(services \* zones). In a typical, big cluster with 10,000
services, 3 zones, 2 ports per service, it results in at most 10,000 \* 3 \* 2 \* 2
= 120,000 metric series on a single node.

However, this is unrealistic. We have successfully deployed Hubble metrics which
are aggregated by pod and workload. In practice, it is extremely rare for a pod
(or a set of pods on a node) to try to contact all possible services in the
cluster. Realistically the maximum cardinality per node is 100 pods \* 5 unique
services \* 2 ports \* 3 zones = 3,000.


1. We can exclude misbehaving metrics (i.e. publish only top X talkers) and/or
   introduce a hard limit for the number of series published
2. This feature is completely optional, so it is up to the user to decide if
   they accept the metric increase
3. The user can enable it for only a subset of zones (by not listing them in the
   configuration)
