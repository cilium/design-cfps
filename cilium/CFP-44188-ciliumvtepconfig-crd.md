# CFP-44188: CiliumVTEPConfig CRD for Dynamic VTEP Management

**SIG:** SIG-Datapath ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-04-07

**Cilium Release:** 1.20

**Authors:** Murat Parlakisik <parlakisik@gmail.com>

**Status:** Draft

## Summary

Replace the static CLI flag-based VTEP configuration with a cluster-scoped
`CiliumVTEPConfig` CRD (`cilium.io/v2alpha1`) that supports dynamic updates and
per-node assignment via `nodeSelector`, plus a companion per-node
`CiliumVTEPNodeConfig` for per-VTEP-endpoint status reporting. This enables
production use cases where operators want simple overlay connectivity to an
external gateway as an alternative to BGP or L2 announcements — a different
connectivity *mechanism*, suited to environments where those advertise-based
approaches don't apply.

## Motivation

The VTEP integration has been in beta since its introduction in
[PR #17370](https://github.com/cilium/cilium/pull/17370) (Cilium 1.12). The
original design uses static CLI flags (`--vtep-endpoint`, `--vtep-cidr`,
`--vtep-mac`, `--vtep-mask`) baked into the Cilium ConfigMap at install time.
This has several operational problems:

1. **Agent restarts required for any change.** Adding, removing, or modifying a
   VTEP endpoint requires updating the ConfigMap and restarting every Cilium
   agent in the cluster, causing datapath disruption.

2. **Single mask for all CIDRs.** The `--vtep-mask` flag applies one prefix
   length to every VTEP CIDR, preventing mixed prefix lengths (e.g., `/24` and
   `/16` in the same cluster).

3. **No per-node differentiation.** Every node gets the same VTEP
   configuration. In multi-zone or multi-site deployments, different nodes
   need to reach the same external CIDR via different VTEP gateways.

4. **No operational visibility.** There is no way to determine whether a VTEP
   endpoint is successfully programmed in the BPF map or if configuration
   errors exist.

5. **No CI/CD integration**. The feature is still in beta stage due to support and e2e test cases .

These limitations block adoption in environments where VTEP integration would
otherwise be the simplest and most natural connectivity solution.

### The Case for Overlay-to-Gateway Simplicity

VTEP is not equivalent to BGP or L2 announcements, and it isn't trying to be.
Those features dynamically *advertise* the cluster's own addresses so the
surrounding network learns how to send traffic *toward* Cilium. VTEP works the
other way and by a different mechanism: it gives pods a statically-configured,
bidirectional overlay tunnel to a *known* external gateway, advertising nothing.

That makes VTEP the natural fit precisely where the advertise-based approaches
don't apply:

* **No BGP-speaking upstream to peer with.** The agent has no neighbor to
  establish a session with. Cilium's own L2 announcement docs scope that feature
  to "on-premises deployments within networks without BGP based routing"; VTEP
  fills the same gap without requiring BGP either.
* **No L2 adjacency to the destination.** L2 announcements are ARP/NDP-based and
  only work within the same L2 network. VTEP reaches an external gateway over a
  routed underlay.

In those environments, pods send traffic via the existing VXLAN overlay directly
to an external VTEP endpoint — no BGP sessions to configure and maintain, no L2
announcement policies, no route redistribution. The Cilium agent encapsulates
traffic destined for the configured external CIDRs and sends it to the VTEP
endpoint, and return traffic comes back through the same tunnel.

## Goals

* Replace static CLI flags with a `CiliumVTEPConfig` CRD (`v2alpha1`) for VTEP
  configuration
* Support dynamic add/update/remove of VTEP endpoints without agent restarts
* Enable per-node VTEP assignment via `nodeSelector` for multi-zone deployments
* Provide per-node, per-VTEP-endpoint status reporting (synced, errors, last
  sync time) via a companion per-node `CiliumVTEPNodeConfig`, without agents
  writing to a shared object
* Support variable prefix lengths per VTEP CIDR (via BPF LPM Trie)
* Offer simple overlay-to-gateway connectivity as an alternative connectivity
  mechanism for environments without BGP or L2 announcement infrastructure

## Non-Goals

* Multi-VNI support — the existing VNI=2 / world-identity model is preserved
* IPv6 VTEP endpoints — IPv4 only, consistent with current VTEP support
* Changes to the VXLAN encapsulation format or behavior
* Changes to VTEP's datapath integration with other features (masquerade, egress
  gateway, encryption, L7 proxy). This CFP preserves the existing integration
  semantics; improvements are tracked as follow-up work (see
  [Impact: Integration with Other Datapath Features](#impact-integration-with-other-datapath-features))


## Proposal

### Overview

![VTEP Architecture](../cilium/images/CFP-44188-vtep-connectivity.png)

Worker nodes can be independently
assigned to different VTEP endpoints using `nodeSelector`. The red and blue arrows represent traffic
from different `CiliumVTEPConfig` objects — each config targets a subset of
nodes via label selectors and directs their VXLAN-encapsulated traffic to
the appropriate external VTEP endpoint. This per-node assignment is what
enables multi-zone and multi-site deployments where each group of nodes has
its own local gateway.

The design introduces four components:

1. **CiliumVTEPConfig CRD** (`v2alpha1`) — a cluster-scoped custom resource that
   declares VTEP endpoints with optional node targeting via `nodeSelector`. It
   carries no `.status`.
2. **CiliumVTEPNodeConfig CRD** (`v2alpha1`) — a cluster-scoped, per-node custom
   resource (one object per node, `metadata.name = node.Name`), created and
   populated by the Cilium **operator** by evaluating each `CiliumVTEPConfig`'s
   `nodeSelector` against the node. The **agent** is the sole writer of its own
   node's `.status`. This mirrors the `CiliumBGPNodeConfig` pattern.
3. **VTEPReconciler** — an agent-side controller that watches only its own
   `CiliumVTEPNodeConfig`, reconciles the BPF map and Linux routes, and reports
   per-endpoint status back to that object.
4. **BPF LPM Trie map** — replaces the existing Hash map to support
   variable-length prefix matching.

### CRD API

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumVTEPConfig
metadata:
  name: zone-a
spec:
  nodeSelector:                              # optional
    matchLabels:
      topology.kubernetes.io/zone: "zone-a"
  vtepEndpoints:
  - name: dc1-router
    cidr: "10.1.1.0/24"
    tunnelEndpoint: "10.169.72.236"
    mac: "82:36:4c:98:2e:56"
  - name: dc1-lb
    cidr: "10.2.0.0/16"
    tunnelEndpoint: "10.169.72.237"
    mac: "aa:bb:cc:dd:ee:01"
```

**Key design decisions:**

| Decision | Rationale |
|---|---|
| `v2alpha1` API | Brand-new, unreviewed surface; alpha carries no compatibility guarantee, matching the `CiliumL2AnnouncementPolicy` precedent |
| Cluster-scoped (not namespaced) | VTEP config is infrastructure-level, managed by platform teams |
| `nodeSelector` on the config, not per-endpoint | Matches the physical topology model: a set of endpoints belongs to a site/zone |
| No `.status` on `CiliumVTEPConfig` | Status lives on the per-node `CiliumVTEPNodeConfig` so agents never contend on a shared object |
| Max 8 VTEP endpoints per node | BPF map size constraint; the real limit is per-node across all matching configs, validated and reported on `CiliumVTEPNodeConfig` |
| `shortName: cvtep` | Quick operational access: `kubectl get cvtep` |
| Per-endpoint named entries with `+listType=map` | Enables strategic merge patch for individual endpoint updates |

**Validation:**

All fields use kubebuilder validation markers:
- `tunnelEndpoint`: IPv4 address (`Format=ipv4`)
- `cidr`: IPv4 CIDR notation (e.g., `10.1.1.0/24`)
- `mac`: MAC address (colon-separated hex)
- `name`: DNS label format (lowercase alphanumeric, hyphens, 1-63 chars)
- `vtepEndpoints`: MinItems=1, MaxItems=8

### Operator: per-node resolution

The Cilium operator watches `CiliumVTEPConfig` objects and node labels. For each
node it evaluates every config's `nodeSelector`, resolves the set of VTEP
endpoints that apply, and detects **CIDR conflicts** — if the same CIDR comes
from more than one matching config, neither is applied and the conflict is
recorded. It writes the resolved endpoints into that node's
`CiliumVTEPNodeConfig.spec`, creating the object with `metadata.name = node.Name`
and an `ownerReference` for cascade garbage-collection. Centralizing
`nodeSelector` evaluation in the operator means agents do not each watch every
`CiliumVTEPConfig`.

### VTEPReconciler (agent)

The reconciler runs as a Hive cell in each Cilium agent and watches **only its
own** `CiliumVTEPNodeConfig`.

**Reconciliation flow:**

1. On startup and on each update, the agent reads the resolved VTEP endpoints
   from its own `CiliumVTEPNodeConfig.spec`.

2. It computes the **desired state** — a map of normalized CIDR → VTEP endpoint.

3. It diffs desired state against **last-applied state** and performs
   incremental BPF map updates:
   - New CIDRs → `UpdateEntry()`
   - Changed endpoints → `UpdateEntry()` (overwrite)
   - Removed CIDRs → `DeleteByCIDR()`

4. It updates Linux routing table entries for the VTEP CIDRs (routing table 202).

5. **After** the local BPF map sync completes, it writes per-endpoint status
   (synced, errors, last sync time) and a `Ready` condition back to **its own**
   `CiliumVTEPNodeConfig.status`. No agent ever writes to a shared object.

**Node label change handling:** label changes are observed by the operator,
which updates the affected `CiliumVTEPNodeConfig.spec`; the agent then reconciles
the BPF map to match.

**Benefits:**
- Each endpoint can use a different prefix length (`/16`, `/24`, `/25`, etc.)
- The BPF LPM trie performs longest-prefix-match automatically in the datapath
- Removes the `--vtep-mask` global setting entirely

### Status Reporting

Status is reported **per node** on `CiliumVTEPNodeConfig` — written only by the
agent on that node, only after its local BPF map sync completes. The
cluster-scoped `CiliumVTEPConfig` has no `.status`, so no two agents ever contend
on the same object. This avoids the kube-apiserver load and write conflicts that
result from many agents updating a single shared object's status.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumVTEPNodeConfig
metadata:
  name: worker-zone-a-1            # == node name
  ownerReferences:
  - apiVersion: cilium.io/v2
    kind: CiliumNode
    name: worker-zone-a-1
spec:
  vtepEndpoints:                   # resolved by the operator from matching configs
  - name: dc1-router
    cidr: "10.1.1.0/24"
    tunnelEndpoint: "10.169.72.236"
    mac: "82:36:4c:98:2e:56"
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-04-07T10:00:00Z"
    reason: AllEndpointsSynced
    message: "All 1 VTEP endpoints synced to BPF map"
  vtepEndpointStatuses:
  - name: dc1-router
    synced: true
    lastSyncTime: "2026-04-07T10:00:00Z"
```

Operators get true per-node visibility — "synced on zone-a, failing on zone-b" is
representable, which a single shared status object cannot express:

```shell
$ kubectl get ciliumvtepnodeconfig
NAME              READY   AGE
worker-zone-a-1   True    1h
worker-zone-b-1   True    1h
```

### Helm Integration

VTEP is enabled via Helm with a single toggle:

```shell
helm upgrade cilium cilium/cilium \
    --namespace kube-system \
    --reuse-values \
    --set vtep.enabled=true
```

This registers the `CiliumVTEPConfig` and `CiliumVTEPNodeConfig` CRDs. VTEP
endpoints are then configured by applying `CiliumVTEPConfig` objects.

### Per-Zone VTEP Endpoints: Worked Example

Two zones with different VTEP gateways for the same destination CIDR:

```yaml
# Zone-A nodes route 10.200.0.0/16 via VTEP gateway 10.100.1.1
apiVersion: cilium.io/v2alpha1
kind: CiliumVTEPConfig
metadata:
  name: zone-a
spec:
  nodeSelector:
    matchLabels:
      topology.kubernetes.io/zone: "zone-a"
  vtepEndpoints:
  - name: gw-a
    cidr: "10.200.0.0/16"
    tunnelEndpoint: "10.100.1.1"
    mac: "aa:bb:cc:00:01:01"
---
# Zone-B nodes route 10.200.0.0/16 via VTEP gateway 10.100.2.1
apiVersion: cilium.io/v2alpha1
kind: CiliumVTEPConfig
metadata:
  name: zone-b
spec:
  nodeSelector:
    matchLabels:
      topology.kubernetes.io/zone: "zone-b"
  vtepEndpoints:
  - name: gw-b
    cidr: "10.200.0.0/16"
    tunnelEndpoint: "10.100.2.1"
    mac: "aa:bb:cc:00:02:01"
```

Both zones route traffic to `10.200.0.0/16` but via their local VTEP gateway.
When a new zone comes online, the
operator applies a new `CiliumVTEPConfig`

## Impacts / Key Questions

### Impact: Existing VTEP Users

The CLI flags (`--vtep-endpoint`, `--vtep-cidr`, `--vtep-mac`, `--vtep-mask`,
`--vtep-sync-interval`) have been removed. Users must migrate to the CRD.
The migration is straightforward: each set of CLI flag values maps to one
`CiliumVTEPConfig` object with a single VTEP endpoint.

### Impact: Integration with Other Datapath Features

This CFP changes how VTEP is *configured*, not how it composes with the rest of
the datapath — it inherits today's behavior. VTEP matches the destination and
performs encap + redirect on the from-lxc path, which short-circuits the
`to-netdev` hooks on the local node. Concretely, today:

* **Masquerade:** VTEP/overlay traffic is excluded from SNAT.
* **Encryption (WireGuard / IPsec):** not applied by Cilium on the local node for
  VTEP-encapsulated traffic; if the underlay needs confidentiality it must be
  provided at the underlay.
* **Egress gateway:** for an overlapping destination CIDR, VTEP takes precedence
  today by being earlier in the pipeline.
* **L7 proxy:** VTEP route installation currently requires `EnableL7Proxy=true` —
  the L7-proxy path relocates the kernel's local routing rule, which is what lets
  the VTEP routing rule be consulted.

Explicit conflict detection with egress gateway, decoupling VTEP routes from the
L7 proxy, and an end-to-end encryption story are out of scope here and tracked as
follow-up work. The per-node `CiliumVTEPNodeConfig` status is the natural place to
surface per-node integration/conflict information when that work lands.

### Impact: BPF Map Type Change

Changing from Hash to LPM Trie alters the map's behavior:

| Property | Hash | LPM Trie |
|---|---|---|
| Lookup semantics | Exact match | Longest prefix match |
| Key size | 4 bytes | 8 bytes (4 prefix + 4 IP) |
| Preallocation | Supported | Not supported (`BPF_F_NO_PREALLOC` required) |
| Max entries | 8 | 8 |

The LPM Trie is strictly more capable. Existing configurations with uniform
prefix lengths produce identical routing behavior.

## Future Milestones

### Graduating VTEP to GA

With CRD-based management, status reporting, and CI conformance tests, the
VTEP feature has a clearer path to GA status. The CRD API provides the
validation, observability, and operational tooling expected of a GA feature.

### Toward a Generic Routing CRD

`CiliumVTEPConfig` ships in `v2alpha1`, which carries no compatibility guarantee.
If a more generic "routing" CRD emerges, VTEP can fold into it (e.g.
`CiliumVTEPConfig` → `CiliumRouting{kind: VTEP}`) without breaking users — with
VTEP serving as the first concrete in-tree consumer that validates the generic
design.
