# CFP-42466: change-multi-ip-pool-crd

**SIG: SIG-NAME** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2025-11-04

**Cilium Release:** 1.19

**Authors:** kyounghoonJang <matkimchi_@naver.com>

**Status:** Implementable

**Reference** this CFP was translated from Korean to English using AI.

## Summary

Currently, CiliumPodIPPool defines CIDRs as simple strings, preventing granular allocation control. This proposal enhances the schema to support intra-CIDR IP reservation via a reservedRange field. This approach allows users to block allocation for specific parts of a CIDR rather than the entire subnet, offering greater flexibility for handling orphaned IPs and managing pool migrations.

## Motivation

When using Multi-Pool IPAM, operators sometimes need to decommission or replace existing CIDRs.
The only current way to stop IP allocation from a CIDR is to remove it from the pool entirely, which can cause existing pod IPs to become orphaned and trigger warnings in the operator.

To support a two-step migration workflow, we propose enabling CIDRs to be marked as “non-allocatable”:

1. Update the old CIDR with a reservedRange covering its available IP space.

2. Drain and migrate workloads to nodes using the new CIDR.

3. Once the old CIDR is no longer in use, remove it from the pool.

This allows for controlled and predictable transitions between IP pools without service disruption.

## Goals
Enhance the CiliumPodIPPool CRD schema to support a reservedRange field, allowing users to exclude specific IP ranges within a CIDR from being allocated.
## Proposal

### Overview

Current structure:
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  name: green-pool
spec:
  ipv4:
    cidrs:
      - 10.20.0.0/16
      - 10.30.0.0/16
    maskSize: 24
```
Proposed structure:
```yaml
apiVersion: cilium.io/v2
kind: CiliumPodIPPool
metadata:
  name: green-pool
spec:
  ipv4:
    cidrs:
      - cidr: 10.20.0.0/16
        reservedRange: 10.20.0.0-10.20.0.99
      - cidr: 10.30.0.0/16
    maskSize: 24
```

### Implementation Details

- **Struct Change**: The cidrs field in spec will be changed from a simple []string to a list of objects: []struct { cidr: string, reservedRange: string }.
  - cidr: The network CIDR.
  - reservedRange: (Optional) A specific IP range within the CIDR to exclude from allocation.
- **Allocator Logic**: The allocator will parse the reservedRange and exclude those specific IPs from the available pool.
- **Migration Strategy (Lazy Migration)**:
    - **Single Source of Truth**: The Operator will prefer the `v2` version of CiliumPodIPPool when both `v2` and `v2alpha1` objects exist. If only `v2alpha1` is present, the Operator will ingest the `v2alpha1` object.
    In other words, `v2` takes precedence whenever available, and the Operator uses `v2alpha1` only as a fallback when no `v2` resource exists.
    - **Lazy Conversion**: If automatic migration is enabled, the Operator will perform lazy conversion in the reconcile loop:
      1. Detect v2alpha1 objects that have not yet been migrated.
      2. Convert them into the v2 format and create a corresponding v2 resource.
      3. Leave the original v2alpha1 object intact (not deleted) to avoid unexpected disruptions.
    - **Configuration**: The migration behavior will be controlled through configuration (e.g., a value in the Helm chart or a field in the Operator’s ConfigMap), allowing operators to enable or disable automatic v2 migration.
- **Upgrade Guide**: Automated downgrades are not supported. We will include a cautionary note in the documentation, advising users to proceed with migration only when they are confident.

## Impacts / Key Questions

### Impact 1 : Breaking Change - Schema Migration

Since the schema has changed from storing CIDRs as simple strings in a slice to using a struct that contains both the CIDR and the reservedRange field, existing v2alpha users will encounter a schema mismatch issue.

#### Key Question: A way to apply this feature without affecting existing users

Implement this feature while promoting the MultiPool IP functionality to v2.

##### Pros
- Enables us to promote the API to v2 while introducing the new feature.

##### Cons
- The schema changes (string → object format) will be included directly in the official v2 release and may not have full maturity at the time of promotion.

**Note:** To mitigate this concern, the new reservedRange field could be introduced under a beta the v2 API until stability is validated.

#### Key Question: How do we handle resource conflicts and precedence during migration?
If both a v2alpha resource and a v2 resource exist with the same name, how should the operator handle this conflict?

It makes sense to prefer v2 resources, as v2 is generally considered the stable version.
For v2alpha resources, the operator will leave v2alpha objects and log a warning.

###### Pros
- The operator’s conflict-handling logic becomes simpler.
- Since the user’s CRs are not deleted directly, potential accidents or data loss are avoided.

###### Cons
- Stale CRs may remain unattended, which could lead to confusion.

#### Key Question: What happens if the user has to downgrade to an older version of Cilium again

The v2alpha1 objects are not deleted automatically.
If a downgrade occurs, the older Cilium version will continue to read and reconcile these v2alpha1 objects as before