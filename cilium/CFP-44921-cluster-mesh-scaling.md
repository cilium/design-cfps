# CFP-44921: Workload-Controlled Sync and On-Demand Resolution for Cluster Mesh at Scale

**SIG: SIG-ClusterMesh** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-03-21

**Cilium Release:** 1.17

**Authors:** Cong Yue <cong@databricks.com>

**Status:** Draft

## Summary

This CFP proposes two complementary enhancements to Cilium Cluster Mesh that enable scaling to 100+ clusters without compromising stability or correctness:

1. **Workload-controlled sync with scope**: Workloads opt in to cross-cluster endpoint sync via a pod label (`clustermesh.cilium.io/sync: "true"`) and declare their sync scope via an annotation (`clustermesh.cilium.io/sync-scope: regional|global`). Workload owners only annotate their pods — all filtering is handled automatically by infrastructure components. The clustermesh-apiserver in the pod's own cluster (source-side) prevents non-opted-in endpoints from reaching the kvstore. The kvstoremesh in each remote cluster (consumer-side) skips `regional`-scoped endpoints from clusters in a different region. This reduces synced endpoint volume by 95–99% compared to the current all-or-nothing namespace filtering.

2. **Ad-hoc sync on datapath miss**: When the BPF datapath encounters a packet whose destination IP has no `/32` ipcache entry, it emits a new `SIGNAL_IPCACHE_MISS` signal via the existing `cilium_signals` perf event array. A userspace handler with per-IP deduplication resolves the identity from the remote cluster asynchronously and populates the `/32` entry. Combined with a scoped `toEntities: world` egress policy, the first packet proceeds with degraded (WORLD) identity while resolution happens in parallel — delivering on the principle of "slowness over failure."

A proof-of-concept implementing all proposed changes has been built, tested end-to-end on Kind clusters, and is available at [cilium/cilium#44916](https://github.com/cilium/cilium/pull/44916).

## Motivation

Cilium Cluster Mesh's current scaling ceiling is practical, not theoretical. The BPF ipcache map (`MaxEntries = 512,000`) must hold one entry per remote pod IP on every node. With 100 clusters averaging 2,000–5,000 endpoints each, the mesh-wide endpoint count approaches this limit.

Beyond the map size, the sync pipeline (etcd watches → kvstoremesh reflector → local etcd → cilium-agent) faces growing latency and reconnection storms at scale. Bootstrap times (`bootstrap_seconds`) can exceed 5 minutes when syncing hundreds of thousands of endpoints. During rolling updates of remote clustermesh-apiservers, LB failovers cause all connected kvstoremesh instances to reconnect simultaneously, creating thundering-herd re-sync bursts.

The existing namespace filtering mechanism (`clustermesh-default-global-namespace`) provides coarse reduction — all-or-nothing for every pod in a namespace. For large-scale deployments where only a fraction of pods need cross-cluster connectivity (e.g., pod flat L3 for specific ML training workloads), this is insufficient. Organizations must restructure their namespace layout to match mesh topology, coupling infrastructure concerns with application organization.

Additionally, at scale there are inevitable windows where per-pod `/32` ipcache data is incomplete or stale. Today, any missing pod IP silently gets WORLD identity, causing policy drops — even though Cilium already populates per-node pod CIDR prefix (`/24`) entries with the correct VXLAN tunnel endpoint. There is no mechanism to detect or recover from these sync gaps at the datapath level.

## Goals

* **G1**: Enable workloads to control their own cross-cluster sync participation via pod-level labels and annotations, without requiring namespace reorganization or cluster-wide infrastructure configuration changes.
* **G2**: Reduce the volume of endpoints synced through the cluster mesh pipeline by 95–99% for deployments where only a subset of workloads need cross-cluster connectivity.
* **G3**: Support regional scoping so that endpoints marked `sync-scope: regional` are only visible to clusters in the same region, further reducing per-consumer sync volume in multi-region deployments.
* **G4**: Provide a datapath-level safety net that detects ipcache misses and triggers on-demand identity resolution from the remote cluster, ensuring "slowness over failure" semantics for the first packet during sync gaps.
* **G5**: Maintain full backward compatibility — all new behaviors are opt-in via flags and annotations. Existing deployments with no annotations continue to work identically.
* **G6**: Compose cleanly with existing namespace filtering — the new label filter stacks on top of namespace filtering, and regional scoping stacks on top of label filtering.

## Non-Goals

* **NG1**: Cross-cluster service discovery or load-balancing. This proposal targets pod flat L3 connectivity (any-pod-to-any-pod by IP). Service-level sync (`service.cilium.io/global`) is orthogonal.
* **NG2**: Extending the cluster ID limit beyond 511. Identity space partitioning is unchanged.
* **NG3**: Replacing or modifying the existing namespace filtering mechanism. Workload-controlled sync is additive.
* **NG4**: Implementing a centralized control plane (e.g., xDS-based) for cluster mesh. This is a focused enhancement to the existing etcd-based sync pipeline.
* **NG5**: Holding or deferring packets in BPF. TC programs must return immediately (`TC_ACT_OK`/`TC_ACT_SHOT`); there is no `TC_ACT_DEFER`. The design accepts degraded identity on the first packet rather than dropping it.
* **NG6**: CiliumEndpointSlice (CES) support in the initial implementation. CES lacks direct pod labels, requiring either cross-referencing or filtering at the CiliumEndpoint level. CES support is deferred to a follow-up milestone.

## Proposal

### Overview

The proposal consists of four parts that form a layered filtering and resilience architecture:

```
All pods in cluster (5,000)
  │
  ▼ Namespace filter (existing): clustermesh-default-global-namespace=false
Global namespace pods (200)
  │
  ▼ Label filter (Part A): clustermesh.cilium.io/sync=true
Opted-in pods (50)              ← written to source cluster's etcd
  │
  ▼ Scope filter (Part C, consumer-side): sync-scope=regional, region match?
Pods visible to this consumer (15)  ← programmed into consumer's BPF ipcache
  │
  ▼ Ad-hoc sync (Part D): safety net for any remaining misses
First packet: WORLD identity (allowed by scoped policy) → async /32 resolution
Subsequent packets: exact pod identity
```

### Part A: Source-Side Label Filtering (clustermesh-apiserver in the pod's own cluster)

**Component**: `clustermesh-apiserver/clustermesh/root.go`

"Source-side" refers to the clustermesh-apiserver running in the same cluster as the workload — not the workload itself. The clustermesh-apiserver's `endpointSynchronizer` gains a label-based filter. On every CiliumEndpoint upsert, if label filtering is enabled (`--enable-sync-label-filter=true`), the synchronizer checks for `clustermesh.cilium.io/sync: "true"` on the endpoint's `ObjectMeta.Labels`. Endpoints without the label are converted to delete events, removing any previously synced state from the kvstore.

```go
func (es *endpointSynchronizer) upsert(ctx context.Context, key resource.Key, obj runtime.Object) error {
    endpoint := obj.(*types.CiliumEndpoint)

    if es.labelFiltered {
        if endpoint.Labels == nil || endpoint.Labels[LabelSync] != "true" {
            return es.delete(ctx, key)
        }
    }
    // ... proceed with existing sync logic
}
```

**Prerequisite**: `TransformToCiliumEndpoint()` in `pkg/k8s/factory_functions.go` currently strips `Labels` and `Annotations` (sets them to `nil`). This must be changed to preserve them so the sync label and sync-scope annotation survive the K8s → CiliumEndpoint transformation.

**Configuration**: Via `--enable-sync-label-filter` flag on the clustermesh-apiserver (default: `false`).

### Part B: SyncScope Propagation via IPIdentityPair

**Component**: `pkg/identity/identity.go`

The `IPIdentityPair` struct — the serialized form of endpoint data in the kvstore — gains a `SyncScope` field:

```go
type IPIdentityPair struct {
    // ... existing fields ...
    SyncScope string `json:"syncScope,omitempty"` // "regional" or "global" (default empty = global)
}
```

The clustermesh-apiserver reads `clustermesh.cilium.io/sync-scope` from the CiliumEndpoint's annotations and propagates it into the `IPIdentityPair` written to etcd. Using `omitempty` ensures backward compatibility: old consumers ignore the unknown field, and entries without the annotation produce no extra JSON.

### Part C: Consumer-Side Regional Filtering (kvstoremesh in each remote cluster)

**Components**: `pkg/clustermesh/kvstoremesh/regional_filter.go`, `pkg/clustermesh/kvstoremesh/remote_cluster.go`

"Consumer-side" refers to the kvstoremesh in a remote cluster that reads endpoint data from the source cluster's etcd — not the workloads in the remote cluster. The kvstoremesh reflector gains optional filter function support. A `NewRegionalEndpointFilter` constructs a filter that:

1. Deserializes the `IPIdentityPair` from the kvstore value
2. Checks the `SyncScope` field — if not `"regional"`, accepts the entry (global or unscoped)
3. For `"regional"` entries, looks up the source node's region via a `NodeRegionCache` (populated from CiliumNode data, using the standard `topology.kubernetes.io/region` label)
4. Compares against the local cluster's region (configured via `--cluster-region` flag on kvstoremesh)
5. Accepts if regions match; deletes from local kvstore if they differ

```go
func NewRegionalEndpointFilter(localRegion string, cache *NodeRegionCache) func(store.Key) bool {
    return func(key store.Key) bool {
        var pair identity.IPIdentityPair
        if err := json.Unmarshal(kv.Value, &pair); err != nil {
            return true // can't parse — accept to be safe
        }
        if pair.SyncScope != "regional" {
            return true // global or unscoped — always accept
        }
        hostRegion, found := cache.RegionForHost(pair.HostIP.String())
        if !found {
            return true // unknown region — accept to be safe
        }
        return hostRegion == localRegion
    }
}
```

The filter is injected into the syncer's `OnUpdate()` path:

```go
func (o *syncer) OnUpdate(key store.Key) {
    if o.filter != nil && !o.filter(key) {
        o.DeleteKey(context.Background(), key)
        return
    }
    o.UpsertKey(context.Background(), key)
}
```

**Design choice: filter on the consumer side, not source side.** The source-side clustermesh-apiserver writes to its own etcd without knowing which remote clusters will consume the data. Only the consumer knows its own region and can make the regional filtering decision. CiliumNode data (which includes `topology.kubernetes.io/region` labels) is always synced regardless of namespace filtering, so region information is always available.

### Part D: Ad-Hoc Sync via BPF Signal

**Components**: `bpf/lib/signal.h`, `bpf/bpf_lxc.c`, `pkg/signal/signal.go`, `pkg/datapath/signalhandler/ipcache_miss.go`

#### BPF Side

A new signal type `SIGNAL_IPCACHE_MISS` is added to the existing `cilium_signals` perf event array, following the same pattern as `SIGNAL_NAT_FILL_UP` and `SIGNAL_AUTH_REQUIRED`:

```c
enum {
    SIGNAL_NAT_FILL_UP = 0,
    SIGNAL_CT_FILL_UP,
    SIGNAL_AUTH_REQUIRED,
    SIGNAL_IPCACHE_MISS,        // NEW
};

struct ipcache_miss_msg {
    __u32 dst_ip;               // destination IP (network byte order)
    __u16 cluster_id;           // cluster context (if known from /24)
    __u16 pad;
};
```

In `handle_ipv4_from_lxc()` (`bpf_lxc.c`), when `lookup_ip4_remote_endpoint()` returns NULL (no `/32` match, falls through to `/24` with WORLD identity), the BPF program emits the signal:

```c
} else {
    *dst_sec_identity = WORLD_IPV4_ID;
    send_signal_ipcache_miss(ctx, ip4->daddr, (__u16)cluster_id);
}
```

The signal uses the existing `SEND_SIGNAL` macro (`ctx_event_output(&cilium_signals, BPF_F_CURRENT_CPU, ...)`) — a lightweight, production-proven perf event write.

#### Userspace Side

A new Hive cell (`signalhandler.Cell`) registers a handler for `SignalIPCacheMiss`:

```go
type IPCacheMissResolver interface {
    ResolveIPCacheMiss(ctx context.Context, ip net.IP, clusterID uint16) error
}

type IPCacheMissHandler struct {
    logger   *slog.Logger
    inflight sync.Map              // per-IP deduplication
    resolver IPCacheMissResolver
}

func (h *IPCacheMissHandler) handleMiss(ctx context.Context, data IPCacheMissData) error {
    ip := data.IP()
    ipStr := ip.String()

    if _, loaded := h.inflight.LoadOrStore(ipStr, struct{}{}); loaded {
        return nil // already resolving this IP
    }

    go func() {
        defer h.inflight.Delete(ipStr)
        resolveCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        h.resolver.ResolveIPCacheMiss(resolveCtx, ip, data.ClusterID)
    }()
    return nil
}
```

The `IPCacheMissResolver` interface is pluggable. In production, it queries the local kvstoremesh etcd cache for the `/32` entry and upserts it into the BPF ipcache. In tests, a mock resolver is injected.

#### Why "Allow First Packet + Async Resolve"

BPF TC programs cannot hold or defer packets — they must return immediately. There is no `TC_ACT_DEFER` or retry mechanism. The signal-and-resolve pattern is the only viable approach:

| Approach | First Packet | Subsequent Packets | Feasibility |
|----------|-------------|-------------------|-------------|
| Hold packet in BPF | N/A | N/A | Not possible (kernel limitation) |
| Drop first, async resolve | DROPPED | Correct identity | Violates "slowness over failure" |
| **Allow first (WORLD) + async resolve** | **Degraded identity** | **Correct identity** | **Proposed** |

The scoped `toEntities: world` egress policy in mesh-enabled namespaces allows the first packet through with WORLD identity. This policy is only exercised during sync gaps — at steady state, `/32` entries are already present and fine-grained identity is used.

## Impacts / Key Questions

### Impact: IPIdentityPair API Extension

The `SyncScope` field is added to `IPIdentityPair`, which is the serialized JSON format stored in etcd. This is a stable API surface.

#### Option 1: `json:"syncScope,omitempty"` (Proposed)

**Pros:**
- Fully backward compatible — old consumers ignore unknown JSON fields
- No extra bytes for endpoints without sync-scope (empty = global by default)
- Standard Go JSON handling, no special deserialization logic needed

**Cons:**
- String field allows arbitrary values; only "regional" and "global" are meaningful
- Cannot be removed without a migration once widely deployed

#### Option 2: Separate metadata key in etcd (key suffix or sibling key)

**Pros:**
- No change to `IPIdentityPair` struct
- Metadata can be updated independently of the endpoint data

**Cons:**
- Requires atomic reads of two keys per endpoint
- More complex consumer logic
- Higher etcd write amplification

**Recommendation**: Option 1. The `omitempty` tag ensures zero overhead for unscoped endpoints, and the single-key design matches the existing endpoint storage pattern.

### Impact: Preserving Labels/Annotations in TransformToCiliumEndpoint

The current `TransformToCiliumEndpoint()` strips `Labels` and `Annotations` for memory efficiency. Preserving them increases the in-memory size of each CiliumEndpoint object in the informer cache.

#### Option 1: Preserve all Labels and Annotations (POC approach)

**Pros:**
- Simple implementation
- Future-proof for additional annotation-based features

**Cons:**
- Memory increase proportional to label/annotation count per pod
- May carry labels irrelevant to cluster mesh

#### Option 2: Copy only `clustermesh.cilium.io/*` labels and annotations

**Pros:**
- Minimal memory overhead — at most 2 entries per endpoint
- Clear boundary of what the transformer preserves

**Cons:**
- Harder to extend if additional labels/annotations are needed later
- Slightly more complex transform logic

**Recommendation**: Option 2 for production. The POC uses Option 1 for simplicity, but production should selectively copy only the `clustermesh.cilium.io/` prefixed entries.

### Impact: CiliumEndpointSlice Compatibility

The POC requires `ciliumEndpointSlice.enabled: false` because CES batches multiple endpoints into a single object without preserving per-endpoint pod labels.

#### Option 1: Filter at CiliumEndpoint level, skip CES path for filtered namespaces

**Pros:**
- Simple — reuses existing CiliumEndpoint sync path
- No CES struct changes needed

**Cons:**
- Loses CES batching efficiency for mesh-enabled namespaces
- Two sync paths (CE for filtered, CES for unfiltered) increases complexity

#### Option 2: Extend CoreCiliumEndpoint in CES with sync metadata

**Pros:**
- Preserves CES batching efficiency
- Single sync path

**Cons:**
- CES API change (new fields in `CoreCiliumEndpoint`)
- Higher coordination effort with CES maintainers

**Recommendation**: Start with Option 1 in the initial release. Pursue Option 2 as a follow-up once the annotation-based filtering is proven in production.

### Key Question: Signal Flood Under Sustained ipcache Misses

If many pods simultaneously send traffic to unsynced destinations (e.g., during mass bootstrap), the BPF signal rate could be high.

**Mitigations:**
- Per-IP deduplication in the userspace handler (`sync.Map` with `LoadOrStore`)
- BPF perf events are lossy by design — dropped signals delay resolution but cause no failure
- The scoped WORLD policy ensures packets are not dropped during resolution
- Rate limiting can be added to the signal handler if needed
- Resolution timeout (5s default) prevents goroutine accumulation

### Key Question: Security Implications of Scoped WORLD Egress

The `toEntities: world` egress policy in mesh-enabled namespaces allows egress to ANY WORLD destination, not just remote cluster pods. This includes internet-bound traffic if it resolves to WORLD identity.

**Mitigations:**
- The policy is scoped to mesh-enabled namespaces only — non-global namespaces are unaffected
- Pods in mesh-enabled namespaces are already designed for cross-cluster communication
- Can be tightened with `toPorts` restrictions to limit allowed protocols/ports
- At steady state (>99% of traffic), `/32` entries are present and WORLD identity is not exercised
- Ad-hoc sync resolves the correct identity within milliseconds, minimizing the WORLD window

## Future Milestones

### Deferred Milestone 1: CiliumEndpointSlice Support

Extend the sync label and sync-scope filtering to work with the CiliumEndpointSlice path. This likely requires adding sync metadata to `CoreCiliumEndpoint` or implementing a hybrid approach where the clustermesh-apiserver filters individual endpoints out of slices before syncing.

### Deferred Milestone 2: Auto-Detection of Cluster Region

The current design requires `--cluster-region` to be explicitly configured on kvstoremesh. A future improvement would auto-detect the region from the local node's `topology.kubernetes.io/region` label, eliminating manual configuration.

### Deferred Milestone 3: Production IPCacheMissResolver

The POC defines the `IPCacheMissResolver` interface but does not include a production implementation. The production resolver needs to:
- Map the missed IP to a remote cluster (via `/24` → CiliumNode CIDR mapping)
- Query the kvstoremesh local etcd cache (or remote cluster etcd directly) for the `/32` entry
- Upsert the resolved entry into the BPF ipcache
- Emit metrics for resolution latency, success rate, and trigger frequency

### Deferred Milestone 4: Observability Enhancements

Add dedicated metrics for:
- `cilium_clustermesh_sync_label_filtered_total`: Count of endpoints filtered by sync label (source-side)
- `cilium_clustermesh_regional_filtered_total`: Count of endpoints filtered by regional scope (consumer-side)
- `cilium_ipcache_miss_signal_total`: Count of BPF ipcache miss signals received
- `cilium_ipcache_miss_resolution_duration_seconds`: Histogram of ad-hoc resolution latency
- `cilium_ipcache_miss_resolution_errors_total`: Count of failed resolutions

### Deferred Milestone 5: Conditional Cluster Identity on CIDR-Prefix Entries

When namespace filtering is **disabled** (all namespaces are global), the `/24` CIDR-prefix fallback entries could safely use a cluster-specific identity instead of WORLD. This would eliminate the need for the scoped WORLD egress policy in that configuration. However, this is mutually exclusive with namespace filtering (the recommended production configuration) due to security concerns — see the [proposal document](https://github.com/cilium/cilium/pull/44916) for the detailed analysis of why WORLD identity on `/24` entries is intentional.
