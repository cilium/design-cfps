# CFP-003: Template

**SIG: SIG-Agent, SIG-CNI**

**Begin Design Discussion:** 2023-11-17

**Cilium Release:** X.XX

**Authors:** Will Daly <widaly@microsoft.com>

**Status:** Dormant

## Summary

Enable features such as endpoint health checking and ingress controller that are currently incompatible with Cilium's delegated IPAM mode.


## Motivation

Cilium has an IPAM mode called "delegated plugin". In this mode, Cilium CNI invokes another CNI plugin to allocate and release IP addresses (see ["Plugin Delegation" in the CNI spec](https://www.cni.dev/docs/spec/#section-4-plugin-delegation) for details).

Unlike other IPAM modes, the cilium-agent daemonset is NOT involved in IPAM. However, several Cilium features require Cilium to assign itself an IP, outside the context of a CNI invocation. These features include endpoint health checking (`endpointHealthChecking.enabled=true`) and ingress controller (`ingressController.enabled=true`). When using delegated IPAM, these features are unavailable and [blocked by validation on cilium-agent startup](https://github.com/cilium/cilium/blob/70ae8d0ef536de807aab849291e5a68758cb8d4d/pkg/option/config.go#L3782).


## Goals

* Support endpoint health checking and ingress controller when using Cilium's delegated IPAM mode.
* The solution should work with any conformant CNI IPAM plugin (avoid assumptions about specifics plugins/platforms).
* The solution should *not* leak IPs, even if cilium-agent crashes and restarts.


## Non-Goals

* This CFP does not propose any changes to other IPAM modes, just to delegated IPAM.


## Proposal

### Overview

When it needs to allocate IPs for itself, cilium-agent invokes the delegated IPAM plugin directly.


### IPAM Plugin Operations

The delegated IPAM plugin supports these three operations (as of CNI spec 0.4.0):

| Operation  |   Usage                        |      Input                             |   Output                     |
|------------|--------------------------------|----------------------------------------|------------------------------|
| ADD        | Allocate an IP                 | CNI_CONTAINERID, CNI_NETNS, CNI_IFNAME | IPs (possibly IPv4 and IPv6) |
| DEL        | Release an IP                  | CNI_CONTAINERID, CNI_IFNAME            | Success/failure              |
| CHECK      | Verify that an IP is allocated | CNI_CONTAINERID, CNI_NETNS, CNI_IFNAME | Success/failure              |

(The above table is highly simplified, see the [CNI spec](https://www.cni.dev/docs/spec) for full details.)

The semantics of the above operations differ significantly from how other Cilium IPAM implementations work. In particular, Cilium's `ipam.IPAM` struct supports idempotent allocation of a specific IP using [AllocateIP](https://github.com/cilium/cilium/blob/70ae8d0ef536de807aab849291e5a68758cb8d4d/pkg/ipam/allocator.go#L47). This is used to restore IPs on cilium-agent restart, ensuring that the IP doesn't change and potentially disrupt the dataplane. This isn't possible with delegated IPAM, because:

* The required inputs do not include the IP address. By convention, some [IPAM plugins support an additional "ips" argument](https://www.cni.dev/docs/spec), but this is not universal.
* The CNI ADD operation is not idempotent. According to [the spec](https://www.cni.dev/docs/spec/#add-add-container-to-network-or-apply-modifications): "A runtime should not call ADD twice (without an intervening DEL) for the same (`CNI_CONTAINERID`, `CNI_IFNAME`) tuple."


### IP Leakage

Another challenge with delegated IPAM is releasing IPs that are no longer in use. Once CNI ADD completes successfully, the IP is allocated. In a cloud environment, this may involve configuring the cloud network to route the IP to the node. If cilium-agent repeatedly allocates IPs (for example, crashing on startup before recording that it allocated the IP), these IPs would be unavailable for pods. This can be a serious problem in some environments.

Note that it's acceptable for cilium-agent to allocate an IP without releasing it before the node is deleted. This is equivalent to someone "pulling the plug" on the node (or, in a cloud environment, deleting the VM), so any real IPAM implementation will need to handle this case anyway.


### Process for cilium-agent to invoke delegated IPAM

Given the above constraints, how can cilium-agent safely invoke the delegated IPAM plugin?

First, note that cilium-agent allocates a small number of IPs for itself. For example, if both endpoint health checking and ingress controller are enabled in a single-stack cluster, then cilium-agent needs to allocate exactly two IPv4 addresses.

Each "kind" of address that cilium-agent needs to allocate can be assigned a unique CNI_CONTAINERID, known in advance. For example, endpoint health checking might use `CNI_CONTAINERID="cilium-agent-health"`, and ingress controller might use `CNI_CONTAINERID="cilium-agent-ingress"`. This allows cilium-agent to refer to an address that may have been allocated previously without knowing the exact IP address.

The other two parameters (`CNI_NETNS` and `CNI_IFNAME`) can be set to dummy values (perhaps `CNI_NETNS="host"` and `CNI_IFNAME="eth0"`?). These are required by the CNI spec (since a delegated IPAM plugin implements the same interface as a "full" CNI plugin), but are not used by any IPAM plugins that I'm aware of.

The protocol for cilium-agent to call delegated IPAM is then relatively simple:

1. If there is an IP to restore, invoke `CNI CHECK` to ensure that the IP is still allocated. If `CNI CHECK` succeeds, then return success.
2. `CNI DEL` to ensure any previously-allocated IP is released. Continue to step 3 even if `CNI DEL` errors.
3. `CNI ADD` to allocate a new IP. If it succeeds, then use the returned IP; otherwise, return failure.


### Complications and caveats

* **CNI state**: Some IPAM plugins store state on-disk (example: host-local writes to files in /var/lib/cni/networks by default, but this can be overridden in the CNI config). These directories *must* be mounted read-write in the cilium-agent pod, otherwise IPs could be leaked or double-allocated. Since this depends on the specific delegated IPAM plugin used, the user must configure this in the Cilium chart using `extraHostPathMounts`.

* **Cilium config change**: Suppose a user first configures cilium with endpoint health checking, then disables it. This will leak one IP per IP family per node, since cilium-agent won't execute `CNI DEL` on every possible IP it might have allocated in previous configurations. I'd argue this is acceptable as long as it's documented: the IPs would eventually be released as nodes are deleted and replaced.

* **Cilium CNI version**: Current default Cilium CNI version is 0.3.1, but the `CNI CHECK` operation isn't supported until 0.4.0. The Cilium CNI code is compatible with 0.4.0, so I think it's safe to set 0.4.0 in the conflist.

* **CNI Spec 1.1 GC operation**: [CNI spec 1.1 introduces a new "GC" operation](https://github.com/containernetworking/cni/pull/1022). The idea is that the container runtime calls GC with a list of all known attachments, and the CNI plugin cleans up any attachments not in the list. The cleanup includes invoking delegated IPAM plugins to release IPs. This is a problem, since the container runtime won't know about IPs that cilium-agent allocated for itself by invoking the IPAM plugin directly. One possible solution would be for Cilium CNI's GC operation to inject IPs allocated by cilium-agent before Cilium CNI invokes the delegated IPAM plugin's GC. Unclear if this is allowed or forbidden by the CNI spec.

* **CNI conflist installation**: cilium-agent needs to read the CNI conflist, which might not yet exist if it's installed by another daemonset (e.g. when Cilium is configured with `cni.install=false`). Easy thing to do is exit with an error, but it would be better to retry or watch the conflist directory.


### Prototype

I wrote a small, hacky prototype to demonstrate that the proposed approach is possible:

https://github.com/cilium/cilium/compare/main...wedaly:cilium:delegated-ipam-cilium-agent-prototype


## Impacts / Key Questions

### Key Question: Is this compliant with the CNI spec?

The goal of the CNI spec is to define the interface between the container runtime and the CNI plugin. Invoking it directly from cilium-agent probably isn't something the spec writers ever had in mind. The main concern is that as the CNI ecosystem evolves, assumptions in this proposal will be broken.

### Key Question: Possible to move envoy to pod network?

If envoy were running in pod network as a separate daemonset, then it would get assigned an IP by the container runtime automatically. I think ingress controller / envoy is the most important feature unblocked by this CFP. I suspect moving envoy out of the host netns would greatly complicate the datapath, however.


## Future Milestones

N/A
