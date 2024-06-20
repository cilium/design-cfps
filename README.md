This directory contains CFP design proposals for features impacting repos across the
Cilium Github organization.

# Purpose of CFPs

The purpose of a Cilium Feature Proposal (CFP) is to allow community members to gain feedback
on their designs from the Committers before the community member commits to
executing on the design. By going through the design process, developers gain a
high level of confidence that their designs are viable and will be
accepted.

NOTE: This process is not mandatory. Anyone can execute on their own design
without going through this process and submit code to the respective repos.
However, depending on the complexity of the design and how experienced the
developer is within the community, they could greatly benefit from going through
this design process first. The risk of not getting a design proposal approved
is that a developer may not arrive at a viable architecture that the community will
accept.

# How to create CFPs

To create a CFP, it is recommended to use the `CFP-003-template.md`
file as an outline. The structure of this template is meant to provide a starting
point for people. Feel free to edit and modify your outline to best fit your
needs when creating a proposal. When you are ready to submit your CFP please:

1. Create a CFP issue in the repo your design applies to if you haven't already
2. Create the file in this repo, with a path of `<repo>/CFP-###-subject.md` where the number is the CFP issue number

For example, if your issue is filed in https://github.com/cilium/hubble with
the issue number 000, and the subject of that CFP is to "Change foo to bar",
the path would be `hubble/CFP-000-change-foo-to-bar.md`.

Many design docs also begin their life as a Google doc or other shareable
file for easy commenting and editing when still in the early stages of discussion.
Once your proposal is done, submit it as a PR to the design-cfps folder.

Promote awareness about your CFP in the community:

- Share it on the [#development channel in Slack](https://cilium.slack.com/archives/C2B917YHE) or [one of the more specific SIG channels](https://docs.cilium.io/en/v1.13/community/community/#id1)
- Raise the design for discussion during the [weekly community call](https://docs.cilium.io/en/v1.13/community/community/#id1)

# Getting a design approved

For a CFP to be considered viable, a Cilium committer needs to aprove it.
After the approval, the design can be merged. A merged design proposal 
means the proposal is viable to be executed on, but not that there is a
100% chance it will be accepted.

# Design proposal drift

After a design proposal is merged, it's likely that the actual implementation
will begin to drift slightly from the original design. This is expected and
there is no expectation that the original design proposal needs to be updated
to reflect these differences.

The code and our documentation are the ultimate sources of truth. CFPs are merely
the starting point for the implementation.
