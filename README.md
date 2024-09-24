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
needs when creating a proposal.

Many design docs also begin their life as a Google doc or other shareable
file for easy commenting and editing when still in the early stages of discussion.
Once your proposal is ready, submit it as a PR to it's dedicated project folder (`cilium, hubble, tetragon`).

### When you are ready to submit your CFP please:

1. Create a CFP issue in the repo your design applies to if you haven't already
2. Create the file in this repo, with a path of `<repo>/CFP-###-subject.md` where the number is the CFP issue number

For example, if your issue is filed in https://github.com/cilium/hubble with
the issue number 000, and the subject of that CFP is to "Change foo to bar",
the path would be `hubble/CFP-000-change-foo-to-bar.md`.

### Getting feedback and promoting your CFP in the community:

- Not sure who to ping? Take a look in the [Cilium's Community Teams](https://github.com/cilium/community/tree/main/ladder/teams) (descriptions of the Cilium teams can be found [here](https://github.com/cilium/cilium/blob/main/CODEOWNERS))
- Share it on the [#development channel in Slack](https://cilium.slack.com/archives/C2B917YHE) or [one of the more specific SIG channels](https://docs.cilium.io/en/v1.13/community/community/#id1)
- Raise the design for discussion during the [weekly community call](https://docs.cilium.io/en/v1.13/community/community/#id1)

# Getting a design approved

A CFP can be merged with the approval of one committer. A CFP merged as implementable means the proposal is viable to be implemented, but not that there is a 100% chance the code will be accepted. Dormant and declined CFPs are also an important part of documentation for the project and should also be merged.

# Design proposal drift

After a design proposal is merged, it's likely that the actual implementation
will begin to drift slightly from the original design. This is expected and
there is no expectation that the original design proposal needs to be updated
to reflect these differences.

The code and our documentation are the ultimate sources of truth. CFPs are
the starting point for the implementation, but can also be updated as needed.

# Status

The status of a CFP indicates its maturity. There are five different statuses
that a CFP can have:

1. Draft: A CFP in any form (Google doc, HackMD, PR, ect.) with associated Github issue that has not yet
been merged into this repository.

2. Implementable: CFPs that have been approved by one committer and merged into
this repository are given the implementable status. This means that the ideas in the CFP
have been agreed in principle and the coding work can now be started. When merged,
the CFP should be up to date and the relevant stakeholders should be in alignment
even if they are still going through the practical ramifications of how to implement it.

3. Released [Project] X.Y: Everything listed in the CFP as part of the proposal (not future milestones) is merged into the project repository with the listed release.

4. Dormant: This status is given to a CFP that has not been
implemented for a variety of reasons, like not enough engineering cycles or not
currently a focus of the project. Any CFP where there is no active effort to
build the solution can have the dormant status. This serves as a way to preserve
previous discussions on a solution, but where the solution was not yet agreed
upon as implementable. Dormant CFPs can be reactivated
at any time if there is interest in the project. The next step for a dormant CFP is to amend it to be Implementable.

5. Declined: This proposal was considered by the community but ultimately rejected. The community may come back to the proposal in the future but a new CFP should be used. Rejected proposals can be useful as documentation on why the given proposal did not make sense.

## Other resources 

Need more clarification? Here are a list of resources from the Cilium community, and some FAQs on how to get more involved! Can't find your answer here? Please reach out on the Cilium Slack [#development](https://cilium.slack.com/archives/C2B917YHE) for other questions.

### FAQs

<details>
<summary>Who can review a CFP?</summary>
    Anyone can leave a helpful review on a CFP! Questions, ideas, and points of clarification that can help refine a CFP are greatly appreciated.
</details>

<details>
<summary>Who can approve a CFP?</summary>
    Any [committer](https://github.com/cilium/cilium/blob/main/MAINTAINERS.md) can approve a CFP.
</details>

<details>
<summary>How can I gauge interest on my CFP?</summary>
    To get preliminary feedback on your CFP, attend a Cilium weekly [community meeting](https://docs.cilium.io/en/stable/community/community/#community-meetings) or ask on the Cilium Slack . If possible, ask in the relevant [SIG](https://docs.cilium.io/en/stable/community/community/#special-interest-groups) slack channel or meeting.
</details>
