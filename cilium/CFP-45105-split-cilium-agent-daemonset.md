# CFP-003: Template

**SIG: SIG-NAME** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-04-07

**Cilium Release:** 1.19.2

**Authors:** Mohammadreza Kiani (mohammadrezakiani.shojaee@gmail.com)

**Status:** Implementable

## Summary

Ability to split cilium agent daemonset into multiple daemonsets per different node pools.
* NOTE: We can work on implementing this feature on the helm chart by opening respective PRs, but first, we want to know the maintainers' and also the community's opinion about the possible different options which we will explain here.

## Motivation

For better resource optimization in case of different usage pattern in different node pools based on nodes' sizes and workloads.

## Goals

The goal here is to make the chart flexible to let us deploy one operator alongside with multiple agent daemonsets per nodepool.

## Proposal

### Overview

While using the community cilium helm chart to deploy the operator and agent, we faced some issues as we cannot split the agent daemonset into multiple daemonsets via separate releases in the same cluster because of some naming conflicts of some RBAC resources defined for the agent.

The list of the problematic resources is as following:

- ClusterRole/cilium
- ClusterRoleBinding/cilium
- Role/cilium-config-agent
- RoleBinding/cilium-config-agent

There are two options as following which both of them are possible to implement and there is no clear trade off to decide which one is better and that's why we will put both options here to first decide which one is a better go.

### Option 1:

Add the ability to enable/disable all related RBAC resources, so that we can have one release which will contain operator and all RBAC resources which will be shared via other multiple agent daemonset releases (in agent releases, the RBAC resources would be disabled).

#### Pros

- No duplicate RBAC resources.
- Easier management of the resources.
- Easier implementation and maintenance in the chart.

#### Cons

- No separation and isolation for the separated agent daemonsets RBAC resources.
- Having a kind of dependency of the shared RBAC resources in all agent daemonset releases on the operator release.

### Option 2:

Make the RBAC resources naming dynamic (generated via the release name via a helper function), so that each daemonset agent will have its own RBAC resources.

#### Pros

- Better separation and isolation for separated agent daemonsets RBAC resources.

#### Cons

- Duplicate RBAC resources.
- More complex management and maintenance.
- More difficult implementation in the chart.
