# CFP-12781: Enforce Host Firewall Ingress Policy Before NodePort DNAT/SNAT

**SIG: SIG-Policy** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-05-25

**Cilium Release:** 1.20

**Authors:** Jakub Hlavnicka <jakub.hlavnicka@illumio.com>

**Status:** Draft

**Issue:** [cilium/cilium#12781](https://github.com/cilium/cilium/issues/12781)

## Summary

When both `ENABLE_HOST_FIREWALL` and `ENABLE_NODEPORT` are active, enforce host firewall ingress policy on the original (pre-DNAT/SNAT) packet before NodePort LB performs DNAT and SNAT. This enables `CiliumClusterwideNetworkPolicy` to match on original external source IPs and NodePort destination ports, which is not possible today because DNAT rewrites the destination to the backend pod IP and SNAT obscures the original client address.

## Motivation

Without this change, host firewall policy evaluates the packet **after** NodePort DNAT/SNAT has already rewritten the destination (and potentially the source). This means policies cannot:

1. Block traffic to the NodePort range (30000-32767) from external sources
2. Whitelist specific external source IPs for particular NodePort ports

For example, the following policy does not work as intended today:

```yaml
apiVersion: cilium.io/v2
kind: CiliumCIDRGroup
metadata:
  name: allowed-nodeport-source
spec:
  externalCIDRs:
  - 142.217.23.90/32
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: host-baseline
spec:
  nodeSelector: {}
  ingress:
  - fromEntities:
    - world
    toPorts:
    - ports:
      - port: "1"
        endPort: 30000
        protocol: TCP
      - port: "1"
        endPort: 30000
        protocol: UDP
  - fromEntities:
    - cluster
  - fromCIDRSet:
    - cidrGroupRef: allowed-nodeport-source
    toPorts:
    - ports:
      - port: "30500"
        protocol: TCP
  egress:
  - toEntities:
    - world
  - toEntities:
    - cluster
  - toEntities:
    - health
```

This policy intends to:
1. Allow world traffic only to ports below 30000
2. Allow all cluster-internal traffic
3. Allow a specific external IP (142.215.23.90) to reach NodePort 30500

Today, rules (1) and (3) are **ineffective** because by the time host firewall evaluates the packet, DNAT has already rewritten the destination port to the backend pod's port.

**Related Issue:** [cilium/cilium#12781](https://github.com/cilium/cilium/issues/12781)

## Goals

* Enable `CiliumClusterwideNetworkPolicy` to match on original external source IPs before SNAT obscures them
* Enable `CiliumClusterwideNetworkPolicy` to match on NodePort destination ports before DNAT rewrites them to backend pod ports
* Provide a feature flag to opt-in to this behavior, ensuring backwards compatibility with existing deployments
* Maintain existing NodePort load balancing functionality when the feature is disabled (default)

## Proposal

### Overview

Introduce a new configuration option `hostFirewall.enforceBeforeNodePortDNAT` (defaulting to `false`) that, when enabled, enforces host firewall ingress policy on the original packet before NodePort LB performs DNAT and SNAT.

The implementation adds two new BPF tail call targets (`CILIUM_CALL_IPV4_NODEPORT_HOST_POLICY` and `CILIUM_CALL_IPV6_NODEPORT_HOST_POLICY`) that run the policy check in their own stack frame to avoid exceeding the 512-byte BPF stack limit, then recirculate via `CILIUM_CALL_IPV4/6_FROM_NETDEV` with `TC_INDEX_F_SKIP_HOST_FIREWALL` set so the second pass skips straight to DNAT.

### Configuration

A new Helm value and agent configuration option:

```yaml
hostFirewall:
  enabled: true
  enforceBeforeNodePortDNAT: false  # Default: false for backwards compatibility
```

When `enforceBeforeNodePortDNAT: true`:
- Host firewall ingress policy is evaluated on the original packet (pre-DNAT/SNAT)
- Policy can match on original external source IPs
- Policy can match on NodePort destination ports (30000-32767)

When `enforceBeforeNodePortDNAT: false` (default):
- Current behavior is preserved
- Host firewall ingress policy is evaluated after NodePort DNAT/SNAT
- Existing deployments are unaffected

### BPF Implementation

The implementation introduces new tail call targets that:

1. Intercept packets destined for NodePort services before DNAT/SNAT
2. Run host firewall policy check against the original packet headers
3. If policy allows, recirculate the packet with `TC_INDEX_F_SKIP_HOST_FIREWALL` flag set
4. The second pass proceeds directly to NodePort DNAT/SNAT processing

This design:
- Avoids exceeding the 512-byte BPF stack limit by using separate tail call frames
- Ensures policy verdict is based on original packet headers
- Maintains full NodePort functionality after policy check passes

### Packet Flow (with feature enabled)

```
External packet arrives
    |
    v
tail_handle_ipv4_from_netdev
    |
    v
[Feature enabled?] --No--> [Current flow: DNAT/SNAT then policy]
    |
    Yes
    v
tail_nodeport_ipv4_host_policy (new)
    |
    v
[Host firewall policy check on ORIGINAL packet]
    |
    v
[Policy allows?] --No--> DROP
    |
    Yes
    v
Set TC_INDEX_F_SKIP_HOST_FIREWALL
    |
    v
Recirculate to tail_handle_ipv4_from_netdev
    |
    v
[Skip host firewall, proceed to DNAT/SNAT]
    |
    v
NodePort/LB processing
```

## Why `loadBalancerSourceRanges` Is Not Sufficient

Kubernetes `Service.spec.loadBalancerSourceRanges` provides per-service source IP filtering for LoadBalancer services. While superficially similar, it does not address the use cases motivating this CFP for several reasons:

### 1. Not a Centralized Policy Mechanism

`loadBalancerSourceRanges` is a per-service field that must be set on each individual Service or Gateway object. This means an external controller or platform operator would need to dynamically patch the configuration of user-owned services with allowed source IP ranges. This is invasive and conflict-prone — in environments where services are managed by GitOps tooling such as ArgoCD or Flux, external mutations to service specs will be detected as drift and reverted, creating a reconciliation loop. A `CiliumClusterwideNetworkPolicy`, by contrast, is a single cluster-wide policy object that is independent of individual service definitions.

### 2. No Support for Exclusions (Allow + Deny Combinations)

`loadBalancerSourceRanges` is a whitelist-only mechanism — it can only specify which source CIDRs are allowed. There is no way to express exclusion rules or combine allow and deny semantics. `CiliumClusterwideNetworkPolicy` supports richer policy expressions including `fromCIDRSet` with `except` fields.

### 3. Requires Non-Default Cilium Configuration

By default, Cilium only enforces `loadBalancerSourceRanges` on LoadBalancer and ExternalIPs service frontends. Extending enforcement to NodePort frontends requires setting `bpf.lbSourceRangeAllTypes=true` in the Cilium Helm configuration. This is a non-starter on natively integrated Cilium deployments such as GKE Datapath v2 and AKS ACNS, where platform operators do not control or expose Cilium configuration flags.

### 4. Kubernetes API Rejects `loadBalancerSourceRanges` on NodePort Services

The Kubernetes API server validates that `spec.loadBalancerSourceRanges` (and the legacy annotation `service.beta.kubernetes.io/load-balancer-source-ranges`) may only be set when `spec.type` is `LoadBalancer`. Attempting to apply source ranges to a NodePort service results in:

```
The Service "example" is invalid: spec.LoadBalancerSourceRanges: Forbidden: may only be used when `type` is 'LoadBalancer'
```

This means source range filtering via this mechanism is fundamentally unavailable for NodePort services at the Kubernetes API level, regardless of what the underlying CNI supports.

## Impacts / Key Questions

### Impact: Breaking Change for Existing Users

This is a significant semantic shift in how host firewall policy interacts with NodePort traffic. Enabling this by default would break existing deployments that rely on the current behavior where policies match post-DNAT addresses.

**Mitigation:** The feature is gated behind `hostFirewall.enforceBeforeNodePortDNAT` which defaults to `false`. Users must explicitly opt-in.

### Key Question: Feature Flag Naming and Location

Where should the feature flag live in the configuration hierarchy?

### Option 1: `hostFirewall.enforceBeforeNodePortDNAT`

#### Pros

* Clear association with host firewall functionality
* Explicit about what the flag controls (timing relative to NodePort DNAT)
* Follows existing Helm chart patterns

#### Cons

* Long flag name
* May not be discoverable for users looking under NodePort config

### Option 2: `nodePort.hostFirewallBeforeDNAT`

#### Pros

* Discoverable under NodePort configuration
* Associates the feature with NodePort behavior

#### Cons

* Conceptually the change is to host firewall behavior, not NodePort behavior
* May cause confusion about what exactly is being changed

### Key Question: Default Behavior in Future Releases

Should this become the default behavior in a future major Cilium release?

### Option 1: Keep disabled by default forever

#### Pros

* Never breaks existing users
* Predictable behavior

#### Cons

* Arguably incorrect default (policy should match what users specify)
* Requires users to know about and enable the flag

### Option 2: Enable by default in a future major release (e.g., Cilium 2.0)

#### Pros

* Correct behavior becomes the default eventually
* Major version is appropriate for breaking changes

#### Cons

* Will require migration effort for affected users
* Needs clear deprecation timeline and warnings
