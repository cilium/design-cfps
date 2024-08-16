# CFP-25694: Move cilium/cilium-cli code into cilium/cilium repository

**Code Name: Mega Ketchup**

**SIG: [sig-cilium-cli]**

**Begin Design Discussion:** 2023-05-25

**Cilium Release:** 1.15

**Authors:** Michi Mutsuzaki `<michi@isovalent.com>`

**Status:** Implementable

## Summary

Move [cilium/cilium-cli] code into [cilium/cilium] repository.

## Motivation

The main motivation for moving [cilium/cilium-cli] code into [cilium/cilium]
repository is to minimize the feature development overhead for adding new tests
to `cilium connectivity test`.

Currently, adding a new test to `cilium connectivity test` requires the following
steps:

- Open a pull request against [cilium/cilium-cli] repo to add a new test, and
  wait for the CI build to finish.
- Modify GitHub workflows files to pick up the CI version of `cilium-cli` in
  your pull request against [cilium/cilium] repo.
- Once the new tests passes, undo the change in the workflow files.
- Get these pull requests reveiewed and merged.

Even after all these steps, the new test does not become a part of the regular
CI until the [sig-cilium-cli] team releases a new version of `cilium-cli` that
includes the test, and Renovate updates GitHub wokflow files to pick up the new
`cilium-cli` version. This has been a major source of frustration for some Cilium
contributors.

## Goals

* Move [cilium/cilium-cli] code into [cilium/cilium] main branch under `cli/`
  directory. This enables Cilium contributors to implement new features with
  tests in a single pull request against [cilium/cilium] repository.
* Establish a backporting process for changes inside `cli/` directory.
* Avoid disrupting the cilium-cli development and release processes as much as
  possible.

## Non-Goals

* Rename the name of the `cilium-cli` binary from `cilium` to something else.
* Release `cilium-cli` as a part of Cilium releases.

## Proposal

This proposal consists of three steps to move [cilium/cilium-cli] code into
[cilium/cilium]:

### 1. Move [cilium/cilium-cli] code to [cilium/cilium] under `cli/` directory

I'm assuming that you want to preserve the Git history of [cilium/cilium-cli]. See
[michi-covalent/cilium] on how the code migration might look like. Squashing
these commits into a single commit is another option.

### 2. Make [cilium/cilium-cli] code a part of `github.com/cilium/cilium` Go module

Merge [cilium/cilium-cli] code into `github.com/cilium/cilium` Go module by
removing `cli/go.mod`. This allows [cilium/cilium-cli] code to depend on
[cilium/cilium] code in the same tree.

This is **not** to suggest `cilium-cli` to be tied to a specific `cilium`
version. The goal of this proposal is to make it easy for contributors to add
tests against the latest Cilium code. Continue to maintain the backward
compatibility of `cilium-cli` with respect to `cilium` versions.

### 3. Release `cilium-cli` from [cilium/cilium-cli] repository

We do not need to change the release process / cadence to accomplish the goals
defined in this CFP. Continue releasing `cilium-cli` from [cilium/cilium-cli] at
its own cadence using its own versioning that is independent of [cilium/cilium]
versioning as you have been doing in [cilium/cilium-cli] repository.

See [michi-covalent/cilium-cli] as an example of how [cilium/cilium-cli] repo
might look like after the code migration. Basically it contains:

- [`go.mod`](https://github.com/michi-covalent/cilium-cli/blob/865cac4f148ce88cd04d99f8ecfe61a0ae4f645f/go.mod)
- A simple [`main.go`](https://github.com/michi-covalent/cilium-cli/blob/865cac4f148ce88cd04d99f8ecfe61a0ae4f645f/main.go)
  that calls [`NewCiliumCommand`](https://github.com/cilium/cilium-cli/blob/44ae1874fae4544c0db34dac89c11e37365b76ef/cli/cmd.go#L27)
- Release [tags](https://github.com/cilium/cilium-cli/tags) and [artifacts](https://github.com/cilium/cilium-cli/releases)

## Impacts / Key Questions

### Impact: Backporting to stable branches

Starting in `v1.15`, you need to backport bug fixes related to
`cilium connectivity test` command from `main` branch to affected stable
branches since these stable branches use their own copy of `cli/` to run
`cilium connectivity test`. Apply the same [backporting process] as the rest of
the Cilium codebase.

### Impact: Module name change

This proposal impacts users who depend on [cilium/cilium-cli] as a library.

- The module path needs to be updated from `github.com/cilium/cilium-cli` to
  `github.com/cilium/cilium/cli`.
- Instead of depending on a version of `github.com/cilium/cilium-cli` (`v0.14.5`
  for example), you need to inspect `go.mod` of a specific `github.com/cilium/cilium-cli`
  version, and transitively find the `github.com/cilium/cilium/cli` version to
  depend on.
- Alternatively, depend on a particular `github.com/cilium/cilium` version without
  inspecting `go.mod` file in `github.com/cilium/cilium-cli`.

If you must retain the Go module name, you need to copy the `cilium-cli` code
back to [cilium/cilium-cli] repo and modify all the import paths back to
`github.com/cilium/cilium-cli` for each release..
See https://github.com/cilium/design-cfps/pull/9#discussion_r1211996796 for details.

### Key Question: How do you run CI for CLI? More specifically, how do you ensure that changes to Cilium CLI remain compatible with older released versions of Cilium?

Here is the list of GitHub workflows that currently run for each pull requests in [cilium/cilium-cli]
using the latest version of Cilium:

- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/aks-byocni.yaml
- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/eks-tunnel.yaml
- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/eks.yaml
- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/externalworkloads.yaml
- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/gke.yaml
- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/kind.yaml
- https://github.com/cilium/cilium-cli/blob/main/.github/workflows/multicluster.yaml

[cilium/cilium] has a similar coverage using CI images instead of released images,
so it's probably an overkill to port all of these workflows to [cilium/cilium]. My
proposal is to move https://github.com/cilium/cilium-cli/blob/main/.github/workflows/kind.yaml
to [cilium/cilium], and run this workflow for pull requests that modify files
under `cli/` directory. This ensures that any changes under `cli/` directory get
tested against a released version of Cilium.

## Alternative approach: Extend connectivity test from cilium/cilium repo

Sebastian Wicki suggests another approach to extend `cilium connectivity test`
command from [cilium/cilium] repo using cilium-cli's [Hook interface]. This
approach involves two steps:

### Step 1: Implement the Hook interface in cilium/cilium repo.

See [this commit](https://github.com/cilium/cilium/commit/26526a1890de6e05ca58d37b5a2da5822a3f22f0)
as an example for how to implement the [Hook interface]. This enables you to
extend `cilium connectivity test` without having to migrate the entire [cilium/cilium-cli]
code base into [cilium/cilium] repo. Then, build a local version of cilium-cli
with the additional hook during the CI.

### Step 2: Import the hook back in cilium/cilium-cli repo.

Import the hook from the previous step in [cilium/cilium-cli] repo so that
additional tests in [cilium/cilium] repo get included in upstream cilium-cli
releases. See [this commit](https://github.com/cilium/cilium-cli/commit/7b474e61be358fb0b9f2cd6fd075ba843b5d78f5)
as an example.

[sig-cilium-cli]: https://github.com/orgs/cilium/teams/sig-cilium-cli
[cilium/cilium]: https://github.com/cilium/cilium
[cilium/cilium-cli]: https://github.com/cilium/cilium-cli
[michi-covalent/cilium]: https://github.com/michi-covalent/cilium/pull/169
[michi-covalent/cilium-cli]: https://github.com/michi-covalent/cilium-cli/tree/cli-test
[backporting process]: https://docs.cilium.io/en/stable/contributing/release/backports/
[kubernetes/kubernetes staging directory]: https://github.com/kubernetes/kubernetes/tree/master/staging/
[apimachinery]: https://github.com/kubernetes/apimachinery
[Hook interface]: https://github.com/cilium/cilium-cli/blob/4a6cb76243704f96dfc02ff312b57e4d0ced0d84/internal/cli/cmd/hooks.go#L13-L17
