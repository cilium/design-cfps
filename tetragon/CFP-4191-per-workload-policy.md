# CFP-4191: Tetragon per-workload policies

**SIG:** SIG-TETRAGON

**Begin Design Discussion:** 2025-11-05

**Tetragon Release:** X.XX

**Authors:** Andrea Terzolo <andrea.terzolo@suse.com>, Kornilios Kourtis <kkourt@cisco.com>

**Status:** Draft

## Summary

Today, with Tetragon it is not possible to share a common enforcement logic across many Kubernetes workloads. For each workload users need to create a separate `TracingPolicy` with the same enforcement but with different values.

## Motivation

The current approach of one `TracingPolicy` per workload leads to 2 main problems:

- P1 (Scaling): The current eBPF implementation (one program + many maps per `TracingPolicy`) scales poorly in clusters with many per-workload policies (attachment limits, redundant evaluations, memory growth).

- P2 (UX / Composition): Crafting many near-identical `TracingPolicy` resources with only small differences (e.g. filter values) is operationally cumbersome and error prone.

In this document we propose to tackle scaling (P1) and UX (P2) separately.

## Goals

- Reduce the number of attached eBPF programs and map footprint when a common enforcement logic is shared across many workloads.
- Expose a friendly API to reuse common part of the policies avoiding the user to rewrite the same logic multiple times.
- Preserve existing expressiveness (selectors, filters, actions).
- Minimize API churn.

## Non-Goals

- TBD

## P1: Scaling the BPF Implementation

The current Tetragon implementation creates at least one BPF program and several maps per `TracingPolicy` resource.
This has some limitations:

- Many ebpf programs attached to the same hook may add latency when the hook is the hot path.
- If ebpf programs rely on eBPF trampoline, they are and are subject to the [`BPF_MAX_TRAMP_LINKS`](https://elixir.bootlin.com/linux/v6.14.11/source/include/linux/bpf.h#L1138) limit (38 on x86). So in some cases a limited number of programs can be attached to the same hook.
- Tens of maps per policy and several large maps (for example, policy filters, socktrack_map, override_tasks) lead to multi‑MB memlock per policy.

### Option 1: Share eBPF Programs & Maps across different TracingPolicies

This design for simplicity introduces two concepts: templates and bindings.

- A template is a way to define the enforcement logic we want to share across policies. So it injects the eBPF programs plus some maps that will be populated later by bindings.
- A binding is a way to provide specific values for the enforcement logic.

When a template is deployed, Tetragon creates a `BPF_MAP_TYPE_HASH` map (`cg_to_policy_map`) to match cgroup IDs with binding `policy_id`s. Initially, the map is empty, so the template has no effect.

When a binding is deployed, for each cgroup matching the binding, Tetragon inserts a (`cgroup_id` → `binding_id`) entry into `cg_to_policy_map`.

If a cgroup already has a binding for that template, there could be different options to handle the conflict:

1. Rollback of both bindings involved in the conflict. This ensures that the system remains in a consistent state and avoids idempotency/ordering issues between multiple Tetragon agents.
2. Take the binding with the highest priority. Each binding should have a predefined priority value.

Once the cgroup is associated with the binding, the id is used to look up the binding values in specialized maps.
According to the type of values and operators used in the template, different specialized maps are created to store the binding values.
For example, for string eq/neq filters, Tetragon could create a `BPF_MAP_TYPE_HASH_OF_MAPS` map where the key is the `binding_id` and the value is the hash set of strings sourced from the binding. Each binding will create an entry in this map. This map could use the same 11‑bucket sizing technique as existing `string_maps_*` maps.

This [proof of concept](https://github.com/cilium/tetragon/issues/4191) shows a preliminary implementation of this design.

**Pros**:

- small code changes to the existing Tetragon eBPF programs
- possibility to change the binding values at runtime by updating the specialized maps without reloading the eBPF programs.
- ...

**Cons**:

- different types of values and operators require different specialized maps, increasing implementation complexity.
- ...

### Option 2: Tail Call policy chains

TBD

**Pros**:

- TBD

**Cons**:

- TBD

## P2: Avoid Repetition in Policy Definitions

The current Tetragon `TracingPolicy` API requires users to repeat the same enforcement logic for each workload, changing only the values (for example, allowed binaries). The following is an example of two policies that share the same logic but have different values:

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "policy-1"
spec:
  podSelector:
    matchLabels:
      app: "my-deployment-1"
  kprobes:
  - call: "security_bprm_creds_for_exec"
    syscall: false
    args:
    - index: 0
      type: "linux_binprm"
    selectors:
    - matchArgs:
      - index: 0
        operator: "NotEqual"
        values:
        - "/usr/bin/sleep"
        - "/usr/bin/cat"
        - "/usr/bin/my-server-1"
      matchActions:
      - action: Override
        argError: -1
  options:
  - name: disable-kprobe-multi
    value: "1"
---
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "policy-2"
spec:
  podSelector:
    matchLabels:
      app: "my-deployment-2"
  kprobes:
  - call: "security_bprm_creds_for_exec"
    syscall: false
    args:
    - index: 0
      type: "linux_binprm"
    selectors:
    - matchArgs:
      - index: 0
        operator: "NotEqual"
        values:
        - "/usr/bin/ls"
        - "/usr/bin/my-server-2"
      matchActions:
      - action: Override
        argError: -1
  options:
  - name: disable-kprobe-multi
    value: "1"
```

### Option 1: Template + Binding

We could introduce two new CRDs: `TracingPolicyTemplate` and `TracingPolicyBinding`.

#### `TracingPolicyTemplate`

A `TracingPolicyTemplate` specifies variables which can be populated at runtime, rather than being hardcoded at load time. Selectors within the policy reference these variables by name.

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyTemplate
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

#### `TracingPolicyBinding`

A `TracingPolicyBinding` associates a `TracingPolicyTemplate` with concrete values for its variables and a workload selector.

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyBinding
metadata:
  # <template>-<bindingName>
  name: "block-process-template-values-1"
spec:
  templateRef:
    name: "block-process-template"
  overrideAllowed: false
  overridePriority: 0
  # Same selectors used today in TracingPolicy with no constraints
  podSelector:
    matchLabels:
      app: "my-app-1"
  containerSelector:
    matchExpressions:
      - key: name
        operator: In
        values:
        - main
  bindings:
  - name: "targetExecPaths"
    values:
    - "/usr/bin/true"
    - "/usr/bin/ls"
```

if at least one of the binding involved in the conflict has `overrideAllowed` set to `false`, Tetragon should rollback all bindings involved and log an error. In case of `overrideAllowed` set to `false`, `overridePriority` is ignored.

If `overrideAllowed` is set to `true`, Tetragon should use `overridePriority` to determine which binding takes precedence.

**Pros**:

- freedom to use arbitrary selectors for each binding.
- no requirements for the user to add labels or other identifiers to workloads to ensure exclusivity.

**Cons**:

- users need to be aware of potential overlapping bindings for the same template and handle conflicts accordingly.

### Option 2: Template + Binding (mutually exclusive selectors)

Similar to Option 1, but we require that workload selectors in bindings for the same template are mutually exclusive. This guarantees to users that there will be no overlapping bindings for the same template, but at the cost of flexibility.

The template resource is identical to Option 1, while the binding resource slightly changes:

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyBinding
metadata:
  # <template>-<bindingName>
  name: "block-process-template-values-1"
spec:
  templateRef:
    name: "block-process-template"
  bindings:
  - name: "targetExecPaths"
    values:
    - "/usr/bin/true"
    - "/usr/bin/ls"
```

No selectors are specified in the binding. Instead, users must ensure that pods are labeled in the correct way in order to match the intended binding. To match the above binding, users must label their pods with `tetragon-block-process-template=values-1` (`tetragon-<template>=<bindingName>`).

**Pros**:

- mutually exclusive selectors by design, no risk of accidental overlapping bindings.

**Cons**:

- require users to label workloads appropriately to ensure exclusivity.
- no override allowed, this could be a limitation for some use cases, in which users may want to override a global binding with a more specific one.
- doesn't allow to specify different values for different containers within the same pod.
