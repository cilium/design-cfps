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
Once your proposal is ready, submit it as a PR to the design-cfps folder for approval.

Promote awareness about your CFP in the community:

- Share it on the [#development channel in Slack](https://cilium.slack.com/archives/C2B917YHE) or [one of the more specific SIG channels](https://docs.cilium.io/en/v1.13/community/community/#id1)
- Raise the design for discussion during the [weekly community call](https://docs.cilium.io/en/v1.13/community/community/#id1)

# Getting a design approved

For a CFP to be considered implementable, a Cilium committer needs to aprove it.
After their approval, the design can be merged. A merged design proposal 
means the proposal is viable to be implemented, but not that there is a
100% chance the code will be accepted.

# Design proposal drift

After a design proposal is merged, it's likely that the actual implementation
will begin to drift slightly from the original design. This is expected and
there is no expectation that the original design proposal needs to be updated
to reflect these differences.

The code and our documentation are the ultimate sources of truth. CFPs are
the starting point for the implementation, but can also be updated as needed.

# Status

The status of a CFP indicates its maturity. There are four different statuses
that a CFP can have:

1. Draft: A CFP in any form (Google doc, Github issue, ect.) that has not yet
been merged into this repository.

2. Implementable: CFPs that have been approved by one committer and merged into
this repository are given the implementable status. This means that the ideas in the CFP
have been agreed in principle and the coding work can now be started. When merged,
the CFP should be up to date and the relevant stakeholders should be in allignement
even if they are still going through the practical ramifications of how to implement it.

4. Updated: If there have been signifigant changes been the original CFP and the
implementation, the CFP can be updated to reflect those changes. The new status of
the CFP is Updated since it has changed signifigantly since it was first merged.

5. Dormant: This status is given to CFP that have not or did not yet end up being
implemented for a variety of reasons, like not enough engineering cycles or not
currently a focus on of the project. This can be either CFP that were discussed
but not implemented and the project would like to preserve those discussions for
future use or CFPs that were deemed implementable, but no one ever got around
to implementing them and the CFP is now dormant. Dormant CFPs can be reactivated
at any time if there is interest in the project.
