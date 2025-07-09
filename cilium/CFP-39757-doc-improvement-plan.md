# CFP-003: Template

**SIG: Documentation** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2025-05-27

**Cilium Release:** `v1.19` or later

**Authors:** Peter Matulis <peter.matulis@isovalent.com>

**Status:** Implementable

## Summary

Establish a common set of standards for Cilium documentation that the project
can work towards.

## Motivation

Make the documentation better for all users of Cilium.

## Goals

Implement the following:

* documentation suggestions - elements that should be considered for every software feature
* documentation language - common terms that optimizes communication among contributors and reviewers
* documentation types - a rational framework for how to think about documentation
* formal structure - leading to consistency across the doc set
* readiness level definitions (for Beta and Stable)
* practical and detailed resources
    * example documentation outlines
    * templates that assist with the planning and writing of documentation
    * guidelines on how certain files and directories should be structured

## Proposal

# Improving Cilium Docs

The Cilium documentation has evolved organically over time. Although it
contains good content, its organization is sub-optimal, there is a lack of
consistency, and there is no common approach to how software should be
documented. This document describes how we can revitalize and improve the
Cilium docs by offering the following ideas:

* **documentation suggestions**<br>
  elements that should be considered for every software feature

* **documentation language**<br>
  common terms that optimizes communication among contributors and reviewers

* **documentation types**<br>
  a rational framework for how to think about documentation

* **formal structure**<br>
  leading to consistency across the doc set

* **readiness level definitions**<br>
  for Beta and Stable

* **practical and detailed resources**

    * example documentation outlines
    * templates that assist with the planning and writing of documentation
    * guidelines on how certain files and directories should be structured

Regarding the meanings of Beta and Stable, I wasn't able to find definitions in
the doc set hence I am proposing some via this larger initiative. They are
based on [feedback from the community][readiness-levels-community] a while
back. Since they are pretty general I don't see any change in how features have
been classified in the past. Defining these terms is more to assist with
implementation of the docs, not strict adherence.

## General strategy

The way forward is to identify a few topics that are relatively self-contained
and adapt them according to the new framework. These can serve as model docs
for others to follow. This establishes a beachhead. At this point we haven't
touched the top-level sections yet, just the lower-level technical topics. We
continue to work on things topic by topic until we feel the time is ripe to
move up to the top-level sections.

Concerning enforcement of the proposal and who would be responsible for things,
the aforementioned model docs can go far here. We can start by just pointing
them out and explaining that we're trying to improve things. Merge away if that
appears to cause consternation. Those people in the docs-structure team would
be the first to get onboard. Maybe I could be admitted to that team in due
course.

We should be careful of not introducing friction for contributors. We can merge
as we have been doing if we feel pushback, at least until we feel, if ever,
that standards should rise across the board. Some aspects of my proposal are
more important than others and those can be given emphasis, most notably
separating procedural content from conceptual content. It's of value to both
reviewers and contributors to have some kind of reference frame, even if it's
not strictly applied. It's something to inspire towards, a direction.
Contributors may include content that they may not have otherwise.

## Doc suggestions

The documentation suggestions are listed below. A readiness level should not be
approved without its corresponding doc suggestions being satisfied. However,
suggestions are only included when it makes sense to do so.

| Readiness Level | Documentation suggestions |
| :---: | ----- |
| **Stable** | [Diagram](#diagram)<br> [Scalability testing](#scalability-testing)<br> [Security](#security)<br> [Detailed feature explanation](#detailed-feature-explanation)<br> [Troubleshooting](#troubleshooting)<br> [Upgrade and downgrade](#upgrade-and-downgrade)<br> [Reference](#reference)<br> [Further reading](#further-reading)<br> [Monitoring](#monitoring) |
| **Beta** | [Simple feature explanation](#simple-feature-explanation)<br> [Failure scenarios](#failure-scenarios)<br> [Prerequisites](#prerequisites)<br> [Installation, configuration, enablement](#installation-configuration-enablement)<br> [Caveats and limitations](#caveats-and-limitations)<br> [Verification](#verification)<br> [Disable](#disable)<br> [Removal](#removal) |

The implementation of these suggestions are guided by a second dimension: the
documentation type. There are four such types: conceptual, how-to, tutorial,
and reference.

## Doc language

It's important for doc contributors/reviewers to learn the terms used to
discuss documentation. This provides the means for effective and efficient
collaboration. There are three kinds of terms:

* software readiness levels - [Appendix A](#appendix-a---software-feature-readiness-levels)
* doc suggestions (docsuggests) - [Appendix B](#appendix-b---documentation-suggestions)
* doc types (doctypes) - [Appendix C](#appendix-c---documentation-types)

## Starter templates

The purpose of these templates is to lend a consistent structure to the
documentation. They are considered how-to templates but they branch off into
other doctypes. View the how-to doctype as the kernel of your doc efforts.

There is a Beta template and a Stable template.

In terms of general writing guidance, abide by the [OSS doc style
guide][oss-docs-style-guide]. In the case of any disagreement, these templates
take precedence.

### <p style="text-align:center;">BETA</p>

TITLE:<br>
Must be action-based and use the imperative mood (e.g. "Audit all listening
ports"). See [How-to page titles](#how-to-page-titles).

---

INTRO TEXT:<br>
Note: Include this content always. No heading. See [Simple feature
explanation](#simple-feature-explanation).

---

ADMONISHMENT:<br>
Add a call-out and link to the Caveats and limitations
sub-section if it exists:

.. note::

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Review section [Caveats and limitations](#caveats-and-limitations) before continuing.

---

HEADING:<br>
**Prerequisites**

Note: Use this heading if appropriate. See [Prerequisites](#prerequisites).

---

MAIN BODY OF THE PAGE:<br>
Add one heading per chronological step

Notes:

* Use the imperative for each heading (e.g. "Configure Cilium").
* Ensure that at least the following are covered: installing and/or configuring
  and/or enabling the feature confirming a successful outcome (e.g. "Verify
  debug mode") - this should normally be the last heading.

HEADING:<br>
**Resources**

Note: Use this section heading, along with a sub-heading, if appropriate.
Maintain the given order:

**Caveats and limitations**

**Failure scenarios**

**Disable**

**Removal**

See [Extra pages](#standalone-pages) and [Ordering of items](#ordering-of-items).

---

### <p style="text-align:center;">STABLE</p>

TITLE:<br>
*Review and, if necessary, update content from the previous readiness level
(Beta).*

*Below are new requirements for the Stable readiness level. Add each item if
applicable and fill them in.*

---

DIAGRAM:<br>
Note: Place a diagram somewhere. Use if appropriate. See [Diagram](#diagram).

---

SUB-PAGE:<br>
**Concepts**

See [Detailed feature explanation](#detailed-feature-explanation).

Note: This page is always needed for a major feature.

---

SUB-PAGE:<br>
**Reference**

See [Reference information](#reference-information).

---

HEADING:<br>
**Resources**

Note: The below sub-headings have been added for the Stable readiness level.
Use if appropriate.

**Monitoring**

**Security**

**Scalability testing**

**Troubleshooting**

**Upgrade and downgrade**

**Further reading**

See [Extra pages](#standalone-pages) and [Ordering of items](#ordering-of-items).

---

## Documentation outline

The documentation of a feature involves headings, sub-headings, and possibly
several pages. These make up the documentation outline, where the how-to is
treated as the core documentation type. There must always be at least one
how-to for a major feature.

What follows are examples of documentation outlines for software features of
increasing complexity. Page titles and many section headings should be used
literally as given here. Generally only the how-to titles in italics need to be
called something original.

The current top-level structure includes the following main categories:

`+ NETWORKING >`<br>
`+ SECURITY >`<br>
`+ OBSERVABILITY >`<br>
`+ OPERATIONS >`

Under each of the above are many second-level sections. The example outlines
given below are intended to live underneath this existing hierarchy (top-level
or their second-level categories). The top-level will most likely need to
change in the future but this is what we are working with for now.

### Simple feature

**one page:** how-to

    	`+ Some title [how-to]`<br>
        	`+ section 'Resources'`<br>
            	`sub-section 'Removal'`

### Medium feature

**two pages:** how-to + another doctype page

    	`+ Some title [how-to]`<br>
        	`+ section 'Resources'`<br>
            	`sub-section 'Troubleshooting'`<br>
            	`sub-section 'Removal'`<br>
    	`+ Reference`

### Major feature

**four pages:** one how-to + two other doctype pages + one info breakout page

    	`+ Some title [how-to]`<br>
        	`+ section 'Resources'`<br>
            	`sub-section 'Scalability testing'`<br>
            	`sub-section 'Disable'`<br>
            	`sub-section 'Removal'`<br>
    	`+ Concepts`<br>
    	`+ Reference`<br>
    	`+ Resources`<br>
    	`+ Troubleshooting`

### Mega feature

**five pages:** two how-to + one other doctype pages + two resources breakout pages

    	`+ Some title #1 [how-to]`<br>
        	`+ section 'Resources'`<br>
            	`sub-section 'Upgrade and downgrade'`<br>
            	`sub-section 'Scalability testing'`<br>
            	`sub-section 'Disable'`<br>
            	`sub-section 'Removal'`<br>
    	`+ Some title #2 [how-to]`<br>
    	`+ Reference`<br>
    	`+ Resources`<br>
    	`+ Caveats and limitations`<br>
    	`+ Failure scenarios`

## How-to page titles

How-to page titles must be action-based and be expressed in the imperative mood (first word):

* " Provision a cluster without kube-proxy " 👍
  * " Kube-proxy Replacement " 👎

No hyper-capitalisation \-  use sentence case:

* " Enable SDS mode for inspection " 👍
  * " Enable SDS Mode for Inspection " 👎

The above also applies to section headings in the main body of the how-to.

**Finding an action-based title is a very good exercise**. If you are having
trouble doing this, it may mean:

* you're trying to do too many things - consider splitting your work into several how-tos
* you're trying to stuff a mix of doctypes into one page, resulting in a lack of clarity
* you're not sure what you're trying to do at all

## Standalone pages

Standalone pages are pages that exist without their own sections in the
navigation menu (they reside directly under the root topic section). The only
scenarios where standalone pages exist are:

* all how-to pages
* a single conceptual page
* a single reference page

In the case of standalone conceptual and reference pages, they should render
exactly as the following in the navigation menu (Sphinx can suppress page
titles):

* "Concepts"
* "Reference"

## Index pages

Index pages are always directly related to:

1. a section in the navigation menu; and
2. a directory on the file system

Do not place documentation content into an index page. An index page should be used solely to

1. initiate a local table of contents
2. provide a short introduction to the pages in that table of contents

## Concepts section

Use a conceptual section in the navigation menu, and thus a **concepts**
directory, and an **index.rst** file, if you have multiple conceptual page
types.

Each page title is based on some kind of concept. For any substantial topic,
include a general concepts page named "General concepts" whose file name is
**general.rst**. This is the page to link to whenever we want to refer to the
topic in a general way.

Prefix the topic to the index page's title. For example:

* "TLS inspection concepts"

The section should render as the following in the menu:

* "Concepts" (same as a standalone concepts page)

## Reference section

Use a reference section in the navigation menu, and thus a **reference**
directory, and an **index.rst** file, if you have multiple reference page
types.

Each page title is based on some kind of reference.

Prefix the topic to the index page's title. For example:

* "TLS inspection reference"

The section should render as the following in the menu:

* "Reference" (same as a standalone reference page)

## Resources section

Any single resource page should be accompanied by a resources section in the
navigation menu, and thus a **resources** directory, and an **index.rst** file.
This is in contrast to conceptual or reference page types where a section is
only needed for multiple such pages.

When you have content under a "Resources" sub-section (in a How-to) that grows
too large (about 1/2 a page), place it in this section as separate files.

When there are multiple How-to pages under a given topic, to avoid repeating
the same "Resources" material in each, you can optionally place it in this
section regardless of its size.  You can also use this section when there is
docsuggest content that applies to a topic in general.

Prefix the topic to the index page's title. For example:

* "TLS inspection resources"

The section should render as the following in the menu:

* "Resources"

Each of its pages' names are based on a docsuggest. For example:

* "Caveats and limitations"
* "Troubleshooting"

The file names are:

* **caveats-and-limitations.rst**
* **troubleshooting.rst**

The rendered navigation menu looks like:

    	`+ Resources`<br>
    	`+ Caveats and limitations`<br>
    	`+ Troubleshooting`

## Media

Create a directory under the topic directory called **media** to contain media
files.

## Ordering of items

Maintain the relative ordering of items whether they are sub-sections within a
how-to or pages within a navigation section.

### Sub-sections within a how-to

After the main body of a how-to there is a section for resources:

* Resources
* Caveats and limitations
* Monitoring
* Security
* Upgrade and downgrade
* Failure scenarios
* Scalability testing
* Troubleshooting
* Disable
* Removal
* Further reading

### Pages in the navigation menu

For a given topic, this is the page order as they appear in the menu:

**How-to** pages

**Conceptual** pages

**Reference** pages

**Resources** pages

## Appendix A - Software feature readiness levels

| Readiness Level | Definition |
| :---: | ----- |
| **Stable** | A feature that is appropriate for production due to significant hardening from testing and usage. Specific criteria include: Visibility \- ensures a feature is performing according to expectations Safety \- ensures a failure does not cause undue harm to the environment (fails gracefully) Robustness \- demonstrates real-world production usage |
| **Beta** | A feature that is appropriate for testing. Production use is only advised where preventative steps have been taken, such as pre-use staging environments, to mitigate against potential problems. User testing and feedback is highly encouraged in order to demonstrate stability and identify issues. |

## Appendix B - Documentation suggestions

### Diagram

The explanation of certain types of features can be greatly assisted by
including a diagram. Diagrams should follow a standard creation method and
style guide. Often accompanies/reinforces a conceptual treatment.

### Simple feature explanation

A brief explanation of the feature. These questions should be concisely
answered:

1. Technically, what is the feature all about?
2. Generally, why should a user care?
3. Specifically, what is/are the most common use case/s?

### Detailed feature explanation

A more in-depth conceptual treatment of the feature. Always needed for a major
feature. This is a standalone page that is typically appropriate for a major
feature. Think of why a feature is considered "major" and produce a few
paragraphs to answer that. This page should always be referenced (linked to)
whenever the feature is mentioned in any other page (another how-to, tutorial,
etc).

### Reference information

Tables/lists of configuration options, software versions, API calls, and
suchlike things. A diagram that is often consulted can fall into this category,
such as one describing a Reference Architecture. There is also a doctype called
Reference.

### Prerequisites

Software dependencies, computing resources, knowledge topics.

### Installation, configuration, enablement

The essence of a how-to or tutorial. The procedural steps for accomplishing
something.

### Verification

The user should come away confident that a procedure has yielded the desired
outcome. Some kind of test (typically command input and output).

### Disable

A way to "turn off" an enabled/configured feature, usually accompanied by a
"turn back on" (enable) method.

### Removal

A complete removal procedure for an enabled/configured feature. If not
possible/advisable, then a note should be added in the **Caveats and
Limitations** section. A replacement term for "cleaning-up".

### Further reading

Any pertinent resources for the user. Can be within the current doc set or an
external resource.

### Upgrade and downgrade

Instructions on how to upgrade or downgrade software. Omit the "and downgrade"
in the title if it's not advisable/possible to downgrade.

## Appendix C - Documentation types

### How-to

A **how-to** shows how to accomplish a task. The idea is that the user will
look through the docs, find a how-to, and decide whether it's of value to them.
A reasonable degree of knowledge can be assumed on the part of the reader.

Section headings are action-based (imperative mood) and each one moves the user
closer to the end goal (chronological order).

It's okay to collect multiple, related, and short procedures into one document
and title the page accordingly (e.g. "Manage software packages").

The how-to is the only doctype that must exist for every software feature. It
is the core doctype.

### Tutorial

In contrast to a how-to, a **tutorial** *tells* a reader what they should know.
It includes all the steps (within reason) needed to reach the tutorial's
objective(s) and assumes very little knowledge.

A tutorial is not meant to present material for the first time. It makes use of
already-documented material and can include best practices and helpful
supplementary information. Tutorials aren't constrained to a single software
feature, and can leverage other features and tools to arrive at a better
learning experience. Tutorials are educational in nature.

A tutorial is less formally structured than a how-to. It need not abide by
docsuggests and action-based titles/headings but it should be informed by them.

#### Quickstart

A **quickstart** is a special kind of tutorial. It onramps a user into a piece
of software or a specific way of using some software. It is characterised by
its brevity.

### Reference

A **reference** page contains reference information. Its telltale sign is a
table of significant size, a long list of items, or a diagram that is often
referred to in other pages.

If a how-to, conceptual page, or tutorial needs to refer to the above kind of
information, create a separate reference page and link to it.

### Conceptual

A **conceptual** page explains how a feature works and/or delves into even more
abstract notions of the feature, like how it came to be developed and why. A
certain degree of conceptual content is needed on most pages (e.g. how-to), but
when multiple paragraphs start to form, it is time for a dedicated conceptual
page.

This kind of page can be reinforced with example CLI commands/output, but this
is not to be confused with a procedure to follow, which is the domain of a
how-to or a tutorial.

<!-- LINKS -->

[readiness-levels-community]: https://docs.google.com/document/d/18OSu3vj44PV8gsnHTgOROWnDvY7SEyWEqVY_ONO_Rr0/edit?tab=t.0#heading=h.kkjsspvdjy5m
[oss-docs-style-guide]: https://docs.cilium.io/en/stable/contributing/docs/
