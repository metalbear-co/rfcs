- Feature Name: `charts-operator-unification`
- Start Date: 2026-03-05
- Last Updated: 2026-03-05
- RFC issue: [metalbear-co/rfcs/issues/18](https://github.com/metalbear-co/rfcs/issues/18)
- RFC reference:
  - [COR-1197](https://linear.app/metalbear/issue/COR-1197/move-charts-development-to-the-operator-repo)
  - [COR-1294 (sub-issue)](https://linear.app/metalbear/issue/COR-1294/1-5-move-charts-and-support-release-with-actions)

## Summary
[summary]: #summary

Unify the charts and operator repos by moving the contents of charts into the operator. Also, release the charts whenever the operator is released.

## Motivation
[motivation]: #motivation

Changes to the operator frequently require changes to be made to the charts. Some features require simultaneous changes in the operator, charts and mirrord (3 separate PRs) as well as docs and config docs (2 more separate PRs). This:
1. Makes local development and testing more difficult
2. Increases admin time overhead for PR authors and reviewers
2. Can result in the repos being out of sync with eachother
3. Is annoying

[COR-1109](https://linear.app/metalbear/issue/COR-1109/automate-configurationmd-update) reduces the number of PRs to 4, and by merging the charts and the operator, it can be reduced to 3.

_Problem 1: Hidden dependencies_

For example, take [COR-1149](https://linear.app/metalbear/issue/COR-1149/accept-otel-context-propagation-carriers-connect-to-our-tracing), which required 5 separate PRs. In this case, the feature (which was a customer request) was announced as released before it was present in the charts and before user documentation was available. This was due to the charts being released without an operator release, which resulted in a misleading changelog entry in the charts. The hidden dependency here was that the changes in the charts release required the operator to be released.

_Problem 2: Misleading changelogs/ when is the feature available to use?_

The operator changelog is updated upon operator release automatically, but until the charts are released the latest operator features are not available to users. Since we publish the [operator changelog in our docs](https://github.com/metalbear-co/docs-changelog), users may see features that are not yet available fo use. By forcing an operator release to release the charts, we can ensure released operator features are available immediately.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The contents of `metalbear-co/charts` is a copy of `metalbear-co/operator/charts`. Changes are made to `metalbear-co/operator/charts` and then copied to `metalbear-co/charts` when the operator is released.

- **For developers writing code changes** this means that changes to the charts occur in the same commit as the related changes to the operator, in `metalbear-co/operator`.
- **For developers performing a release** this means that you must only make one release commit, in `metalbear-co/operator`.
- **For non-developers at MetalBear** this means it is easy to tell when operator features are released AND available to customers, which is useful if a feature has been requested.
- **For users and customers** this means that features in the operator changelog will always be available in the latest chart release.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Assumptions:

- Releasing the operator should always trigger a charts release
  - Impact: developers will always have to update all changelogs and operator dependencies even when charts do not change otherwise
- Code will not be committed to the charts repo UNLESS it is pushed by the operator release, ever. otherwise, repos would have to sync both ways
  - Impact: it must be very clear or enforced that code cannot be committed directly to the charts repo

### Repository structure:

`metalbear-co/operator`
- visibility: private
- allows PRs?: yes
- main `HEAD` points to: most recent release commit _or_ unreleased changes
- code changes come from: developer PRs

`metalbear-co/charts`
- visibility: public
- allows PRs?: no
- main `HEAD` points to: most recent (charts) release commit
- code changes come from: pushed automatically from `metalbear-co/operator/charts`

### New release process:

- To release the operator and charts:
  - Create a release PR on the operator repo as before, with the addition of version bumps and changelog updates in the charts dir
  - When merged, create a GitHub release with semver version tag
  - Check that the operator is released and the charts are released with up to date code
- To release the charts only:
  - Create a release PR on the operator repo in the same way as a release PR created in the charts repo
  - When merged, manually dispatch the new charts release workflow from the operator repo actions
  - Check that the charts are released with up to date code
  - **Edge case**: Consider the following: `charts-only release commit merges -> another commit that touches the charts dir merges -> the new charts release workflow updates the charts repo`
      - This is not possible in the operator release process, since the release depends on a manually tagged commit
      - Possibly avoidable with a special commit name pattern, eg. `charts-release-YY-MM-DD`.

### Release flow:

- Release PR opened, CI runs with extra checks for release
- Release commit merges, GitHub release is created with a tag
- Tag push triggers a workflow that:
  - Triggers the operator release workflow
  - Pushes charts dir contents to charts repo
  - Triggers a charts release in the charts repo

### Implementation order

To minimise impact if something goes wrong:

- 1. Copy the charts source code to the operator repo, within its own directory
  - Trigger charts PR CI from the operator PR CI when files in charts dir change
  - Adjust files changed checks to account for charts/ non charts files
- 2. Move any open PRs/issues of ours from the charts to the operator repo, leave any PRs/issues opened by the users
- 3. In the operator repo, add a new job that will:
  - Push current state of charts source code to the charts repo main
  - Trigger a release there
- 4. Adjust operator release job to also trigger job from 3.
  - Operator release must finish before charts release triggered, so the charts are able to fetch operator (and if operator release fails, charts do not get released)
  - The trigger job from 3 should run when a semver version tag is created (same as current operator release)
- 5. Add an option to manually trigger job from 3., without operator release 
  - Add check to require up-to-date changelog in charts (if changelogs are not merged)


## Drawbacks
[drawbacks]: #drawbacks

- This change has the potential to break operator and/or charts releases if there are bugs
  - Customers will be directly affected if something is released incorrectly (e.g. incorrect dependencies) or if releases cannot be performed at all
- Operator CI/ actions may be made more complex
  - For example, separate changelog checks will be necessary for the operator and charts
  - Changed files checks for the operator may need to be altered to exclude the charts directory


## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?-->

## Prior art
[prior-art]: #prior-art

<!--Discuss prior art, both the good and the bad, in relation to this proposal. Examples include:

- Does this feature exist in similar products?
- What lessons can we learn from how competitors handle similar problems?
- Are there any published blog posts or papers that discuss this approach?-->

## Unresolved questions
[unresolved-questions]: #unresolved-questions

_Resolve before implementation_

- Will there any negative changes to development experience for the charts or operator?
- Will the 3 changelogs be kept separate? Can we combine them into 1?
  - How will this work during charts only releases?
  - Either: releases are always operator and charts AND changelogs can be unified
  - or releases can be charts only or both AND changelogs must stay separate
- Would there ever be a reason for code to be pushed to the charts repo when a release is not happening?
  - For example, if there are changes to the (public) charts README.md
  - Can we account for/ allow/ disallow this?
- How do we ensure that updates to the charts code that merge _after_ a release PR dont get included in code pushed to the charts repo when performing a charts-only release?
  - ie. we only push code up to and including release commit (may require specific commit name)
- How do we avoid charts CI failures on release PRs for the operator and charts?
  - The charts CI will attempt to install the chart being released, which depends on the operator version being released, which doesn't exist yet
  - It is safe to disable the charts CI on release PRs (there are no extra checks in the charts CI for release PRs)?

## Future possibilities
[future-possibilities]: #future-possibilities

- Fully automate operator releases
  - Ideally, developers wouldn't have to create a release commit, wait for it to merge and create a GitHub release
  - If we could automate creation of the release commit and GitHub release with tag, we could have a manual dispatch only job to perform release
    - Would we need a way to point to the specific commit to be released?
- Move to a monorepo: eventually, we could move all of our code into one repo
  - When considering this in the past, issues raised included reluctance to use git submodules and concerns about code that generates or manages licenses being public
- Publish the charts changelogs to the website (if changelogs are kept separate)
  - (see [operator changelog](https://github.com/metalbear-co/docs-changelog))
