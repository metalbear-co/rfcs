- Feature Name: `github_pr_verification`
- Start Date: 2026-02-12
- Last Updated: 2026-02-12
- RFC PR: [metalbear-co/rfcs#9](https://github.com/metalbear-co/rfcs/pull/9)
- RFC reference:
  - [metalbear-co/rfcs#9](https://github.com/metalbear-co/rfcs/pull/9)

## Summary
[summary]: #summary

Add the ability for the mirrord operator to automatically post a verification comment on a GitHub pull request when a mirrord session targeting that PR's branch completes. This creates visible proof that code was tested against a real environment, directly within the PR conversation.

## Motivation
[motivation]: #motivation

Today, when a developer uses mirrord to test their changes against a staging or production environment, there is no record of that testing visible in the pull request. Reviewers have no way to know whether the code was actually tested against a real environment or only unit tested locally.

This creates several problems:

1. **No visibility into real-environment testing.** A reviewer looking at a PR sees CI checks (lint, unit tests, build) but has no indication of whether the author ran their code against a live deployment. This is the most valuable kind of testing mirrord enables, yet it leaves no trace.

2. **No accountability for testing rigor.** Teams that want to enforce "test against staging before merging" have no automated way to verify this happened. It relies on trust and manual communication ("I tested it locally with mirrord").

3. **Missed organic distribution opportunity.** Every PR review is a touchpoint where developers see tooling in action. Tools like Codecov, Vercel, and Dependabot grew significantly through ambient visibility in PR comments — developers see the badge, get curious, and adopt the tool. mirrord currently has no presence in this high-traffic surface.

### Use Cases

- **Individual developer:** Runs `mirrord exec --target deploy/checkout-service -- node server.js`, tests their checkout flow fix against staging, ends the session. A comment automatically appears on their open PR showing what was tested, against which target, and for how long. The reviewer sees this when they open the PR.

- **Team lead enforcing testing policy:** Wants all PRs to be tested against staging before merge. Can see at a glance which PRs have mirrord verification comments and which don't. In the future, this could become a required check via GitHub's branch protection rules.

- **CI pipeline integration:** A CI job runs `mirrord exec` as part of an end-to-end test suite. The operator posts a verification comment showing the CI-driven test session, including which pipeline triggered it.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Setup (one-time, by cluster admin)

1. Create a GitHub token with permission to comment on PRs in the target repositories:

```bash
kubectl create secret generic mirrord-github-token \
  --namespace mirrord \
  --from-literal=token=ghp_xxxxxxxxxxxx
```

2. Enable the feature in the operator's Helm values:

```yaml
operator:
  github:
    enabled: true
    tokenSecret: mirrord-github-token
    apiUrl: "https://api.github.com"  # override for GitHub Enterprise
```

3. Upgrade the operator:

```bash
helm upgrade mirrord-operator metalbear/mirrord-operator -f values.yaml
```

That's it. No per-repo configuration, no CI workflow changes, no developer action required.

### Daily usage (zero additional steps)

Developers use mirrord exactly as they do today. The only new behavior is that the mirrord CLI sends the git repository and branch information alongside the session metadata it already sends to the operator.

When a session ends, the operator:

1. Looks up the open PR matching the session's branch name
2. Posts a formatted comment on the PR

The comment looks like this:

> ### mirrord verified
>
> | | |
> |---|---|
> | **Target** | `deploy/checkout-service` |
> | **Namespace** | `staging` |
> | **User** | `han` |
> | **Duration** | 4m 32s |
> | **Branch** | `han/fix-checkout-flow` |
>
> *Tested with [mirrord](https://mirrord.dev)*

If the same developer runs multiple sessions against the same PR, subsequent sessions update the existing comment (or post a new one — see [Unresolved questions](#unresolved-questions)).

### Interaction with existing features

- **Jira integration:** Independent. Both can fire on session end simultaneously. A session can produce both a Jira time-tracking webhook and a GitHub PR comment.
- **CI sessions:** CI-triggered sessions include additional metadata (`ci_info.provider`, `ci_info.pipeline`, `ci_info.triggered_by`). The PR comment can display this to distinguish CI-driven verification from manual developer sessions.
- **Copy target / isolated testing:** Sessions using `copy_target` work the same way — the comment will indicate that an isolated copy was used rather than the live target.
- **Admin dashboard:** The admin dashboard (served via the license server) already displays session history and utilization data. PR verification events would complement this — the dashboard shows aggregate session metrics, while the GitHub comment provides per-PR visibility. In a future iteration, verification history could be surfaced in the dashboard as well (see [Future possibilities](#future-possibilities)).

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Architecture overview

```
Developer machine                    Customer's K8s cluster
┌──────────┐                        ┌─────────────────────────┐
│ mirrord   │───session + git info──►│ mirrord operator        │
│ CLI/IDE   │                        │                         │
└──────────┘                        │  Session ends:          │
                                     │  ├─ Telemetry (existing)│
                                     │  ├─ Metrics (existing)  │
                                     │  ├─ Jira webhook (exist)│
                                     │  └─ GitHub comment (NEW)│
                                     │         │                │
                                     └─────────┼────────────────┘
                                               │
                                               ▼
                                     GitHub API (customer's
                                     GitHub / GHE instance)
```

All session data stays within the customer's infrastructure. The only external call is to the customer's own GitHub instance to post the comment.

### Changes to the session CRD

Extend `MirrordClusterSessionSpec` with git repository information:

```rust
/// Git repository information sent by the client.
pub struct SessionGitInfo {
    /// Git branch name (e.g., "han/fix-checkout-flow").
    /// Already available via `jira_metrics.branch_name` but duplicated
    /// here for clarity and to decouple from Jira.
    pub branch_name: String,

    /// Repository owner (e.g., "metalbear-co").
    pub repo_owner: String,

    /// Repository name (e.g., "mirrord").
    pub repo_name: String,
}
```

Add to `MirrordClusterSessionSpec`:

```rust
pub git_info: Option<SessionGitInfo>,
```

The mirrord CLI already knows the git remote and branch (it currently sends `branch_name` for Jira). The CLI would additionally parse the git remote URL to extract `repo_owner` and `repo_name`.

### Operator configuration

New fields in `ServiceConfig`:

```rust
/// Path to file containing a GitHub token (PAT or GitHub App installation token).
#[clap(long, env = "OPERATOR_GITHUB_TOKEN_PATH")]
pub github_token_path: Option<PathBuf>,

/// GitHub API base URL. Defaults to https://api.github.com.
/// Override for GitHub Enterprise Server instances.
#[clap(long, env = "OPERATOR_GITHUB_API_URL",
       default_value = "https://api.github.com")]
pub github_api_url: String,
```

Helm chart additions in `values.yaml`:

```yaml
operator:
  github:
    enabled: false
    tokenSecret: ""
    apiUrl: "https://api.github.com"
```

The token secret is mounted as a volume at `/github/token` and read at startup, following the same pattern as the license secret.

### New module: `operator/session/src/github.rs`

A new module (~150 lines) with three responsibilities:

**1. `GitHubClient` struct**

Holds the `reqwest::Client`, token, and API base URL. Initialized once at operator startup if `github_token_path` is configured.

**2. Branch-to-PR matching**

```rust
async fn find_pr_for_branch(
    &self,
    owner: &str,
    repo: &str,
    branch: &str,
) -> Result<Option<PrInfo>>
```

Uses the GitHub REST API:
```
GET /repos/{owner}/{repo}/pulls?head={owner}:{branch}&state=open
```

This returns PRs whose head branch matches. If multiple PRs exist for the same branch (unlikely but possible), we use the most recently updated one.

**3. Comment posting**

```rust
async fn post_verification_comment(
    &self,
    owner: &str,
    repo: &str,
    pr_number: u64,
    comment_body: &str,
) -> Result<()>
```

Uses:
```
POST /repos/{owner}/{repo}/issues/{pr_number}/comments
```

The comment body is rendered from a format function that takes the session spec and duration as input.

### Hook point in `BaseController`

In `base_controller.rs`, the `report_session_finished` flow already handles Jira webhooks. The GitHub comment is added as a parallel async task:

```rust
// Existing Jira webhook
if let Some(jira_url) = &self.jira_webhook_url {
    let client = self.http_client().clone();
    tokio::spawn(async move { /* POST to Jira */ });
}

// New: GitHub PR comment
if let Some(github) = &self.github_client {
    if let Some(git_info) = &session.spec.git_info {
        let github = github.clone();
        let git_info = git_info.clone();
        let session = session.clone();
        tokio::spawn(async move {
            if let Err(e) = github.post_session_verification(
                &git_info, &session, duration
            ).await {
                tracing::warn!(%e, "Failed to post GitHub PR comment");
            }
        });
    }
}
```

Failures are logged but do not affect session cleanup. This matches the Jira webhook's fire-and-forget pattern.

### Client-side changes (mirrord CLI / IDE extensions)

The mirrord CLI needs to send `git_info` when creating a session. The information is derived from the local git repository:

- **Branch name:** `git rev-parse --abbrev-ref HEAD` (or equivalent libgit2 call)
- **Remote URL:** `git remote get-url origin`, then parsed to extract owner and repo name
  - Handles both SSH (`git@github.com:metalbear-co/mirrord.git`) and HTTPS (`https://github.com/metalbear-co/mirrord.git`) remotes
  - Non-GitHub remotes are ignored (feature is GitHub-specific)

If the CLI cannot determine git info (not in a git repo, no remote configured, non-GitHub remote), `git_info` is `None` and no PR comment is posted. This is a graceful degradation — the feature is optional and best-effort.

### Required GitHub token permissions

The token needs minimal permissions:

| Permission | Scope | Reason |
|------------|-------|--------|
| `pull_requests: read` | Target repos | Find PRs by branch name |
| `issues: write` | Target repos | Post comments on PRs (PR comments use the Issues API) |

For a fine-grained PAT, these are scoped to specific repositories. For a classic PAT, the `repo` scope covers both.

### Comment format

The comment is a markdown table. For a standard developer session:

```markdown
### mirrord verified

| | |
|---|---|
| **Target** | `deploy/checkout-service` |
| **Namespace** | `staging` |
| **User** | `han` |
| **Duration** | 4m 32s |

*Tested with [mirrord](https://mirrord.dev)*
```

For a CI-triggered session, additional fields are included:

```markdown
### mirrord verified (CI)

| | |
|---|---|
| **Target** | `deploy/checkout-service` |
| **Namespace** | `staging` |
| **Pipeline** | `e2e-tests` |
| **Triggered by** | `push` |
| **Duration** | 12m 08s |

*Tested with [mirrord](https://mirrord.dev)*
```

## Drawbacks
[drawbacks]: #drawbacks

- **Token management burden.** The cluster admin must create and manage a GitHub token. Tokens expire and need rotation. This is mitigated by using fine-grained PATs with long expiration or a GitHub App installation token (which auto-refreshes).

- **Network egress from cluster.** The operator needs outbound HTTPS access to the GitHub API (or GitHub Enterprise instance). Some highly locked-down clusters may not allow this without explicit firewall rules.

- **Comment noise.** If a developer runs many short mirrord sessions against the same PR, it could create many comments. This needs a strategy — either updating a single comment or batching (see [Unresolved questions](#unresolved-questions)).

- **Git info accuracy.** The CLI derives repo info from the local git remote. If the developer has an unusual remote configuration (e.g., a fork with a different remote name), the owner/repo may be wrong. This is an edge case that can be addressed by allowing explicit configuration.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why the operator (not the CLI or a cloud service)?

Three alternatives were considered:

**Alternative A: mirrord cloud service.** The CLI sends session data to mirrord's cloud, which posts to GitHub. This was rejected because customers have strong preferences to keep session data within their infrastructure. Routing through an external cloud creates privacy and compliance concerns.

**Alternative B: CLI posts directly.** The CLI uses the developer's own `gh` auth to post the comment. This is simple but has drawbacks: comments appear as the developer (no consistent bot identity), it requires every developer to have `gh` auth configured with the right permissions, and it doesn't work for IDE extensions without additional setup.

**Alternative C: GitHub Action.** A workflow file in each repo queries the operator for session data and posts comments. This requires per-repo CI configuration, which adds adoption friction and defeats the "zero-setup for developers" goal.

**The operator approach** is the best fit because:
- The operator already has session data and already fires webhooks on session end (Jira pattern)
- One credential in one place (K8s secret) covers all developers and all repos
- No per-repo or per-developer configuration
- Session data never leaves the customer's infrastructure
- It follows an established pattern in the codebase (~250 lines of new code)

### Why a PAT and not a GitHub App?

A GitHub App would provide a branded bot identity (`mirrord[bot]`) and eliminate token rotation concerns. However:

- Registering a GitHub App requires MetalBear to host the app, or the customer to register their own
- If MetalBear hosts it, the App's private key would need to be distributed to customer clusters, or session data would need to flow through MetalBear's infrastructure — both undesirable
- If the customer registers their own App, it adds significant setup friction

A PAT (or machine user account) is simpler to set up and keeps everything self-contained. The code is auth-method-agnostic — if a customer later sets up a GitHub App and provides its installation token instead, the same code works without changes.

## Prior art
[prior-art]: #prior-art

- **Codecov** posts coverage reports as PR comments. Uses a GitHub App for branding. Grew significantly through viral PR visibility — developers see coverage badges on PRs they review and discover the tool.

- **Vercel** posts deploy preview links as PR comments. Each comment includes a preview URL and deployment status. This became one of Vercel's primary acquisition channels.

- **Dependabot / Renovate** post PR comments and create PRs for dependency updates. The ambient visibility in PR feeds drives awareness and adoption.

- **SonarCloud** posts code quality analysis as PR comments with detailed metrics. Uses both comments and Check Runs for structured reporting.

- **mirrord's own Jira integration** (built by @gememma in PR #925) follows the exact same pattern: fire an HTTP request on session end with session metadata. The GitHub feature extends this pattern to a different target.

All of these tools demonstrate that PR-level visibility drives organic adoption and provides genuine value to development teams.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Comment deduplication strategy.** When a developer runs multiple sessions against the same PR, should we: (a) update the existing comment with the latest session, (b) append sessions to a single comment, or (c) post separate comments? Option (a) keeps the PR clean but loses history. Option (b) could make the comment very long. Option (c) risks noise. Leaning toward (a) — find and update the existing mirrord comment if one exists.

- **Multi-repo sessions.** If a developer's branch exists in a fork rather than the main repo, the `owner/repo` derived from the remote may not match where the PR is open. Need to decide if we should search across forks or require the remote to match exactly.

- **Rate limiting.** GitHub's API rate limit is 5,000 requests/hour for authenticated requests. For a busy cluster with many sessions, we may need to batch or throttle requests. This is unlikely to be a problem in practice (each session is 2 API calls: find PR + post comment), but should be considered.

- **Opt-out mechanism.** Should there be a way for individual developers to opt out of PR comments? For example, a mirrord config flag like `"github_verify": false` that tells the operator to skip the comment for that session.

- **Comment identity.** When using a PAT, the comment appears as the token owner. Should we recommend creating a dedicated machine user (e.g., `mirrord-bot`) for a cleaner appearance? Document this as a best practice?

## Future possibilities
[future-possibilities]: #future-possibilities

- **GitHub Check Runs.** Instead of (or in addition to) comments, create Check Runs that appear in the PR's Checks tab. These can show structured pass/fail data and can be made required via branch protection rules — enabling "must test with mirrord before merging" policies.

- **GitHub App registration.** Provide a MetalBear-registered GitHub App on the GitHub Marketplace that customers can install for branded `mirrord[bot]` identity. The operator would use the App's installation token instead of a PAT. The API calls remain identical.

- **Traffic summary in comments.** If the mirrord layer/agent captures HTTP request metadata (method, path, status code, latency), this could be included in the comment to show what was actually tested. This would make the verification comment significantly more valuable.

- **Session comparison.** Compare the current session against previous sessions for the same target and show regressions (e.g., "latency increased 3x on `/api/checkout`").

- **GitLab and Bitbucket support.** The same pattern applies to other Git hosting platforms. The `github.rs` module could be generalized to a `git_hosting.rs` module with provider-specific implementations.

- **Admin dashboard integration.** Surface PR verification history in the admin dashboard (license-server based). Show which PRs have been verified, by whom, and when — giving team leads a centralized view alongside existing session utilization metrics.

- **Automatic session triggering.** Instead of waiting for developers to manually run mirrord, a webhook could trigger a mirrord session automatically when a PR is opened or updated, running a predefined test suite against the target deployment.
