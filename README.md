# MetalBear RFCs - [RFC Book](https://metalbear-co.github.io/rfcs/)

The "RFC" (request for comments) process provides a consistent path for proposing substantial changes 
to MetalBear products (`mirrord` and future projects) so the team can align on technical direction.

## Table of Contents

  - [When you need to follow this process](#when-you-need-to-follow-this-process)
  - [What the process is](#what-the-process-is)
  - [The RFC life-cycle](#the-rfc-life-cycle)

## When you need to follow this process

You need to follow this process for "substantial" changes, including:

  - Major new features or capabilities
  - Significant architectural changes
  - Breaking changes to APIs or user-facing behavior
  - Changes that affect multiple components or teams

You don't need an RFC for:

  - Bug fixes
  - Documentation improvements
  - Refactoring that doesn't change behavior
  - Minor feature additions that don't affect architecture

When in doubt, discuss with the team first.

## What the process is

To propose a major change:

  - Fork or clone the RFC repository
  - Create a Github issue under the RFC repository
  - Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is descriptive)
  - Fill in the RFC with care—explain motivation, design details, drawbacks, and alternatives
  - Submit a pull request and link it the Github and Linear issue
  - Assign reviewers whose comments you're seeking
  - Build consensus through discussion and iterate on feedback
  - The team will review and decide to merge, close, or postpone
  - Before merging, rename your file to the next available RFC number in main (e.g., `0042-my-feature.md`)
  - Once merged, the RFC is "active" and can be implemented

## The RFC life-cycle

Once an RFC is "active":

  - Authors may implement it and submit PRs to the relevant repositories
  - Being active means the team has agreed to the approach in principle
  - It doesn't guarantee immediate implementation or specific priority

Minor changes to active RFCs can be made via amendments. Substantial changes should be new RFCs 
that reference the original.

---

## License

This repository is licensed under the MIT License.

## Contributions

Unless you explicitly state otherwise, any contribution submitted for inclusion 
shall be licensed under the MIT License, without additional terms or conditions.
