# CFP-45965: Service LoadBalancer IP as backend pod egress source IP

SIG: SIG Datapath / Service LoadBalancer / BGP

Begin Design Discussion: 2026-05-29

Cilium Release: TBD

Authors: Ayush Rathore <ayushrathor104@gmail.com>

Status: Provisional

Related Issue: https://github.com/cilium/cilium/issues/45965

## Summary

This CFP proposes an optional Cilium feature that allows backend pods selected by a Kubernetes `Service` of type `LoadBalancer` to use the Service LoadBalancer IP as the source IP for selected outbound traffic.

Today, Cilium can allocate and advertise Service LoadBalancer IPs for inbound traffic. However, outbound traffic from the backend pods normally uses the pod IP, node IP, or another egress identity depending on the datapath mode and NAT configuration. Some environments need outbound traffic from workloads behind a Service to be attributed to the same stable Service LoadBalancer IP used for inbound reachability.

The proposed feature provides Service-level egress identity:

```text
backend pod -> external target
source IP = Service LoadBalancer IP
```

The initial implementation target is IPv4 with a single LoadBalancer VIP owner model.

## Motivation

Some environments use Service LoadBalancer IPs as stable tenant-facing or workload-facing identities. For those environments, inbound and outbound identity should be symmetric from an external system's point of view.

Example use cases:

- external firewall allowlisting based on the Service LoadBalancer IP
- audit logs where external systems should see the Service IP rather than an internal pod or node IP
- tenant attribution and billing based on stable Service IPs
- legacy integrations that expect the same IP for inbound and outbound connectivity
- bare-metal environments where Cilium already owns LoadBalancer IP allocation/advertisement through LB IPAM, L2 announcement, or BGP

Without this feature, users may need to configure separate egress gateways or external NAT systems even though the Service already has a stable LoadBalancer IP.

## Goals

- Allow selected backend pods of a LoadBalancer Service to use that Service's LoadBalancer IP as egress source IP.
- Keep the feature opt-in and disabled by default.
- Support an IPv4 L2/single-owner model as the first implementation target.
- Preserve normal Cilium datapath behavior for Services that do not opt in.
- Avoid leaking raw pod-IP-sourced traffic from remote backend nodes when the Service LoadBalancer IP should be used.
- Support cross-node backends by steering traffic to the LoadBalancer VIP owner node.
- Preserve reply delivery back to the original pod, including remote backend pods.
- Define the design questions for BGP advertisement and ECMP behavior before expanding the implementation.

## Non-Goals

- This CFP does not propose replacing Cilium Egress Gateway.
- This CFP does not propose standalone L4 load-balancer mode.
- This CFP does not initially support IPv6.
- This CFP does not initially support distributed reverse NAT state across multiple nodes.
- This CFP does not initially guarantee long-lived connection survival across VIP owner failover.
- This CFP does not initially support multi-node BGP advertisement for the same egress VIP unless the reverse path safety model is agreed upon.

## Proposal

### Overview

The proposed first implementation uses a single LoadBalancer VIP owner model.

For a backend pod selected by an opted-in LoadBalancer Service:

1. The source/backend node detects that the pod should use a Service LoadBalancer IP as egress source.
2. If the LoadBalancer VIP owner is local, the node applies SNAT from `pod_ip` to `lb_ip`.
3. If the LoadBalancer VIP owner is remote, the source/backend node tunnels the packet to the owner node.
4. The owner node applies SNAT from `pod_ip` to `lb_ip`.
5. The external target replies to the LoadBalancer IP.
6. The owner node performs reverse NAT from `lb_ip` back to `pod_ip/pod_port`.
7. If the backend pod is remote, the owner node tunnels the reply back to the backend pod's node.
8. The backend pod receives the reply as part of the original connection.

Target flow:

```text
pod on worker-2
  -> source-node steer
  -> tunnel to LB VIP owner master-1
  -> owner SNAT pod_ip -> LB VIP
  -> external target replies to LB VIP
  -> owner reverse-DNATs LB VIP -> pod_ip
  -> reply is tunneled/delivered back to pod
```

### User-facing API

The initial API can be a Service annotation:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-lb-service
  annotations:
    service.cilium.io/egress-source-lb-ip: "true"
spec:
  type: LoadBalancer
  selector:
    app: example
  ports:
    - port: 80
      targetPort: 8080
```

The annotation is intentionally Service-scoped because the requested behavior is tied to the Service LoadBalancer IP identity.

Open question: maintainers may prefer this to be modeled as a policy or CRD instead of an annotation, especially if future versions need richer matching, per-destination rules, protocol selection, or multi-Service behavior.

### Datapath model

The current POC uses three conceptual maps:

```text
forward map:
  pod_ip -> lb_ip

reverse map:
  external reply tuple -> original pod_ip/pod_port

steer map:
  pod_ip -> { lb_ip, owner_node_ip }
```

When a backend pod sends traffic to an external destination, the source node checks whether the pod has an LB egress mapping.

If no mapping exists, traffic continues through the normal Cilium datapath.

If a mapping exists and the owner node is local, the packet is SNATed locally to the LoadBalancer IP.

If a mapping exists and the owner node is remote, the packet is tunneled to the owner node. The owner node then performs the SNAT operation so that external traffic is emitted from the LoadBalancer IP owner.

On reply, the owner node uses reverse state to restore the original pod destination. If the pod is remote, the owner tunnels the packet back to the pod's node.

### Control-plane model

The control-plane side is responsible for maintaining the mapping between backend pods, Service LoadBalancer IPs, and VIP owner nodes.

Required inputs:

- opted-in Service
- Service LoadBalancer IP
- selected backend pods/endpoints
- current LoadBalancer VIP owner node
- owner node tunnel endpoint

For L2 announcement, this can align with the current single-owner model where one node answers for the VIP.

For BGP, the ownership model needs explicit design agreement because multi-node advertisement may allow replies to land on a node that did not create reverse NAT state.

### Current POC status

A working IPv4 POC exists on branch:

```text
lb-egress-source-lbip
```

Rebased base:

```text
e41231047c
```

Final rebased checkpoint commit:

```text
4f2486fdab lb-egress: steer remote backend egress via LB IP owner
```

Validated runtime example:

```text
backend pod:     10.233.80.245 on stg-nc-workload2-worker-2
LB VIP:          10.20.32.240
LB VIP owner:    stg-nc-workload2-master-1
owner node IP:   10.20.36.181
external target: 10.20.32.9:80
```

Observed result:

- Worker/source node no longer emits raw pod-IP-sourced packets to the external target.
- Owner node emits the full TCP/HTTP flow sourced from the LoadBalancer IP.
- External target replies to the LoadBalancer IP.
- Backend pod receives a successful HTTP response.
- Cross-node backend delivery works.
- Reverse path can tunnel replies back to the remote backend pod.
- Branch has been rebased onto upstream/main and smoke-tested after rebase.

### Validation commands used in the POC

BPF build:

```bash
make -C bpf
```

Go tests:

```bash
go test -count=1 \
  ./pkg/lb_egress \
  ./pkg/l2announcer \
  ./pkg/loadbalancer/... \
  ./pkg/maps/lbegress
```

Runtime compile checks:

```bash
kubectl -n kube-system logs "$POD" -c cilium-agent --since=5m | \
  grep -Ei "undeclared function|Failed to compile bpf_lxc|Endpoint regeneration failed|verifier|permission denied" || true
```

Expected result:

```text
no output
```

## Impacts / Key Questions

### Impact: stateful behavior tied to VIP ownership

This feature is stateful because the node that performs SNAT must also be able to reverse-NAT replies. In the first implementation, reverse state is local to the owner node.

This is straightforward for a single-owner L2 model, but it creates an important BGP design question.

### Key Question 1: BGP single-owner advertisement vs multi-node ECMP

For BGP-advertised LoadBalancer IPs, should this feature preserve a single egress owner or support multi-node advertisement/ECMP?

#### Option 1: Single-owner BGP advertisement

Advertise the LoadBalancer IP only from the selected egress owner node when this feature is enabled.

##### Pros

- Matches the current POC model.
- Keeps reverse NAT state local and simple.
- Avoids reply packets landing on nodes without reverse state.
- Easier first implementation.
- Easier to reason about operationally.

##### Cons

- Reduces load distribution for the selected VIP.
- Owner node can become a bottleneck.
- Owner failover may break existing connections.
- Needs careful integration with BGP advertisement logic.

#### Option 2: Multi-node BGP advertisement / ECMP

Allow multiple nodes to advertise the same LoadBalancer IP while using it as egress source IP.

##### Pros

- Better traffic distribution.
- Better alignment with ECMP-based BGP deployments.
- May avoid single-node bottlenecks.

##### Cons

- Reply may land on a different node than the node that performed SNAT.
- Requires symmetric ECMP assumptions, distributed reverse state, or another mechanism.
- Significantly more complex for the first implementation.
- Failure behavior is harder to reason about.

### Key Question 2: Service annotation vs CRD/policy

The POC uses a Service annotation because the behavior is Service-scoped.

However, if maintainers expect future matching rules, a CRD or policy may be preferable.

Possible future fields:

- destination CIDR selection
- protocol/port selection
- per-Service egress behavior
- explicit owner selection
- fallback behavior
- BGP advertisement mode

### Key Question 3: separate maps vs deeper CT/NAT integration

The POC uses separate LB egress maps for forward, reverse, and steering behavior.

Open question: should the implementation integrate more deeply with Cilium CT/NAT/EgressGateway infrastructure instead?

Potential advantages of deeper integration:

- reuse existing timeout/GC behavior
- reuse existing NAT observability
- reduce duplicate state-management code
- align better with established datapath patterns

Potential advantage of separate feature maps:

- clearer experimental scope
- easier initial POC isolation
- less risk of disrupting existing datapath behavior

### Key Question 4: owner failover and connection survival

If the VIP owner changes, existing reverse NAT state may be lost. The first implementation may only guarantee that new flows work after owner change.

Open question: is this acceptable for alpha/initial implementation, or should owner failover behavior be solved before merge?

## Alternatives Considered

### Use Cilium Egress Gateway only

Cilium Egress Gateway already provides a mechanism for controlling outbound source IP behavior. However, this CFP is specifically about reusing the Kubernetes Service LoadBalancer IP as the egress identity for the backend pods of that Service.

Egress Gateway may still be relevant as an implementation building block or conceptual model.

### Local SNAT on every backend node

Each backend node could SNAT pod traffic directly to the LoadBalancer IP.

This is simple for outbound packets, but unsafe for replies unless the network guarantees that replies always return to the same node or reverse state is distributed.

### External NAT device

Users can solve this outside Cilium with an external firewall/router/NAT system.

This adds operational complexity and duplicates functionality when Cilium already manages LoadBalancer IP ownership and advertisement.

## Failure Modes

### Owner node failure

Expected initial behavior:

- existing flows may break
- a new owner is selected by the existing LB ownership mechanism
- new flows can use the new owner once maps are updated

### Cilium agent restart on owner

Expected initial behavior:

- existing reverse state may be lost
- new flows should recover after maps are repopulated

### Backend pod deletion

Expected behavior:

- forward and steer entries for the deleted backend should be removed
- stale reverse state should eventually expire or be garbage-collected

### Service annotation removal

Expected behavior:

- new outbound flows from backend pods use normal datapath behavior
- old reverse state should expire or be garbage-collected

## Test Plan

### Unit / integration tests

- annotation parsing and Service model flag
- map lifecycle and cleanup
- endpoint/backend selection updates
- owner update handling
- BPF build coverage for IPv4-only and dual-stack builds
- L2 owner update behavior

### Runtime tests

- local backend pod with local VIP owner
- remote backend pod with remote VIP owner
- source node packet capture confirms no raw pod IP leak
- owner node packet capture confirms LB-IP-sourced traffic
- external target receives traffic from LB IP
- reply returns to pod
- owner node restart
- backend pod deletion/recreation
- annotation removal

### BGP validation tests

- advertise VIP only from selected owner node
- advertise VIP from multiple nodes and observe reverse-state behavior
- owner failover while new flows are created
- router ECMP behavior with asymmetric replies

## Future Milestones

### Milestone 1: IPv4 L2 single-owner alpha

Initial implementation behind an opt-in feature gate and Service annotation.

### Milestone 2: BGP ownership mode

Define and implement the BGP-safe ownership model.

### Milestone 3: reverse map timeout and GC

Add production-grade timeout, cleanup, and observability.

### Milestone 4: deeper CT/NAT integration

If maintainers prefer, migrate from standalone feature maps into existing Cilium CT/NAT infrastructure.

### Milestone 5: IPv6 support

Add IPv6 support after the ownership and state-management model is accepted.

### Milestone 6: richer API

If needed, replace or supplement the Service annotation with a CRD/policy for richer matching and operational controls.

## Open Maintainer Questions

1. Is the single LoadBalancer VIP owner model acceptable for an initial implementation?
2. For BGP-advertised LoadBalancer IPs, should this feature force single-owner advertisement, or should it support multi-node advertisement/ECMP?
3. Should the datapath integrate with existing CT/NAT/EgressGateway machinery instead of separate lb-egress maps?
4. Is a Service annotation acceptable as the initial API?
5. Is it acceptable for the initial implementation to preserve new-flow correctness but not long-lived connection survival across owner failover?
6. Which maintainer group should primarily review this: Service LB, datapath, BGP, or Egress Gateway?
