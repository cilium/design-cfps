# CFP-25694: Move cilium/cilium-cli code into cilium/cilium repository

**Code Name: Mega Ketchup**

**SIG: [sig-cilium-cli]**

**Begin Design Discussion:** 2023-05-25

**Cilium Release:** 1.15

**Authors:** Michi Mutsuzaki `<michi@isovalent.com>`

## Summary

Move [cilium/cilium-cli] code into [cilium/cilium] repository.

## Motivation

The main motivation for moving [cilium/cilium-cli] code into [cilium/cilium]
repository is to minimize the feature development overhead for:

- adding new tests to `cilium connectivity test`.
- adding new tasks to `cilium sysdump`.

## Goals

* Move [cilium/cilium-cli] code into [cilium/cilium] main branch under `cli/` directory.
* Establish a backporting process for changes inside `cli/` directory.
* Don't disrupt the cilium-cli development and release processes.

## Non-Goals

* Rename the name of the `cilium-cli` binary from `cilium` to something else.
* Release `cilium-cli` as a part of Cilium releases.

## Proposal

This proposal consists of three steps to move [cilium/cilium-cli] code into
[cilium/cilium]:

### 1. Move [cilium/cilium-cli] code to [cilium/cilium] under `cli/` directory

I'm assuming that you want to preserve the Git history of [cilium/cilium-cli]. See
[michi-covalent/cilium] on how the code migration might look like.

### 2. Make [cilium/cilium-cli] code a part of `github.com/cilium/cilium` Go module

Merge [cilium/cilium-cli] code into `github.com/cilium/cilium` Go module by
removing `cli/go.mod`. This allows [cilium/cilium-cli] code to depend on
[cilium/cilium] code in the same tree.

This is **not** to suggest `cilium-cli` to be tied to a specific `cilium`
version. The goal of this proposal is to make it easy for contributors to add
tests against the latest Cilium code. We'll continue to maintain the backward
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

### Impact: Additional overhead for cilium-cli development

### Impact: Module name change

This proposal impacts users who depend on [cilium/cilium-cli] as a library.

- The module path needs to be updated from `github.com/cilium/cilium-cli` to
  `github.com/cilium/cilium/cli`.
- Instead of depending on a version of `github.com/cilium/cilium-cli` (`v0.14.5`
  for example), you need to inspect `go.mod` of a specific `github.com/cilium/cilium-cli`
  version, and transitively find the `github.com/cilium/cilium/cli` version to
  depend on.

### Key Question: Do we lose any test coverage?

### Key Question: How do we run CI for CLI? More specifically, how do we ensure that changes to Cilium CLI remain compatible with older released versions of Cilium?

### Key Question: Where do we store the release artifacts? Will the cilium/cilium repository release page contain both CNI and CLI releases? Or do we keep cilium-cli for CLI releases?

## Alternatives to Consider

[sig-cilium-cli]: https://github.com/orgs/cilium/teams/sig-cilium-cli
[cilium/cilium]: https://github.com/cilium/cilium
[cilium/cilium-cli]: https://github.com/cilium/cilium-cli
[michi-covalent/cilium]: https://github.com/michi-covalent/cilium/pull/169
[michi-covalent/cilium-cli]: https://github.com/michi-covalent/cilium-cli/tree/cli-test
[backporting process]: https://docs.cilium.io/en/stable/contributing/release/backports/
