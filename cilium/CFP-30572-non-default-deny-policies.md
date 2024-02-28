# CFP-30572: Non-default-Deny Network Policy

**SIG: SIG-Policy**

**Begin Design Discussion:** 2024-02-02

**Cilium Release:** 1.16

**Authors:** Casey Callendrello <cdc@isovalent.com>

## Summary

Allow users to create network policies that do not implicitly set endpoints in to a default-deny mode. 

## Motivation

Consider a large, multitenant cluster. The cluster administrator is distinct from
the workload owners, and the workloads are diverse and pre-existing.

The administrator may wish to create a Cilium Clusterwide Network Policy (CCNP)
that affects most or all existing workloads. For example, they may wish to
- Globally grant access to a security scanner
- Ensure that workloads can access kube-dns
- Proxy all DNS requests for monitoring purposes
- Deny access to sensitive IPs, such as cloud provider metadata APIs

All of these can already be accomplished by the existing policy language. However, there
is an operational risk here: any of these broad-based policies may be the first policy
that applies to a given endpoint.

By default, all Cilium endpoints have unrestricted network access. If at least one
network policy applies to an endpoint, then all traffic is denied except that
explicitly allowed by policy.

This presents a challenge for the cluster administrator. They cannot assume that
existing, heterogeneous worloads already have network policies provisioned. So, by
creating an administrative network policy -- even one that only allows traffic --
they may select endpoints that **previously were in the allow-all mode**. Thus, 
they may unintentionally block legitimate traffic for an existing workload, causing an outage.

## Goals

* Ability to safely roll-out broad-based network policies, without the risk
  of disrupting existing traffic.

## Non-Goals

* An iterative, iptables-style firewall rules engine
* Arbitrary, multi-level rule priority

## Proposal

### Overview

We propose a new field in the policy spec, `EnableDefaultDeny`. This field specifies
whether or not endpoints selected by a policy should be placed in to default-deny mode
_on behalf of this policy_.

This field is optional. The default value per-direction is `true`. This matches the existing behavior,
wherein any policy automatically sets the subject endpoints' mode to default-deny. 


Sample YAML & go types:
```yaml
enableDefaultDeny:
  egress: false
  ingress: true
```
```go
type PolicySpec struct {
  // ...
  EnableDefaultDeny DefaultDenyConfig `json:"enableDefaultDeny,omitempty"`
}

type DefaultDenyConfig struct {
  Ingress *bool `json:"ingress,omitempty"`
  Egress  *bool `json:"egress,omitempty"`
}
```

Note that this is unrelated to the `IngressDeny` and `EgressDeny` policy expressions. Those dictate the action for selected peer traffic, whereas the `EnableDefaultDeny` specifies what should happen with unselected peer traffic.

### Per-direction mode

Endpoints have a default mode *for each direction*. That is to say, if an endpoint only has policies with ingress peer selectors, then ingress is default-deny, and egress is default-allow.

The rest of this document, for shorthand, assumes all policies operate in the same direction. The logic described is identical but independent for each direction.

### Precedence

When an endpoint is selected by multiple policies, default-deny takes precedence over default-allow. That is to say, if a new policy with `enableDefaultDeny: false` selects an endpoint with existing policies, no new traffic shall be allowed except that selected by the new policy.

For each traffic drection, an endpoint can be in the following states:

* No policies select this endpoint. Default allow.
* Only `enableDefaultDeny true` policies select. Default deny; only selected peers allowed.
* `enableDefaultDeny true` and `false` policies select. Default deny; only selected peers allowed.
* Only `enableDefaultDeny false` policies select. Default allow; only traffic explicitly blocked by an `IngressDeny` / `EgressDeny` selector is blocked.


## Examples

### Allow a security scanner full access to the cluster

```yaml
kind: CiliumClusterwideNetworkPolicy
spec:

  enableDefaultDeny:
    egress: false

  endpointSelector: {}
  ingress:
  - fromEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": security-team
```

### Observe all DNS traffic

```yaml
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: all-ur-dns-are-belong-to-us
spec:

  enableDefaultDeny:
    egress: false
  
  endpointSelector: {}
  egress:
    -  toPorts:
        - ports:
          - port: "53"
            protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
```

## Impacts / Key Questions

### Impact: More permissive policy engine

Making the policy engine more permissive is always a sensitive change. It is important that user intent is respected, and that no traffic is inadvertently allowed. It is critical that existing policies are not made more permissive.

This proposal attempts to respect the user's intent by making the permissive behavior explicitly opt-in and a part of the existing policy API. It does not create any opaque configuration flags or mystery annotations.

As this is a new field, this proposal does not affect any existing policies. Workloads with network policies will not see a change in allowed / denied traffic. Likewise, workloads without any policies will be unaffected.

This proposal does not change the precedence of Deny vs. Allow *peer selectors* -- as always, an `IngressDeny` selector always takes precedence over an `Ingress` selector, independent of the default action.

### Alternative: Agent Configuration

The policy engine already has a tri-state enablement mode:

- Disabled
- Default
- Always

The default is as described above -- endpoints without policy are permissive and the creation of policy changes that endpoint to restricted. Administrators may choose to completely disable network policy (in which all traffic is allowed, always), or set all endpoints to default-deny regardless of policy.

This value cannot be changed on a per-endpoint basis; only per-Agent.

### Option 1: Agent-level configuration

We could add a fourth policy enablement mode that changes the defaulting logic for the Cilium agent. When set to this mode, CCNPs would never cause the subject endpoints to be in the default-deny mode. Other policies, however, still would.  

#### Pros

* Does not require any API changes

#### Cons

* Cluster specific. A workload that is migrated between clusters may stop working. This breaks the principle that the existing API objects should describe workloads completely.
* Diverges the logic between CNP and CCNP. An administrator may be surprised that an otherwise-identical CNP and CCNP would behave completely differently.
* Opaque. The same CCNP, inadvertently applied to a cluster without this option, will instantly drop all traffic. This violates the principle of least surprise. 
* Inflexible. One could imagine a scenario in which all new namespaces are required to have a zero-trust-style network policy, but existing namepaces are "grandfathered". The administrator may wish to create a CCNP that emulates a default-deny mode, *but only for namespaces without a "grandfathered" label*. Given the existing policy language, this would be easy. If, however, the default action is not configurable on a per-CCNP basis, this is impossible.

### Option 2: Per-namespace annotation

#### Pros

* Does not require any API changes
* Non-opaque; workload owners could view enforcement status

#### Cons

* Incomplete. We have a rich policy API type; it should be the complete description of a workload's policy state.
* Surprising. The same CCNP may behave differently on different clusters, based on an unrelated and opaque namespace annotation.
* Inconsistent. Network policies operate on labels; annotations are not supposed to affect network policy
* Imprecise. This only allows for configuration on a namespace basis; where the policy engine can select on a much finer grained basis.

## Future Milestones

### Kubernetes NetworkPolicy equivalent

One could imagine an annotation on a Kubernetes NetworkPolicy that had a similar effect. 

### Hard-coded network policies

Other end-users would like the ability to configure super-CCNPs that cannot be changed by cluster administrators. These would be provided either by ConfigMap or similar mechanism.
