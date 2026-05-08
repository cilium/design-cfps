# CFP-45836: Multicast Egress as CiliumEgressGatewayPolicy Extension

**SIG: SIG-DATAPATH** (with input from SIG-POLICY)

**Begin Design Discussion:** 2026-05-07

**Cilium Release:** TBD (target: 1.20)

**Authors:** Omar Ramadan <ramadan@blockcast.net>

**Status:** Draft

**Issue:** https://github.com/cilium/cilium/issues/45836

> **Update 2026-05-08 after local validation**: the original "multicast egress is broken without this patch" framing is overclaim. The existing CEGP code path already produces `TC_ACT_REDIRECT` for multicast destinations matching a policy via the legacy `fib_lookup → fib_do_redirect → ctx_redirect` chain. This CFP is therefore narrower than originally scoped: (1) datapath simplification — bypass FIB lookup on the gateway for multicast destinations (kernel handles MAC encoding directly, no neighbor resolution needed); (2) identity-resolution defense — force `WORLD_ID` for multicast in `egress_gw_handle_packet` so a stale ipcache entry can't short-circuit the policy match; (3) a new BPF unit test that documents "multicast destinations in `destinationCIDRs` are an intended supported case." Patch is **+199 LOC across 2 files**, no userspace changes; 50/50 existing CEGP + multicast subtests pass with it applied. The PR description contains the full updated framing.

## Summary

Extend `CiliumEgressGatewayPolicy` (CEGP) so its `destinationCIDRs` field accepts multicast CIDRs (e.g., `232.0.0.0/4`) **as an explicitly supported case**. The BPF datapath gains:

1. A FIB-lookup bypass for multicast destinations on the gateway-side from-overlay path (`egress_gw_fib_lookup_and_redirect`): IPv4 multicast (`224.0.0.0/4`) and IPv6 multicast (`FF00::/8`) maps to MAC `01:00:5e:..` / `33:33:..` directly without ARP/ND, so the per-packet `fib_lookup` is wasted work. `ctx_redirect(ctx, egress_ifindex, 0)` directly.
2. An identity-force in `egress_gw_handle_packet` so multicast destinations skip the `identity_is_cluster()` short-circuit regardless of ipcache state.

No new CRD type, no new fields, no userspace controller changes. Same operator UX as unicast egress.

## Motivation

Cilium has two multicast-related features today:

1. **`multicast-enabled` (beta)** — cluster-internal pod-to-pod fanout via VXLAN. Source IP preserved. Subscribers are local pods (auto-populated from IGMP joins) or remote CiliumNodes (CLI-added). Excellent for in-cluster distribution; doesn't egress.

2. **`CiliumEgressGatewayPolicy`** — unicast egress with SNAT to a configurable egress IP, traffic-engineered through a chosen gateway node. Doesn't accept multicast destinations — they're silently dropped at the lxc egress hook.

There's no path between these for the case "publish SSM multicast from a cluster pod to an external multicast-aware network." Receivers may be anywhere — on a directly-peered AS via PIM-SSM, on the public Internet via AMT (RFC 7450) last-mile relays, or in another multicast island reached via [middle-mile AMT tunneling](https://www.ietf.org/archive/id/draft-zzhang-mboned-dynamic-internet-mcast-tunnel-00.txt). In all cases the cluster-side question is the same: how does the publisher pod's multicast packet reach the host's primary interface with a routable source IP?

Without this feature, operators have to choose between:

- `hostNetwork: true` for the publisher pod (loses pod portability — same issue as the original CEGP motivation).
- A custom hostNetwork DaemonSet that proxies multicast for each group (doesn't scale at "many groups" — one per delivery service in CDN-style use cases).
- Multus secondary network attachments with bespoke ARP / IP handling (cluster-side dependency on Multus, complex per-pod manifests).

Extending CEGP gives operators the same mental model and operator UX they already use for unicast egress, and scales cleanly to many multicast groups via existing BPF map infrastructure.

## Goals

- **Multicast destinations in CEGP `destinationCIDRs`**: pods selected by a CEGP can publish to multicast destinations matched by the policy.
- **SNAT to `egressIP` for multicast**: source IP is rewritten on egress, same as unicast — receivers see a stable, routable source.
- **Gateway-node egress**: multicast packets exit the cluster on the host eth0 of a node selected by `egressGateway.nodeSelector`, allowing operators to anchor an L2 announcement / RPF target on specific nodes.
- **No CRD changes**: same `CiliumEgressGatewayPolicy` type, same fields, same operator UX. The change is purely datapath + controller + documentation.
- **Coexist with `multicast-enabled`**: cluster-internal fanout still works; external egress works in addition.
- **Pod portability preserved**: publisher pods remain portable (no hostNetwork required).

## Non-Goals

- **Multicast routing daemon (smcroute / pimd) on cluster nodes**: out of scope. The egress-gateway node's kernel sends via the default route once ifindex is correct; PIM/IGMP downstream of the host's eth0 is the upstream switch / edge router's responsibility.
- **PIM-SSM signaling within the cluster**: Cilium remains source-only for egress. Receivers' joins propagate upstream of the cluster (router/switch concern), not into BPF.
- **Multicast ingress from external networks to cluster pods**: the symmetrical "deliver inbound multicast to a pod" path is a separate enhancement (could be a follow-up CFP — see Future Milestones).
- **Replacement of the existing `multicast-enabled` feature**: this is a complementary capability, not a replacement.
- **AMT, mGRE, BIER, or any specific encapsulation between AS edges**: Cilium produces a regular IPv4 multicast packet on the gateway node's eth0. What happens upstream of eth0 is the network operator's choice (native PIM-SSM, AMT relay, MP-BGP multicast SAFI, etc.).

## Proposal

### Overview

The CFP turns out to be a smaller datapath patch than initially scoped. Reading the existing code:

- `cilium_egress_gw_policy_v4` is **already an `LPM_TRIE`** (`bpf/lib/egress_gateway.h:39`) — a `232.0.0.0/4` policy entry would be a single LPM key. No map schema changes.
- `pkg/egressgateway/policy.go` already iterates `dstCIDRs []netip.Prefix` without filtering — multicast CIDRs are silently accepted today by the userspace path.
- `bpf/bpf_lxc.c:1715` already has multicast detection (`IN_MULTICAST(daddr)`) for the in-cluster `multicast-enabled` feature, with **fall-through** to standard egress when no in-cluster subscribers are registered. Pod-side hook needs no changes.
- `bpf/bpf_host.c:1425` calls `egress_gw_handle_request` on the host's egress path, which already calls `egress_gw_handle_packet` → `lookup_ip4_egress_gw_policy` (LPM). The lookup already returns the right policy entry for multicast destinations *if* one is in the map.

What's actually missing is:

1. **Confirm `lookup_ip4_remote_endpoint(multicast_ip, 0)` returns a sensible identity** (probably WORLD; verify) so that `identity_is_cluster()` doesn't short-circuit `egress_gw_handle_packet` early for multicast destinations.
2. **Egress-gateway-node BPF**: on receiving the VXLAN-encapsulated multicast packet, SNAT source to `egressIP` (existing path works), then `clone_redirect` to the egress ifindex instead of going through `egress_gw_fib_lookup_and_redirect` (FIB lookup may not be the right call for multicast; needs verification).
3. **Optional CT bypass for multicast** — see Key Question 2.
4. **No userspace controller changes** beyond optional logging/metrics.

This is materially smaller than the original 4–6 week scope; might be 2–3 weeks once verification of the existing path is done.

### CRD: no changes

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: multicast-egress
spec:
  destinationCIDRs:
    - 232.0.0.0/4   # SSM range — newly meaningful with this CFP
  egressGateway:
    egressIP: 192.0.2.128
    nodeSelector:
      matchLabels:
        egress-gateway: "true"
  selectors:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: media-publishers
      podSelector:
        matchLabels:
          app: ssm-publisher
```

Same type. Same fields. The semantic addition is the BPF behavior when `destinationCIDRs` includes multicast prefixes.

### BPF map: already LPM (no schema change)

`cilium_egress_gw_policy_v4` is already a `BPF_MAP_TYPE_LPM_TRIE` (`bpf/lib/egress_gateway.h:39`). A single entry for `232.0.0.0/4` (using `EGRESS_PREFIX_LEN_V4(4) = 36` as the key prefix length) covers the entire SSM range with one map entry. No map-schema migration needed.

The userspace controller (`pkg/egressgateway/policy.go:285`) already iterates `dstCIDRs []netip.Prefix` and inserts each as an LPM entry — multicast CIDRs would be inserted by exactly the same code path. No filtering is currently in place to reject multicast CIDRs in `parseCEGP`; the destination side of the LPM trie accepts them transparently. The behavior gap is purely in the datapath.

### Origin-node BPF (`bpf/bpf_host.c` host-egress path; `bpf/lib/egress_gateway.h` lookup)

The pod-side `from-container` hook in `bpf/bpf_lxc.c:1715` already has multicast handling for the in-cluster `multicast-enabled` feature: if the destination is multicast (`IN_MULTICAST(daddr)`) and the group has subscribers in `cilium_mcast_subscriber_v4_*`, it tail-calls `CILIUM_CALL_MULTICAST_EP_DELIVERY`. **Otherwise it falls through** to the standard egress path (`__per_packet_lb_svc_xlate_4` / `tail_ipv4_ct_egress`). For our case (pod-to-external multicast egress, no in-cluster subscribers for the group), the fall-through is the path we want.

The actual EGW request path runs in `bpf_host.c:1425` via `egress_gw_handle_request()`. That function:

1. Calls `ct_extract_ports4` and `ct_is_reply4` (line 501-512). Replies short-circuit; outbound packets continue.
2. Resolves `src_sec_identity` via `__lookup_ip4_endpoint(saddr)` and `dst_sec_identity` via `lookup_ip4_remote_endpoint(daddr, 0)` (line 514-520).
3. Calls `egress_gw_handle_packet(tuple, dst_sec_identity, &gateway_ip)` (line 524).
4. Inside `egress_gw_handle_packet` (line 258), the early return is `if (identity_is_cluster(dst_sec_identity)) return CTX_ACT_OK;` — multicast destinations like `232.X.Y.Z` resolve to WORLD identity (not cluster), so we proceed.
5. `egress_gw_request_needs_redirect_hook` calls `lookup_ip4_egress_gw_policy(saddr, daddr)` (line 116) — LPM trie lookup on `(saddr, daddr_prefix)`.

For our case, the LPM lookup already returns the right policy entry if a `232.0.0.0/4` (or matching prefix) entry was inserted. The function returns `CTX_ACT_REDIRECT` with `gateway_ip` set, and the caller in `bpf_host.c` forwards the packet to the gateway node via the existing VXLAN encapsulation path.

**This means most of the origin-node datapath needs no changes.** The remaining work is:

- Verify `lookup_ip4_remote_endpoint(multicast_ip, 0)` returns a usable identity (or that `egress_gw_handle_packet` doesn't reject when it returns NULL/0). Add explicit handling for "destination is multicast → treat as WORLD identity" if needed.
- Confirm the existing CT path doesn't drop multicast at `ct_create` or related — multicast doesn't have replies, but creating a CT entry on each packet is wasteful. Add a small "skip CT for multicast" carve-out if measurement shows it matters.

### Egress-gateway-node BPF (`bpf/bpf_overlay.c` from-overlay; `bpf/bpf_host.c` to-netdev)

When the encapsulated multicast packet arrives at the gateway node via VXLAN, decapsulation happens in `bpf_overlay.c`. The existing SNAT path (`egress_gw_snat_needed_hook` at `bpf_overlay.c:359`) is called for IPv4 with the source/destination addresses. For multicast destinations, the existing SNAT logic in `egress_gw_snat_needed` (`bpf/lib/egress_gateway.h:155`) just calls the same LPM lookup and returns the egress IP — no multicast-specific logic blocks it.

After SNAT, the packet exits via the gateway node's primary interface. For multicast destinations, the kernel handles the IPv4-to-MAC mapping (`01:00:5e:X.Y.Z`) and delivers to the wire. The existing `egress_gw_fib_lookup_and_redirect` (`bpf/lib/egress_gateway.h:81`) does an FIB lookup which may return `BPF_FIB_LKUP_RET_NO_NEIGH` for multicast (no neighbor entry needed for multicast MAC) — needs verification. If FIB lookup is unsuitable for multicast, we substitute a simpler `clone_redirect(egress_ifindex, 0)` call.

**Specific changes required on the gateway node:**

- Add a multicast-aware branch in `egress_gw_snat_needed_v4`/`_v6` (or in the calling code in `bpf_overlay.c`) that, after SNAT, takes the `clone_redirect(egress_ifindex, 0)` shortcut instead of `egress_gw_fib_lookup_and_redirect` for IPv4 multicast destinations.
- Bypass the existing nat6 / nat4 conntrack helpers (`bpf/lib/nat.h`) for multicast — multicast NAT doesn't track flows the same way.

### Userspace controller (`pkg/egressgateway/manager.go` + `policy.go`)

The existing `parseCEGP` (`pkg/egressgateway/policy.go:344`) parses `destinationCIDRs` as `[]netip.Prefix` without filtering — multicast CIDRs are accepted today silently. The map population code (`policy.go:285`) iterates these and inserts each as an LPM entry. So the userspace path needs minimal changes:

- Optionally validate that mixed unicast+multicast `destinationCIDRs` in a single policy is allowed (currently no such validation; default behavior should remain "allow, operators may want separate policies for clarity").
- Optionally surface a metric or log when multicast policies are reconciled, so operators can debug coverage.

The `excludedCIDRs` field already supported by CEGP could similarly accept multicast CIDRs as a way to exempt specific groups from egress.

### Coexistence with `multicast-enabled`

The two features operate on disjoint subscriber-sets but share the policy / map infrastructure:

| Feature | Subscriber type | Source IP | Destination |
|---|---|---|---|
| `multicast-enabled` (existing) | Local pod (IGMP join) or remote CiliumNode (CLI) | Preserved | Pod / pod / VXLAN |
| Multicast egress (this CFP) | Egress-gateway node's eth0 (implied by `egressGateway.nodeSelector`) | SNAT'd to `egressIP` | Host eth0 (clone_redirect) |

When both are enabled and a publisher pod sends to a group with both local-pod subscribers AND a CEGP egress entry, both deliveries happen: the packet is replicated to local pods (existing path) and emitted on the egress-gateway node's eth0 (new path). The two paths don't conflict because subscriber types are disjoint.

### TTL handling

Multicast TTL (IP-layer) is typically low — IP_MULTICAST_TTL defaults to 1 in many applications. For external SSM tree distribution, TTL needs to be high enough to traverse the path to receivers (multiple hops, possibly across AS boundaries).

Two options:

- **Trust the publisher**: application sets TTL via `IP_MULTICAST_TTL` socket option. BPF doesn't touch it.
- **Operator override**: extend CEGP with optional `multicastTTL` field. BPF rewrites TTL on egress.

This CFP defaults to "trust the publisher" — minimal datapath change, fits the unicast CEGP model where conntrack/TTL aren't manipulated. Operator override can be added later if real deployments need it.

### Testing

- **BPF unit tests** (`bpf/tests/`): pattern after `mcast_tests.c`. Cover the origin-node redirect path (`egress_gw_handle_packet` for multicast destinations resolved as WORLD identity), egress-gateway-node SNAT path (`egress_gw_snat_needed` with multicast destination → `clone_redirect`), CT bypass, coexistence with `multicast-enabled` (group has both local subscribers and CEGP egress entry).
- **Go-level egress gateway suite**: extend `pkg/egressgateway/manager_privileged_test.go`. Cover policy reconcile with multicast `destinationCIDRs`, mixed unicast+multicast policies, excludedCIDRs containing multicast.
- **End-to-end** (Cilium e2e harness): publisher pod (BPF policy: cluster network endpoint) → CEGP → egress-gateway node → host network receiver via tcpdump or `ip mroute`. Verify SNAT'd packets, verify existing unicast CEGP regressions clean. The existing CEGP e2e suite is the template.

## Impacts / Key Questions

### Key Question 1: Does `lookup_ip4_remote_endpoint` return a sensible identity for multicast destinations?

`egress_gw_handle_packet` early-returns `CTX_ACT_OK` if the resolved destination identity is in-cluster (`identity_is_cluster(dst_sec_identity)`). For multicast IPs, ipcache lookup may return:

- WORLD identity → proceed (correct).
- UNKNOWN / 0 / NULL → `identity_is_cluster(0)` is false, so still proceeds.
- An incorrectly-cached identity (e.g., from a stale lookup) → could trigger the cluster-internal short-circuit incorrectly.

Mitigation: add an explicit `IN_MULTICAST(daddr) → use WORLD identity` carve-out before the existing identity check. Small change, makes the contract explicit. Maintainers should weigh in on whether this fits the existing identity model.

### Key Question 2: CT bypass for multicast — necessary or just nice?

The current `egress_gw_handle_request` calls `ct_extract_ports4` + `ct_is_reply4` (`bpf/lib/egress_gateway.h:501-512`). For UDP multicast egress, this works (`ct_extract_ports4` handles UDP) but creates per-packet CT lookups for traffic that doesn't have replies. At "many groups × many packets" scale this could be measurable overhead.

#### Skip CT for multicast (preferred)

##### Pros
- Eliminates per-packet CT lookup overhead for multicast egress.
- Matches the semantic reality: multicast is one-to-many, no replies to track.
- Hubble flow events can be sourced from the policy verdict path instead of CT.

##### Cons
- One more datapath branch.
- If anyone relies on multicast CT entries for some unforeseen purpose, they regress.

#### Keep current CT path

##### Pros
- Smallest diff. No new branch.

##### Cons
- Wasted CPU on every multicast packet.
- CT entries for multicast destinations may pollute CT maps without ever expiring properly (they don't see replies, so the heuristics that age out CT entries based on observed behavior are off).

Recommend skip CT for multicast; ask maintainers if there's a reason to keep it.

### Key Question 3: Should TTL be operator-controllable?

For publishers using mature SSM stacks (FFmpeg, GStreamer, applications using libmmt), `IP_MULTICAST_TTL` is set explicitly by the application and should be trusted. For naive applications or operators wanting a uniform TTL policy, an optional `multicastTTL` CRD field would override.

Defer to a follow-up if requested. This CFP recommends "trust the application."

### Impact: Conntrack bypass

Multicast egress bypasses Cilium's conntrack. This is correct semantically (multicast is one-to-many, not connection-oriented) but means:

- Multicast traffic is invisible to per-connection metrics that rely on CT state.
- Hubble flow visibility for multicast egress would need to come from the BPF policy verdict, not CT events.

Acceptable for v1. If observability is a concern, expose multicast egress packet counts via per-policy metrics (similar to existing CEGP).

### Impact: ARP / L2 announcement of `egressIP`

For receivers' RPF check (running on routers downstream of the gateway node), the unicast `S=egressIP` must be reachable from those routers. This is achieved by:

- Cilium L2 announcement for `egressIP` via LB-IPAM (existing).
- Operator-side configuration ensuring `egressIP` is bound to the gateway node's primary interface, with appropriate ARP suppression for nodes that aren't the L2 leader.

This isn't new work for this CFP — it's the same prerequisite that unicast CEGP egress IPs need today. Documenting it explicitly is enough.

### Impact: Cilium image size / BPF complexity

The datapath additions are small (~300–500 LOC of BPF, similar of Go controller code), but BPF programs have a complexity-budget that increases with each new code path. Worth measuring during implementation that we don't push any of `bpf_lxc` / `bpf_overlay` over the verifier limit.

## Future Milestones

### Multicast ingress to pods from external networks

Symmetrical to this CFP. Pods join a group via IGMP; Cilium's BPF maps the `(S, G)` to the pod endpoint and delivers inbound multicast received on the host's eth0. Useful for SSM receivers running as cluster pods (e.g., a video encoder consuming a multicast feed). Out of scope for this CFP because the egress side is the more pressing operational gap.

### IPv6 multicast (`FF00::/8`)

The proposal here is IPv4-first. IPv6 multicast (e.g., MLD-Snooping, embedded RP) follows the same datapath shape but with different address-family handling in BPF. Should be a parallel follow-up.

### Multicast TTL operator override

If real deployments need uniform TTL policy, add `CiliumEgressGatewayPolicy.spec.multicastTTL`. Defer until requested.

### Hubble flow visibility for multicast egress

Per-policy packet counters surfaced via existing metrics machinery; flow events emitted from BPF policy-verdict path. Defer until users ask.

### Composition with `cilium/cilium#38341` (Multigateway support for Egress Gateway)

The multi-gateway HA proposal extends CEGP with multiple gateway slots. The two CFPs should compose cleanly — multicast egress with multi-gateway means HA failover for multicast publishers. Worth aligning datapath changes between the two.
