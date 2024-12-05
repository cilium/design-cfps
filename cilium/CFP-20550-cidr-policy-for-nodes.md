# CFP-20550: CIDR policy for nodes

**SIG: SIG-Policy**

**Begin Design Discussion:** 2023-04-17

**Cilium Release:** 1.14 or 1.15

**Authors:** Joe Stringer <joe@cilium.io>

## Summary

Resolve policy incompatibilities that result in Kubernetes NetworkPolicy
failing to match traffic towards cluster destinations based on IP address
ranges.

## Motivation

Some users have a requirement in their environment to use generic Kubernetes
NetworkPolicy objects to configure the set of network endpoints that
applications may communicate with. Cilium's interpretation of CIDR policy does
not currently select endpoints within the cluster, including cases where the
kube-apiserver endpoints are on nodes within the cluster, or cases where the
user deploys a hostNetwork application onto nodes within the cluster. Such
users may be able to work around this limitation in existing versions by making
use of [CiliumNetworkPolicy "entities" constructs] in order to select the
"kube-apiserver" or "remote-node" entities in policies. This allows the
relevant traffic, however it does not meet the requirement for expressing the
policy in standard k8s resources.

The following issues indicate that there is strong community interest in removing this limitation:
- Kubernetes network policy with node ip cidr doesn't work ([cilium#12277])
- Cilium non-deterministically classifies CIDR policy matches for range with node IPs ([cilium#16308])
- Kubernetes network policy for api-server ([cilium#20550])
- Not possible to restrict egress traffic to the Kube API (and DNS) only ([eks-anywhere#4658])
- Not possible to allow daemonset pods with hostnetwork=true to reach another pod in the same namespaces ([eks-anywhere#4765])

## Goals

- Allow users to write Kubernetes NetworkPolicies for kube-apiserver IPs and
  enforce policy such that traffic towards those IPs is allowed by the policy.
- Allow users to write Kubernetes NetworkPolicies for node IPs and enforce
  policy such that traffic towards those IPs is allowed by the policy.
- Avoid breaking network policy behaviour for existing users who may not intend
  for their CIDR policies to select nodes within the cluster.

## Non-Goals

- Allow users to create Kubernetes NetworkPolicy objects that select Pod IP
  addresses and allow traffic for those. Users should use [Kubernetes
  podSelector] or  [Cilium Labels-based rules] to allow this traffic, as this
  provides richer semantics and better scaling properties for the number of
  Pods in the cluster.

## Proposal

### Overview

_Provide a high-level overview of the design aspects of the proposal._

Under discussion [here](https://docs.google.com/document/d/1agGJgBwCdU1Nie3pS_TA-v-vmIKBM5xJmi36uZ--Kos/edit)

## Future Milestones

### Ability to match nodes based on node labels

Add the ability for CiliumNetworkPolicy or CiliumClusterwideNetworkPolicy to leverage node selector statements in order to apply policy for traffic to/from specific nodes based on these node selectors.

This would require an approach similar to this proposal in order to associate the node's labels with the locally-allocated identities for remote nodes, as well as exposing new fields in the CNP/CCNP resources in order to specify which traffic should be selected by the policy. Due to the semantic constraints of the latter, this proposal is deferred to a future CFP.

[cilium#12277]: https://github.com/cilium/cilium/issues/12277
[cilium#16308]: https://github.com/cilium/cilium/issues/16308
[cilium#20550]: https://github.com/cilium/cilium/issues/20550
[eks-anywhere#4658]: https://github.com/aws/eks-anywhere/issues/4658
[eks-anywhere#4765]: https://github.com/aws/eks-anywhere/issues/4765

[CiliumNetworkPolicy "entities" constructs]: https://docs.cilium.io/en/stable/security/policy/language/#id3
[Kubernetes podSelector]: https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors
[Cilium Labels-based rules]: https://docs.cilium.io/en/stable/security/policy/language/#labels-based

