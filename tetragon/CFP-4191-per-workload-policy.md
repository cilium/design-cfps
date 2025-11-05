# CFP-4191: Tetragon per-workload policies

**SIG:** SIG-TETRAGON

**Begin Design Discussion:** 2025-11-05

**Tetragon Release:** X.XX

**Authors:** Andrea Terzolo <andrea.terzolo@suse.com>, Kornilios Kourtis <kkourt@cisco.com>

**Status:** Draft

## Summary

Today, with Tetragon it is not possible to share a common enforcement logic across many Kubernetes workloads. For each workload users need to create a separate `TracingPolicy` with the same enforcement but with different values. This design proposes a new model to decouple the enforcement logic from per‑workload values, enabling significant reductions in eBPF programs and map memory usage.

## Motivation

The current approach of one `TracingPolicy` per workload leads to:

- Program scaling limits: At least one eBPF program per policy. In clusters with hundreds of workloads, this may add latency when the hook is the hot path.
- Map memory scaling: Tens of maps per policy and several large maps (for example, policy filters, socktrack_map, override_tasks) lead to multi‑MB memlock per policy.
- Operational friction: Logic is identical but values differ (for example, allowed binaries), yet the model duplicates programs and maps.

## Goals

- Decouple shared enforcement logic from per‑workload values to avoid linear growth in programs and map memory as clusters scale.
- Retain TracingPolicy expressiveness (selectors, filters, actions).

## Non-Goals

- TBD

## Proposal

### Overview

This design is accompanied by a [proof of concept](https://github.com/cilium/tetragon/issues/4191). It introduces two concepts: templates and bindings.

A template is a `TracingPolicy` that declares variables populated at runtime rather than hardcoded at load time. Selectors reference these variables by name.

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "block-process-template"
spec:
  variables:
  - name: "targetExecPaths"
    type: "linux_binprm" # this could be used for extra validation but it's probably not strictly necessary
  kprobes:
  - call: "security_bprm_creds_for_exec"
    syscall: false
    args:
    - index: 0
      type: "linux_binprm"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        valuesFromVariable: "targetExecPaths"
```

Deploying a template alone has no runtime effect because it lacks concrete values for comparisons.

A binding is a new resource (for example, `TracingPolicyBinding`) that supplies concrete values for a template’s variables and targets specific workloads through a selector. The selector is not the same as a podSelector; it is intended to be mutually exclusive across bindings.

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyBinding
metadata:
  name: "block-process-template-values-1"
spec:
  policyTemplateRef:
    name: "block-process-template"
  exclusiveSelector: # it should be mutually exclusive across bindings
    matchLabels:
      app: "my-app-1"
  bindings:
  - name: "targetExecPaths"
    values:
    - "/usr/bin/true"
    - "/usr/bin/ls"
```

Policy logic becomes active only when a `TracingPolicyBinding` is present. The binding populates eBPF maps with the specified values for cgroups that match its selector.

### Details

- When a template is deployed, Tetragon loads the same eBPF programs and maps as today. Additionally, it creates a `BPF_MAP_TYPE_HASH` map (`cg_to_policy_map`) to map cgroup IDs to binding `policy_id`s. Initially, the map is empty, so the template has no effect.
- When a binding is deployed:
  - It receives a new `policy_id`.
  - For each cgroup matching its selector, Tetragon inserts a (`cgroup_id` → `policy_id`) entry into `cg_to_policy_map`. If a cgroup already has a binding for that template, it is rejected.
  - This mapping activates the template logic for the targeted cgroups.
  - Binding values are stored in `BPF_MAP_TYPE_HASH_OF_MAPS` instances (`pol_str_maps_*`). This implementation is very specific to string/charbuf/filename types and the eq/neq operators, but the concept can be extended to other types/operators, more on this later.
  - These maps are keyed by `policy_id` (looked up via `cg_to_policy_map`). The value is a hash set of strings sourced from the binding, using the same 11‑bucket sizing technique as existing `string_maps_*`.

### Results

This design enables:

1. Single eBPF program: One shared program can serve many bindings (for example, 512–1024+), as they reference the same template. This drastically reduces programs loaded into the kernel.
2. Low memory overhead: Per‑binding cost is small—entries in `cg_to_policy_map` plus `pol_str_maps_*` (typically a few KB per binding for modest value lists).

## Impacts / Key Questions

### Impact: new `TracingPolicyBinding` CRD

Adding a `TracingPolicyBinding` CRD introduces a new concept. Users must understand the relationship between templates and bindings and how to manage them effectively.

### Key Question: Is the new CRD necessary?

Could existing `TracingPolicy` mechanisms be extended to carry only values and reference a shared logic template, or is a dedicated CRD essential for clarity and usability?

### Impact: selectors in bindings are intended to be mutually exclusive

In this model, a `cgroup_id` can be associated with at most one binding (`policy_id`) per template. A new binding for the same cgroup is rejected. Binding the same cgroup to multiple policies simultaneously is not allowed.

### Key Question: Is this an acceptable use-case?

- Does exclusivity limit important use cases?
- If overlapping bindings are useful, how should precedence or merging be defined?
- How can we enforce idempotency/ordering between multiple Tetragon agents?

### Impact: partial support for variable types and operators

The proposed binding logic is currently limited to:

- matchArgs / matchData filters
- String / charbuf / filename types
- eq / neq operators

### Key Question: Is full support required for the first version?

Extending the binding logic to other types/operators would require different eBPF maps/approaches. If the design of the API is flexible enough to allow future extensions without breaking changes, could we start with a limited set and expand later based on user needs?

### Impact: multiple bindings per template

Currently only one binding is supported for each template

### Key Question: do we need multiple bindings per template?

The design should be extensible to support multiple bindings without API changes. I'm not sure multi-binding support would be really needed in practice for this reason I would avoid complicating the code too much until we have a real use case for it.
