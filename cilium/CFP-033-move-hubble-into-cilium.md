# CFP-033: Move `cilium/hubble` into `cilium/cilium`

**SIG: `@cilium/sig-hubble`**

**Begin Design Discussion:** 2024/04/11

**Cilium Release:** 1.16

**Author:** Michi Mutsuzaki (`@michi-covalent`)

**Credit:**

This proposal is based on offline discussions with:

- Fabian Fischer (`@glrf`)
- Thomas Graf (`@tgraf`)
- Robin Hahling (`@rolinh`)
- Anna Kapuścińska (`@lambdanis`)
- Tobias Klauser (`@tklauser`)
- Tam Mach (`@sayboras`)
- Alexandre Perrin (`@kaworu`)
- Liz Rice (`@lizrice`)
- Glib Smaga (`@glibsm`)
- Joe Stringer (`@joestringer`)
- Sebastian Wicki (`@gandro`)
- Chance Zibolski (`@chancez`)

## Summary

Move `cilium/hubble` code into `cilium/cilium` repo to improve the Hubble CLI
development and release processes.

## Motivation

Currently, Hubble server code lives in `cilium/cilium` repo, while Hubble CLI
code is in a separate `cilium/hubble` repo. This complicates the Hubble CLI
development and release processes.

### Development process

Currently, modifying Hubble APIs requires dealing with multiple repos:

- Make API and server-side changes in cilium/cilium repo.
- Update go.mod to pick up the cilium/cilium changes in cilium/hubble repo, and
  make corresponding client-side changes.
- Wait for the next Hubble CLI release.
- Wait for Renovate to update [download-hubble.sh](https://github.com/cilium/cilium/blob/main/images/cilium/download-hubble.sh)
  before you can use the new Hubble CLI from the `cilium-agent` container.

### Release process

The current Hubble CLI release process requires:

- Vendor unreleased latest `cilium/cilium` code in `cilium/hubble` repo before a
  new `cilium/cilium` minor version gets released to pick up new Hubble API
  changes.
- Release a new Hubble CLI.
- Make sure [download-hubble.sh](https://github.com/cilium/cilium/blob/main/images/cilium/download-hubble.sh)
  gets updated before the corresponding `cilium/cilium` minor version gets
  released.
 
## Goals

The goal of this proposal is to simplify the Hubble CLI development / release
processes.

* Enable developers to make Hubble server and CLI changes in a single pull
  request, and immediately make the new Hubble CLI available in the CI
  `cilium-agent` images without having to wait for the next Hubble CLI release.
* Build Hubble CLI release binaries from Cilium release tags to simplify the
  release process.

## Non-Goals

* Archive or delete the cilium/hubble repo.
* Publish Hubble CLI releases in `cilium/cilium` repo instead of `cilium/hubble`
  repo. I am not opposed to the idea, but I haven't seen strong incentives to stop
  publishing releases in `cilium/hubble` repo.

## Proposal

This proposal consists of two phases.

### Phase 1: Move cilium/hubble code to cilium/cilium repo

In the fist phase, move `cilium/hubble` code to `cilium/cilium` repo without
changing the current Hubble CLI release process. See the following draft
pull requests for more details regarding the first phase:

- https://github.com/cilium/cilium/pull/31893
- https://github.com/cilium/hubble/pull/1444

After the phase 1 is complete:

- Hubble CLI in the cilium-agent container image gets built as a part of
  the Cilium release process.
- After each Cilium release, Hubble CLI gets released by:
  - Updating go.mod in `cilium/hubble` repo and picking up the new Cilium release, and
  - Following the current release process defined in `cilium/hubble` repo.

Additionally, update the current [integration tests](https://github.com/cilium/hubble/blob/main/.github/workflows/integration-tests.yaml)
and make it fit the style of workflows in `cilium/cilium` repo as a part of the phase 1.
See https://github.com/cilium/cilium/pull/31893#discussion_r1567010043 for additional
context.

#### Phase 1 Versioning / Release Cadence

Assuming that this proposal gets implemented before Cilium v1.16.0 gets released:

- Use the same version as Cilium for Hubble CLI releases. This means that the next
  non-patch Hubble CLI release version is v1.16.0.
- Maintain the current release cadence, which basically means:
  - Release a new minor version for each Cilium minor version.
  - Release patch versions on demand.

### Phase 2: Update the release process

In the second phase, coordinate with `@cilium/sig-hubble` and update the Hubble
CLI release process. Instead of having a separate GitHub workflow in
`cilium/hubble` repository, pushing a release tag in `cilium/cilium` repo can
trigger a workflow to build Hubble CLI release binaries and publish a release
in `cilium/hubble` repo.

After the phase 2 is complete:

- Hubble CLI in the cilium-agent container image gets built as a part of
  the Cilium release process.
- Hubble CLI release binaries gets built and published to `cilium/hubble`
  repo as a part of the Cilium release process.

#### Phase 2 Versioning / Release Cadence

Once the phase 2 is complete, a draft release gets pushed to `cilium/hubble` repo
for each Cilium (pre)release.

- Publish pre-release versions as pre-releases.
- Publish releases from the latest stable branch, roughly once every month. Clearly
  document that these releases are compatible with older Cilium versions.
- Discard drafts from older stable branches.

## Impacts / Key Questions

### Impact: Import path change

Downstream projects that depend on `github.com/cilium/hubble` Go module need to
change import paths from `github.com/cilium/hubble/*` to
`github.com/cilium/cilium/hubble/*`. One could argue this is acceptable since
Hubble CLI is still at major version zero (currently at `v0.13.2`), and therefore
the public APIs should not be considered stable according to [Semantic Versioning
2.0.0 Item 4](https://semver.org/#spec-item-4).

### Impact: Slower development time for Hubble CLI changes

If you are making changes that only modify Hubble CLI, it might take longer to
get your changes merged since running `cilium/cilium` workflows take longer than
running `cilium/hubble` workflows. This is a tradeoff that `@cilium/sig-hubble`
members need to accept to improve the overall development and release processes.

## Impact: Homebrew recipe

[Homebrew recipe for Hubble CLI](https://github.com/Homebrew/homebrew-core/blob/master/Formula/h/hubble.rb)
needs to be updated after Phase 2. I guess we can just send a pull request to
update it? I'm not too familiar with how the Homebrew recipe are being managed.

### Question: How do you version Hubble CLI releases in Phase 1?

`@glibsm` suggests using the Cilium version in go.mod, which is logical since
Hubble CLI gets built from a `cilium/cilium` (pre)release tag. So replace this
line in Makefile:

    VERSION=$(shell cat VERSION)

with with something like:

    VERSION=$(go list -f {{.Version}} -m github.com/cilium/cilium)

Continue maintaining the ["Releases" section in README.md](https://github.com/cilium/hubble/blob/main/README.md#releases)
to clearly document compatible Cilium versions.

### Question: How do you generate release notes?

Create a tool that generates release notes based on changes that went into
`hubble/` directory for a given release. I know what to do.

