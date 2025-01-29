# CFP-041: Clang-format for BPF formatting

**SIG: SIG-Datapath**

**Begin Design Discussion:** 2024-09-10

**End Design Discussion:** 2024-09-24

**Cilium Release:** 1.18

**Status:** Implementable

**Authors:** Dylan Reimerink <dylan.reimerink@isovalent.com>, Lorenz Bauer <lorenz.bauer@isovalent.com>

## Summary

We would like to standardize on clang-format for BPF code formatting. At the moment, checkpatch.pl is leading for the way we format our BPF code (C code). While checkpatch.pl is a good tool for detecting issues in the code, it is not great for fixing any issues it finds.

## Motivation

A number of us are broadly working on refactoring the BPF codebase to allow for pre-compilation of BPF programs instead of compiling them at runtime. To facilitate these changes, we are planning on using automation to edit the codebase. These changes will impact aspects of the code such as indentation which requires us to reformat code not necessarily touched by the changes. To avoid manual reformatting, we would like to use clang-format to automatically format the codebase.

Additionally, having the ability to (semi-)automatically format code via an editor plugin, git hooks, or makefile should boost productivity since contributors no longer have to go back and forth between checkpatch.pl and their editor to manually fix formatting issues.

The reason why this needs to be a proposal is because during testing we have concluded that switching out checkpatch.pl for clang-format cannot be done without changes to the way we format our code. clang-format forces us to make decisions on how we want to format our code which checkpatch.pl doesn't care about.

## Goals

* Use `clang-format` as the canonical tool in CI for enforcing code formatting/style.
* Provide make targets for running `clang-format` on the codebase.
* Disable formatting related checks in checkpatch.pl, leaving linting checks enabled.
* Make the code base easier to modify using automated tools.

## Non-Goals

* Remove checkpatch.pl from the CI pipeline.
* Minimize the diff caused by switching to `clang-format`.

## Proposal

### Overview

We propose the following:

* Create a small 3 person committee consisting of Dylan Reimerink and two other people to debate and decide on the formatting rules we want to enforce.
* Modify the existing `.clang-format` file to reflect the decisions made by the committee.
* Run `clang-format` on the entire C codebase and commit the changes.
* Add these changes to a `.git-blame-ignore` file so the commit doesn't mess up the blame history.
* Change CI to use clang-format to detect formatting issues and switch off formatting checks in checkpatch.pl.
* Backport these changes to older, still maintained branches to minimize merge conflicts during backporting.
* Bypass the normal CODEOWNERS review process, instead having the committee review and approve the PR(s) needed for this change.

### The Committee

Even when trying to get clang-format as close to the existing format, there are still a number of decisions that need to be made. Our current formatting is internally inconsistent having multiple standards for indenting, spacing, alignment, etc. between and within files. These aspects where previously not enforced by checkpatch.pl and up to the individual contributor to decide. However, clang-format requires us to make these decisions.

Code formatting / code style is infamous for being a topic that can lead to heated debates and bikeshedding. Unfortunately chances are slim that we can make every contributor happy. We propose to create a small committee we trust to make these decisions and to given them the power to decide on a canonical style. The committee will consist of Dylan Reimerink being the committer to work on this primarily and two other people yet to be chosen.

### Existing format and `.clang-format` config

The starting point of this work will be the existing format. We already have a `.clang-format` config file in the repository which has been created in the past to approximate the existing formatting style. The committee will review the changes made by clang-format and decide on the best settings to use.

### Initial re-formatting commit

When we are happy with the results produced by `clang-format`, we will run it against the codebase and commit the changes. This will be a single commit to avoid polluting the git history with formatting changes. We will add this commit to a `.git-blame-ignore` which is recognized by both the git cli tool and GitHub to ignore this commit when showing blame information to minimize the impact. 

### Tooling changes

We propose to update the cilium-builder docker image which builds our clang to also provide `clang-format` and standardize on that version of the tool. This means that the formatter will update in tandem with the compiler. This might add additional work when upgrading clang.

We will add make targets to run `clang-format` in-place (directly update files) or output any suggested changes to stdout.In  CI we will run one of these targets to detect if a committer properly formatted their code (clang-format should give no suggestions).

We will modify the checkpatch script in the checkpatch docker image, disabling the formatting checks now covered by clang-format and leaving all other checks enabled.

Lastly, we will add a new linter to CI to detect the addition of `/* clang-format off */` or `// clang-format off` comments which can be used to disable clang-format for a block of code. We will likely need to add a few of these blocks during the initial re-formatting commit for bits of the code clang-format can't handle properly. But it should be discouraged to add more exceptions in the future.

### Backporting

We propose to backport the config, tooling and CI changes to older branches we still maintain. Then run clang-format on these branches to update their formatting in the same way and adding the commit in a `.git-blame-ignore` file.

Causing the older branches to be re-formatted in the same way should minimize merge conflicts when backporting changes. Since formatting only re-arranges code and doesn't change the code itself, the risk of doing this should be minimal.

We recommend formatting older branches with the newest `clang-format` version to minimize potential differences in formatting between clang-format versions.

### Bypassing the normal CODEOWNERS review process

Finally, we propose to bypass the normal CODEOWNERS review process when merging. Instead, the committee should review and approve the PR(s) needed for this change. Once all committee members have approved the PR(s), the PR(s) should be merged by a repo admin or tophat, bypassing the branch protection rules. Otherwise, we would involve additional people at the very end of the process making the purpose of the committee pointless.

## Feedback / Decision

We will be sharing this proposal during the community meeting on 2024-09-10 (10th of July) as well as on the Cilium & eBPF Slack. We request that changes to the proposal be made via the Github PR within a two week period. After this period, we will will notify the community of the final CFP and ask for people who disapprove to tell us then. If no one disapproves, we will consider the proposal accepted, else we will arrange for a vote in accordance with the Cilium governance structure.
