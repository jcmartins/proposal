# Proposing Changes to Sensu

## Introduction

The Sensu project's design process is inspired by Golang's [proposal process](https://github.com/golang/proposal).
Significant changes to Sensu's core functionality or API must be
discussed, and sometimes formally documented, before they can be implemented.

This document describes the process for proposing, documenting, and
implementing changes to the Sensu project.

## The Proposal Process

### Goals

- Make sure that proposals get a proper, fair, timely, recorded evaluation with
  a clear answer.
- Make past proposals easy to find, to avoid duplicated effort.
- If a design doc is needed, make sure contributors know how to write a good one.

### Definitions

- A **proposal** is a suggestion filed as a GitHub issue in the [sensu-go](https://github.com/sensu/sensu-go)
  repository, identified by having the `proposal` label.
- A **design doc** is the expanded form of a proposal, written when the
  proposal needs more careful explanation and consideration.

### Scope

The proposal process should be used for any notable change or addition to the
core functionality or API.
Since proposals begin (and will often end) with the filing of an issue, even
small changes can go through the proposal process if appropriate.
Deciding what is appropriate is matter of judgment we will refine through
experience.
If in doubt, file a proposal.

#### Compatibility

If you intend to make a breaking change to existing 1.x functionality, then
you must discuss how this change breaks that functionality and why you believe
the change to be necessary.

### Process

- [Create an issue](https://github.com/sensu/sensu-go/issues/new) describing the proposal.

- Like any GitHub issue, a `proposal` issue is followed by an initial discussion
  about the suggestion. For `proposal` issues:
	- The goal of the initial discussion is to reach agreement on the next step:
		(1) accept, (2) decline, or (3) ask for a design doc.
	- The discussion is expected to be resolved in a timely manner.
	- If the author wants to write a design doc, then they can write one.
	- A lack of agreement means the author should write a design doc.
	- If there is disagreement about whether there is agreement,
	  [sean@](mailto:sean@sensu.io) is the arbiter.

- It's always fine to label a suggestion issue with `proposal` to opt in to this process.

- It's always fine not to label a suggestion issue with `proposal`.
  (If the suggestion needs a design doc or is declined but worth remembering,
  it is trivial to add the label later.)

- If a `proposal` issue leads to a design doc:
	- The design doc should be checked in to [the proposal repository](https://github.com/sensu/proposal/) as `design/NNNN-shortname.md`,
	  where `NNNN` is the GitHub issue number and `shortname` is a short name
	  (a few dash-separated words at most).
	  Clone this repository with `git clone https://github.com/sensu/proposal` and open a pull request against the repository
    to initiate the review process.
	- The design doc should follow [the template](design/TEMPLATE.md).
	- The design doc should address any specific issues asked for during the
	  initial discussion.
	- It is expected that the design doc may go through multiple checked-in revisions.
	- New design doc authors may be paired with a design doc "shepherd" to help work
	  on the doc.
	- Comments on proposal pull requests should be restricted to grammar, spelling, or
    procedural errors related to the preparation of the proposal itself.
    All other comments should be addressed to the related GitHub issue.

- Once comments and revisions on the design doc wind down, there is a final
  discussion about the proposal.
	- The goal of the final discussion is to reach agreement on the next step:
		(1) accept or (2) decline.
	- The discussion is expected to be resolved in a timely manner.
	- If clear agreement cannot be reached, the arbiter
	  ([sean@](mailto:sean@sensu.io)) reviews the discussion
	  and makes the decision to accept or decline.

- The author (and/or other contributors) do the work as described by the
  "Implementation" section of the proposal.

## Help

If you need help with this process, please ask for help in the #help channel in the
[Sensu Community Slack](https://slack.sensu.io).
