- Feature Name: `github_pr_verification`
- Start Date: 2026-02-12
- Last Updated: 2026-02-13
- RFC PR: [metalbear-co/rfcs#9](https://github.com/metalbear-co/rfcs/pull/9)
- RFC reference:
  - [metalbear-co/rfcs#9](https://github.com/metalbear-co/rfcs/pull/9)

## Summary
[summary]: #summary

Add the ability for the mirrord operator to automatically post a verification comment on a GitHub pull request when a mirrord session targeting that PR's branch completes. The comment includes session metadata (target, namespace, user, duration) and a traffic summary showing which HTTP endpoints were hit, how many times, and sample request/response bodies. Steal mode provides full request+response detail; mirror mode provides request-only detail. This creates visible proof of what was actually tested against a real environment, directly within the PR conversation.

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
2. Posts (or updates) a formatted comment on the PR with session metadata and traffic summary

The comment looks like this:

> ### mirrord verified
>
> | | |
> |---|---|
> | **Target** | `deploy/checkout-service` |
> | **Namespace** | `staging` |
> | **User** | `han` |
> | **Duration** | 4m 32s |
>
> #### Traffic summary
>
> | Method | Path | Status | Count |
> |--------|------|--------|-------|
> | GET | `/api/checkout` | 200 | 12 |
> | POST | `/api/checkout` | 201 | 3 |
> | POST | `/api/checkout` | 400 | 1 |
>
> *Expand each endpoint below for sample request/response bodies.*
>
> *Tested with [mirrord](https://mirrord.dev)*

If the same developer runs multiple sessions against the same PR, subsequent sessions **overwrite** the existing comment with the latest session's data. The rationale: the last session represents the developer's final testing state.

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
┌──────────┐                        ┌──────────────────────────────────┐
│ mirrord   │───session + git info──►│ mirrord operator                 │
│ CLI/IDE   │                        │                                  │
└──────────┘                        │  mirrord agent (in target pod):  │
                                     │  ├─ Intercepts HTTP traffic      │
                                     │  ├─ Accumulates traffic summary  │
                                     │  │   in memory (steal mode only) │
                                     │  └─ Sends summary on session end │
                                     │         │                        │
                                     │  Session ends:                   │
                                     │  ├─ Telemetry (existing)         │
                                     │  ├─ Metrics (existing)           │
                                     │  ├─ Jira webhook (existing)      │
                                     │  └─ GitHub comment (NEW)         │
                                     │    ├─ Session metadata            │
                                     │    └─ Traffic summary + bodies    │
                                     │         │                        │
                                     └─────────┼────────────────────────┘
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
    trafficSummary:
      enabled: true
      includeBodies: true          # include request/response body samples
      maxBodySize: 1024            # max bytes per body sample (truncated beyond this)
      maxSamplesPerEndpoint: 3     # keep latest N samples per method+path+status
```

The token secret is mounted as a volume at `/github/token` and read at startup, following the same pattern as the license secret.

### New module: `operator/session/src/github.rs`

A new module (~250 lines) with three responsibilities:

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
async fn post_or_update_verification_comment(
    &self,
    owner: &str,
    repo: &str,
    pr_number: u64,
    comment_body: &str,
) -> Result<()>
```

This method:
1. Lists existing comments on the PR, searching for one containing the `<!-- mirrord-verification -->` marker
2. If found, updates the existing comment via `PATCH /repos/{owner}/{repo}/issues/comments/{comment_id}`
3. If not found, creates a new comment via `POST /repos/{owner}/{repo}/issues/{pr_number}/comments`

The comment body is rendered from a format function that takes the session spec, duration, and optional traffic summary as input.

### Traffic summary capture (agent-side)

When the GitHub verification feature is enabled, the agent accumulates HTTP traffic metadata in-memory during the session. Both steal and mirror modes are supported, with different levels of detail:

- **Steal mode (full summary):** The agent proxies traffic, so it sees both requests and responses. The traffic summary includes method, path, status code, response size, and request/response body samples.
- **Mirror mode (request-only summary):** The agent copies incoming traffic but never sees responses (they go directly from the original pod back to the client). The traffic summary includes method, path, request body samples, and request count — but status code, response body, and response size are unavailable and shown as `—` in the comment.

#### What is captured

For each HTTP request/response pair intercepted via steal, the agent records:

- HTTP method (GET, POST, etc.)
- Request path (from URI)
- Response status code
- `Content-Type` and `Content-Length` headers
- Request and response body samples (bounded by `maxBodySize`, default 1KB)

#### In-memory accumulator

The agent maintains a bounded `HashMap` keyed by `(method, path, status_code)`:

```rust
struct TrafficSample {
    count: u64,
    content_type: Option<String>,
    avg_response_size: u64,
    /// Ring buffer of the latest N request/response body pairs.
    body_samples: VecDeque<BodySample>,
}

struct BodySample {
    request_body: Option<String>,   // truncated to max_body_size
    response_body: Option<String>,  // truncated to max_body_size
}

/// Keyed by (method, path, status_code).
type TrafficAccumulator = HashMap<(String, String, u16), TrafficSample>;
```

Memory is bounded:
- Body samples capped at `maxBodySize` (default 1KB) per body × 2 (request + response) × `maxSamplesPerEndpoint` (default 3) = ~6KB per unique endpoint
- A typical session with 30 unique endpoints uses ~180KB total
- A hard cap of 100 unique endpoints prevents unbounded growth from high-cardinality paths (e.g., `/users/1`, `/users/2`, ...). Entries beyond the cap are counted but bodies are not sampled.
- Binary bodies (detected via `Content-Type`) are stored as `[binary, {size} bytes]` rather than raw bytes.

#### Data flow on session close

When the session ends, the accumulated traffic summary is serialized and sent to the operator alongside the existing session metadata. This extends the session-close protocol message:

```rust
pub struct SessionTrafficSummary {
    pub entries: Vec<TrafficSummaryEntry>,
}

pub struct TrafficSummaryEntry {
    pub method: String,
    pub path: String,
    pub status_code: u16,
    pub count: u64,
    pub content_type: Option<String>,
    pub avg_response_size: u64,
    pub body_samples: Vec<BodySample>,
}
```

The field is `Option<SessionTrafficSummary>` for backwards compatibility — old agents without traffic capture send `None`, and the operator omits the traffic section from the comment.

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
        let traffic_summary = session.spec.traffic_summary.clone();
        tokio::spawn(async move {
            if let Err(e) = github.post_session_verification(
                &git_info, &session, duration, traffic_summary.as_ref()
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

The comment includes a session summary table and, when available, a traffic summary with request/response body samples. The operator maintains a **single comment per PR**, identified by a hidden HTML marker (`<!-- mirrord-verification -->`). Each new session **overwrites** the previous comment content with the latest session's data.

For a standard developer session with traffic summary:

```markdown
<!-- mirrord-verification -->
### mirrord verified

| | |
|---|---|
| **Target** | `deploy/checkout-service` |
| **Namespace** | `staging` |
| **User** | `han` |
| **Duration** | 4m 32s |

#### Traffic summary

| Method | Path | Status | Count | Content-Type | Avg Response |
|--------|------|--------|-------|--------------|-------------|
| GET | `/api/users` | 200 | 47 | application/json | 1.2 KB |
| POST | `/api/users` | 201 | 3 | application/json | 256 B |
| POST | `/api/users` | 400 | 2 | application/json | 128 B |

<details>
<summary><b>POST /api/users → 201</b> (latest sample)</summary>

**Request:**
​```json
{"name": "test-user", "email": "test@example.com", "role": "viewer"}
​```

**Response:**
​```json
{"id": 42, "name": "test-user", "email": "test@example.com", "created_at": "2026-02-13T16:41:00Z"}
​```

</details>

<details>
<summary><b>POST /api/users → 400</b> (latest sample)</summary>

**Request:**
​```json
{"name": "", "email": "invalid"}
​```

**Response:**
​```json
{"error": "validation_failed", "details": ["name is required", "email format invalid"]}
​```

</details>

*Tested with [mirrord](https://mirrord.dev)*
```

The traffic section is omitted when:
- The agent predates traffic capture support (backwards compatibility)
- Traffic summary is disabled via Helm configuration

In mirror mode, the traffic table omits response columns (status code, response body, response size) since the agent does not see responses. The table still shows method, path, request count, and request body samples — providing visibility into what was tested even without response data.

For a CI-triggered session, additional fields are included:

```markdown
<!-- mirrord-verification -->
### mirrord verified (CI)

| | |
|---|---|
| **Target** | `deploy/checkout-service` |
| **Namespace** | `staging` |
| **Pipeline** | `e2e-tests` |
| **Triggered by** | `push` |
| **Duration** | 12m 08s |

#### Traffic summary
...

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
- It follows an established pattern in the codebase (~550 lines of new code: ~250 for the operator GitHub module, ~200 for the agent traffic accumulator, ~50 for protocol types, ~50 for comment formatting)

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

- **~~Comment deduplication strategy.~~** Resolved: the operator maintains a single comment per PR (identified by `<!-- mirrord-verification -->` marker) and overwrites it with the latest session's data. The rationale is that the last session represents the developer's final testing state — if something was broken, they'd keep testing until it works. History of previous sessions is not preserved in the comment.

- **Multi-repo sessions.** If a developer's branch exists in a fork rather than the main repo, the `owner/repo` derived from the remote may not match where the PR is open. Need to decide if we should search across forks or require the remote to match exactly.

- **Rate limiting.** GitHub's API rate limit is 5,000 requests/hour for authenticated requests. For a busy cluster with many sessions, we may need to batch or throttle requests. This is unlikely to be a problem in practice (each session is 2 API calls: find PR + post comment), but should be considered.

- **Opt-out mechanism.** Should there be a way for individual developers to opt out of PR comments? For example, a mirrord config flag like `"github_verify": false` that tells the operator to skip the comment for that session.

- **Comment identity.** When using a PAT, the comment appears as the token owner. Should we recommend creating a dedicated machine user (e.g., `mirrord-bot`) for a cleaner appearance? Document this as a best practice?

## Future possibilities
[future-possibilities]: #future-possibilities

- **GitHub Check Runs.** Instead of (or in addition to) comments, create Check Runs that appear in the PR's Checks tab. These can show structured pass/fail data and can be made required via branch protection rules — enabling "must test with mirrord before merging" policies.

- **GitHub App registration.** Provide a MetalBear-registered GitHub App on the GitHub Marketplace that customers can install for branded `mirrord[bot]` identity. The operator would use the App's installation token instead of a PAT. The API calls remain identical.

- **Latency tracking in traffic summary.** In steal mode, the agent could measure round-trip latency (time between request received and response sent) and include p50/p99 latency per endpoint in the traffic summary table. This is not included in the initial implementation because: (a) mirror mode cannot measure latency (agent doesn't see responses), creating an inconsistent experience, and (b) accurate latency measurement in the agent requires careful instrumentation to avoid measuring agent overhead rather than actual service latency. This can be added as a follow-up without protocol changes — the `TrafficSummaryEntry` struct can be extended with optional latency fields.

- **Path normalization.** Automatically collapse high-cardinality paths (e.g., `/api/users/123` → `/api/users/:id`) to reduce the number of unique entries in the traffic table. The initial implementation uses raw paths with a hard cap on unique entries. A future iteration could use heuristics (numeric segments, UUID segments) or allow user-configured path patterns.

- **Session comparison.** Compare the current session against previous sessions for the same target and show regressions (e.g., "latency increased 3x on `/api/checkout`").

- **GitLab and Bitbucket support.** The same pattern applies to other Git hosting platforms. The `github.rs` module could be generalized to a `git_hosting.rs` module with provider-specific implementations.

- **Admin dashboard integration.** Surface PR verification history in the admin dashboard (license-server based). Show which PRs have been verified, by whom, and when — giving team leads a centralized view alongside existing session utilization metrics.

- **Automatic session triggering.** Instead of waiting for developers to manually run mirrord, a webhook could trigger a mirrord session automatically when a PR is opened or updated, running a predefined test suite against the target deployment.
