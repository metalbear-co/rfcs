- Feature Name: `multi_cluster`
- Start Date: 2026-01-09
- Last Updated: 2026-02-09
- RFC PR: [metalbear-co/rfcs#0001](https://github.com/metalbear-co/rfcs/issues/3)
- RFC reference:
  - [metalbear-co/rfcs#0001](https://github.com/metalbear-co/rfcs/pull/4)

## Summary

Enable developers to run mirrord against pods in any Kubernetes cluster they have access to, without switching their local kube context. Workflows that span across multiple clusters become available to use via mirrord.

---

## Demo Videos

<details>
<summary><strong> Demo 1: mirrord Multi-Cluster for Kubernetes: Traffic Stealing (Primary + Remote)</strong></summary>

<p><iframe src="https://youtu.be/d7D1d-B0_vI?si=mPqfFAe2v-bwda9b" width="100%" height="400" allowfullscreen></iframe></p>

</details>

<details>
<summary><strong>Demo 2: mirrord Multi-Cluster for Kubernetes: Traffic Mirroring (Primary + Remote)</strong></summary>

<p><em><iframe src="https://youtu.be/hWNnlqo18uw?si=WD5vNZk2K2vqiz24" width="100%" height="400" allowfullscreen></iframe></em></p>

</details>

<details>
<summary><strong>Demo 3: mirrord Multi-Cluster for Kubernetes: Postgres Test - Primary as Management-Only</strong></summary>

<p><em><iframe src="https://youtu.be/v8POUAFOKs4?si=mTRSWxYwJhydC4mG" width="100%" height="400" allowfullscreen></iframe></em></p>

</details>

<details>
<summary><strong>Demo 4: mirrord Multi-Cluster for Kubernetes: Postgres Test - Direct Connection to Remote Cluster</strong></summary>

<p><em><iframe src="https://youtu.be/h77LBVFb_10?si=6wHq8I2gJNblc0Jm" width="100%" height="400" allowfullscreen></iframe></em></p>

</details>

<details>
<summary><strong>Demo 5: mirrord Multi-Cluster for Kubernetes: Postgres Test - Primary as Default </strong></summary>

<p><em><iframe src="https://youtu.be/ccbjy13EEgQ?si=hnPWu_JN7tpvwSdg" width="100%" height="400" allowfullscreen></iframe></em></p>

</details>

---

## Motivation

### The Problem

Today mirrord only intercepts traffic within the active cluster. In HA or multi-cluster mesh environments, there is no guarantee where requests are routed. The current workaround is to run two separate sessions for two clusters and attach the debugger to the desired one. This is frustrating and unreliable.

### Use Cases

**Multi-region traffic testing**: A developer wants to test their local service against traffic from pods running in us-east-1, eu-west-1, and ap-southeast-1 clusters simultaneously.

**Centralized access**: An organization's security policy requires developers to connect through a central management cluster that has credentials to reach workload clusters. The developer's machine cannot directly reach workload clusters.

---

## Solution Overview

The design is built on the existing mirrord operator. Each operator supports both single-cluster and multi-cluster sessions. A private interface allows one cluster to coordinate sessions across others.

For multi-cluster sessions, a "primary" cluster drives sessions in remote clusters using their local operators over this private interface.

---

## Understanding different Cluster Roles

When setting up multi-cluster mirrord, each cluster takes on one or more roles. Understanding these roles is fundamental to understanding how the system works.

### The Primary Cluster

The Primary cluster is where the user connects if he wants to use multi cluster. When a developer runs `mirrord exec`, their CLI talks to the Primary cluster's operator. This is the "entry point" for all multi-cluster operations.

The Primary cluster runs a component we call the **Envoy**. The Envoy is an orchestrator - it doesn't do the actual traffic interception work, but it coordinates all the clusters that do.

In single-cluster mirrord, there is no Envoy. The operator directly manages the session and agent. In multi-cluster, we need this extra layer because something has to coordinate multiple operators across multiple clusters.

### Workload Clusters

Workload clusters are where your actual application pods run. These clusters have agents that intercept traffic, read files, capture environment variables, and do all the things mirrord normally does.

In single-cluster mirrord, the one cluster you connect to is the workload cluster. In multi-cluster, you define which clusters have workloads.

### The Default Cluster

Some operations in mirrord are "stateful" - meaning they need to return consistent data. For example:

- **Environment variables**: If your app calls `os.Getenv("DATABASE_URL")`, it should get ONE answer, not different answers from different clusters.
- **File operations**: If the app reads `/etc/config.yaml`, it should get ONE file, not a merge of files from multiple clusters.
- **Outgoing connections**: If the app connects to a database, it should connect to ONE database.
- **Database branching**: If we create a database branch for testing, we create ONE branch, not one per cluster.

The Default cluster is the single cluster we designate to handle all these stateful operations. When your app does any of these operations, the request goes ONLY to the Default cluster's agent.

In single-cluster mirrord, you don't need to think about this - there's only one cluster, so all operations go there. In multi-cluster, we need to explicitly choose one cluster to be the "source of truth" for stateful operations.

### Management-Only Mode

Sometimes we want the Primary cluster to ONLY orchestrate, without running any workloads itself. This is called "Management-Only" mode.

Why would we want this? Consider a scenario where we have a management cluster that handles all our DevOps tooling, and separate clusters for our actual applications. The management cluster has the permissions to access all other clusters, but it doesn't run the application pods.

In this case, we set `PRIMARY_MANAGEMENT_ONLY=true`. This means that we want only to orchestrate.

Important: If Primary is Management-Only, you MUST set a different cluster as Default. Because Management-Only means "no resources here", but Default means "stateful operations happen here". You can't have stateful operations on a cluster with no resources.

---

## Installation Steps

This section covers how to do the initial setup. The setup involves selecting a primary cluster, installing the operator with multi-cluster enabled, and configuring access to all remote clusters.

### Authentication Methods

Each remote cluster must specify an `authType` that determines how the primary operator authenticates to it. This field is **required** — the Helm chart validates it at install time and fails if missing or not one of the supported values. There are three authentication methods:

1. **EKS IAM** (`authType: eks`) — For AWS EKS clusters. The primary operator generates short-lived tokens using its IAM role (via IRSA). No secrets to manage — tokens are generated locally and auto-refreshed every 10 minutes. Requires an EKS Access Entry on each remote cluster and `sa.roleArn` on the primary operator.

2. **Bearer Token** (`authType: bearerToken`) — Uses ServiceAccount tokens that are automatically refreshed via the TokenRequest API. The operator auto-refreshes the token before expiration. A Secret with the initial token is needed on the primary cluster.

3. **mTLS** (`authType: mtls`) — For clusters that require client certificate authentication. You provide `tlsCrt` and `tlsKey` in the cluster Secret. Certificates are NOT auto-refreshed.

### Remote Cluster Setup

Each remote cluster needs the mirrord operator installed with `multiClusterMember` enabled. This automatically creates the ServiceAccount, ClusterRole, and ClusterRoleBinding needed for the primary operator to manage sessions on that cluster.

#### Bearer Token / mTLS Clusters

Install the operator on the remote cluster:

```bash
  --set operator.multiClusterMember=true
```

This creates a `mirrord-operator-envoy` ServiceAccount with the necessary permissions. After installation, generate an initial token for the primary cluster:

```bash
kubectl create token mirrord-operator-envoy -n mirrord --duration=24h
```

This initial token is only needed for the first setup. Once the primary operator starts, it automatically refreshes tokens using the TokenRequest API before they expire.

#### EKS IAM Clusters

EKS IAM auth lets the primary operator authenticate to remote EKS clusters using its IAM role instead of ServiceAccount tokens. No Secrets to manage — the operator generates short-lived tokens from its IAM identity.

Setup has two parts: **AWS** (IAM/EKS configuration) and **Operator** (Helm values).

##### AWS

**1. Associate an OIDC Identity Provider with the primary cluster**

The primary EKS cluster's OIDC issuer must be registered as an IAM Identity Provider. This is what allows the operator pod to assume an IAM role via IRSA. Skip this if already done.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <PRIMARY_CLUSTER_NAME> --region <REGION> --approve
```

**2. Create an EKS Access Entry on each remote cluster**

This maps the operator's IAM role to a Kubernetes group on the remote cluster. No IAM policy is attached — permissions come from Kubernetes RBAC.

```bash
aws eks create-access-entry \
  --cluster-name <REMOTE_CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE_NAME> \
  --type STANDARD \
  --kubernetes-groups mirrord-operator-envoy
```

**3. Create (or update) the IAM role trust policy**

The IAM role needs a trust policy that allows the primary operator's ServiceAccount to assume it via IRSA. Replace the placeholders with your account ID and the OIDC issuer ID of the **primary** cluster.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ISSUER_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ISSUER_ID>:sub": "system:serviceaccount:mirrord:mirrord-operator",
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ISSUER_ID>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

You can find the OIDC issuer ID from the primary cluster:

```bash
aws eks describe-cluster --name <PRIMARY_CLUSTER_NAME> --region <REGION> \
  --query "cluster.identity.oidc.issuer" --output text
# Returns: https://oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ISSUER_ID>
```

> **Note:** The role doesn't need any IAM policies (no `iam:*`, `s3:*`, etc.). All permissions come from Kubernetes RBAC via the Access Entry. The trust policy only controls *who can assume the role*.

##### Operator

**Remote clusters** — install with the IAM group binding:

```yaml
operator:
  multiClusterMember: true
  multiClusterMemberIamGroup: mirrord-operator-envoy
```

The `multiClusterMemberIamGroup` creates ClusterRoleBindings that grant the specified Kubernetes group the same permissions as the ServiceAccount-based setup. When set, the ServiceAccount-based ClusterRoleBindings are skipped since authentication comes from IAM instead of ServiceAccount tokens.

**Primary cluster** — set `sa.roleArn` so the operator pod can assume the IAM role via IRSA:

```yaml
sa:
  roleArn: "arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE_NAME>"
```

This annotates the operator's ServiceAccount with `eks.amazonaws.com/role-arn`, which tells the EKS pod identity webhook to inject AWS credentials into the pod. The same IAM role is used for all EKS IAM remote clusters — each remote cluster maps that role via its own Access Entry (step 2 above).

### RBAC — How Permissions Work

When the primary operator connects to a remote cluster, it needs permissions to do things there (list targets, create sessions, check health, etc.). These permissions are set up on each remote cluster using Kubernetes RBAC.

The chart creates two ClusterRoles (permission definitions):

1. **`mirrord-operator-envoy`** — Permissions for general operations: listing targets, managing parent sessions, syncing database branches, reading pods and deployments, and running health checks.

2. **`mirrord-operator-envoy-remote`** — Permissions to create and manage child `MirrordClusterSession` resources. This is what actually allows the primary operator to start mirrord sessions on a remote cluster. This ClusterRole is only granted on member clusters, not on the primary, because the primary never receives child sessions — it only creates them on remote clusters.

A ClusterRole by itself doesn't grant anything — it only defines what actions are possible. To actually give those permissions to someone, we create ClusterRoleBindings that connect the ClusterRole to an identity.

**All remote clusters** need `multiClusterMember=true`. This is always required regardless of auth type. It tells the chart to create the ClusterRoles, the `mirrord-operator-envoy` ServiceAccount, and the ClusterRoleBindings that grant both roles to this ServiceAccount.

**EKS IAM clusters** additionally need `multiClusterMemberIamGroup` set (e.g., `mirrord-operator-envoy`). This tells the chart to also create ClusterRoleBindings that grant both roles to a Kubernetes Group instead of just the ServiceAccount. EKS maps the primary operator's IAM role to this group via the Access Entry. When `multiClusterMemberIamGroup` is set, the ServiceAccount-based bindings are skipped because the identity comes from IAM, not from a ServiceAccount token.

So in practice:

- **Bearer token / mTLS remote**: `multiClusterMember=true`
- **EKS IAM remote**: `multiClusterMember=true` + `multiClusterMemberIamGroup=mirrord-operator-envoy`

### Primary Cluster Helm Configuration

Install the operator on the primary cluster with multi-cluster configuration. Each remote cluster requires an `authType` field. You can either:

1. Provide cluster credentials directly in the Helm values (Helm creates the secrets automatically)
2. Create the secrets manually and let the operator discover them

```yaml
operator:
  multiCluster:
    # Enable multi-cluster mode
    enabled: true

    # Logical name of this cluster
    clusterName: "primary"

    # Default cluster for stateful operations (env vars, files, db branching)
    defaultCluster: "staging-cluster"

    # Set to true if primary cluster has no workloads (management-only)
    managementOnly: true

    # Cluster configuration
    # Each key is the real cluster name as you know it (e.g. the EKS cluster name,
    # the kube context name, etc.). The operator uses this name to identify the cluster.
    clusters:
      # Bearer token authentication
      staging-cluster:
        authType: bearerToken
        server: "https://api.staging.example.com:6443"
        caData: "LS0tLS1CRUdJTi..."  # Base64-encoded CA certificate
        bearerToken: "eyJhbGciOiJS..."  # Initial token from `kubectl create token`
        isDefault: true

      # EKS IAM authentication
      my-eks-cluster:
        authType: eks
        region: eu-north-1
        server: "https://ABCDEF1234567890.gr7.eu-north-1.eks.amazonaws.com"
        caData: "LS0tLS1CRUdJTi..."  # Base64-encoded CA certificate (required for EKS)

      # mTLS authentication
      on-prem-cluster:
        authType: mtls
        server: "https://api.onprem.example.com:6443"
        caData: "LS0tLS1CRUdJTi..."  # Base64-encoded CA certificate
        tlsCrt: "LS0tLS1CRUdJTi..."  # Base64-encoded client certificate PEM
        tlsKey: "LS0tLS1CRUdJTi..."  # Base64-encoded client private key PEM

# Required for EKS IAM auth (see EKS IAM Clusters section above)
sa:
  roleArn: "arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE_NAME>"
```

> **Note:** The cluster key names in the `clusters` map should match the real cluster names. For EKS clusters this is especially important — the operator uses the key as the EKS cluster name when signing IAM tokens. EKS IAM clusters don't need a `bearerToken` (the operator generates tokens from its IAM role).

### Where Data Is Stored

When you provide cluster configuration in the Helm values, the chart splits it into two places:

- **ConfigMap** (`clusters-config.yaml`): non-sensitive cluster configuration — `name`, `server`, `caData`, `authType`, `region`, `isDefault`, `namespace`.
- **Secret** (`mirrord-cluster-<name>`): sensitive credentials only — `bearerToken`, `tls.crt`, `tls.key`. Only created if bearer token or mTLS credentials are provided.

For EKS IAM clusters, no Secret is created — everything is in the ConfigMap since authentication is through IAM.

### Manual Secret Creation

Alternatively, you can create the Secret manually outside of Helm. The Secret must be labeled with `operator.metalbear.co/remote-cluster-credentials=true` and named `mirrord-cluster-<cluster-name>`. The cluster configuration (server, authType, etc.) still needs to be provided via Helm values or the `clusters-config.yaml` ConfigMap.

For Bearer Token authentication:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mirrord-cluster-staging-cluster
  namespace: mirrord
  labels:
    operator.metalbear.co/remote-cluster-credentials: "true"
type: Opaque
stringData:
  bearerToken: "eyJhbGciOiJS..."
```

For mTLS authentication:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mirrord-cluster-on-prem-cluster
  namespace: mirrord
  labels:
    operator.metalbear.co/remote-cluster-credentials: "true"
type: Opaque
stringData:
  tls.crt: "LS0tLS1CRUdJTi..."  # Client certificate PEM
  tls.key: "LS0tLS1CRUdJTi..."  # Client private key PEM
```

Note: EKS IAM clusters do NOT need a Secret — they authenticate using the operator's IAM role and only need the cluster configuration in the ConfigMap.

### Verify the connection succeeded

```sh
kubectl --context {PRIMARY_CLUSTER} get mirrordoperators operator -o yaml
```

You need to have something similar, that will show you the clusters that are available and their status. If they are successfuly connected you will see `license_fingerprint` and `operator_version`, if not there will be an error field there.

```yaml
apiVersion: operator.metalbear.co/v1
kind: MirrordOperator
metadata:
  name: operator
spec:
  copy_target_enabled: true
  default_namespace: default
  features:
  - ProxyApi
  license:
    expire_at: "2026-08-07"
    fingerprint: S1hgumDqNoyDarUX7k31ALtxcOuJ9qhlc+HfJQUV4CE
    name: Oaks of Rogalin`s Enterprise License
    organization: Oaks of Rogalin
    subscription_id: 2c8d96be-c3bf-458c-9d35-a0642c2c2a77
  operator_version: 3.137.0
  protocol_version: 1.25.0
  supported_features:
  - ProxyApi
  - CopyTarget
  - CopyTargetExcludeContainers
  - LayerReconnect
  - ExtendableUserCredentials
  - BypassCiCertificateVerification
  - SqsQueueSplitting
  - SqsQueueSplittingDirect
  - MultiClusterPrimary
status:
  connected_clusters:
  - connected:
      license_fingerprint: S1hgumDqNoyDarUX7k31ALtxcOuJ9qhlc+HfJQUV4CE
      operator_version: 3.137.0
    lastCheck: "2026-01-27T09:22:55.824115967+00:00"
    name: remote-1
  - connected:
      license_fingerprint: S1hgumDqNoyDarUX7k31ALtxcOuJ9qhlc+HfJQUV4CE
      operator_version: 3.137.0
    lastCheck: "2026-01-27T09:22:55.766848498+00:00"
    name: remote-2
  copy_targets: []
  sessions: []
  statistics:
    dau: 0
    mau: 0
```

### Token Refresh

Token refresh behavior depends on the authentication method:

**Bearer Token** (`authType: bearerToken`):

1. On startup, the operator reads all cluster secrets and builds an in-memory connection for each cluster
2. It parses the JWT token to determine its original lifetime (from `exp` and `iat` claims)
3. It monitors the token expiration and requests new tokens using the TokenRequest API
4. The refresh happens with a buffer before expiration to ensure uninterrupted connectivity
5. New tokens maintain the same lifetime as the original token

You only need to generate tokens once during initial setup. The operator handles all subsequent renewals automatically.

**EKS IAM** (`authType: eks`):

1. The operator generates a presigned `GetCallerIdentity` STS URL using its IAM role credentials (acquired via IRSA)
2. This presigned URL is base64-encoded and used as a Kubernetes bearer token (the `k8s-aws-v1.` prefixed token format that EKS expects)
3. Tokens are regenerated every 10 minutes — well within the 15-minute presigned URL validity window
4. No Secrets are involved — credentials come from the pod's IAM role

**mTLS** (`authType: mtls`):
Certificates are NOT auto-refreshed. You are responsible for rotating the client certificate and key in the cluster Secret before they expire.

---

## Developer Experience

From a developer's perspective, multi-cluster mirrord is completely transparent. Developers do not need to know which cluster is primary or which is the default. They use mirrord exactly the same way as single-cluster.

### How Developers Use It

The workflow is identical to single-cluster:

1. **Authenticate with the Primary cluster** - The developer's local machine is bound to it through their kubeconfig.

2. **Run mirrord** - Execute `mirrord exec ./myapp` as usual.

3. **The operator handles the rest** - Based on the operator's configuration, it determines whether to use single-cluster or multi-cluster mode.

The CLI automatically detects multi-cluster from the operator's configuration and behaves accordingly. No special flags or configuration are needed from the developer.

### What Developers Get

With multi-cluster enabled, developers benefit from:

- **Incoming traffic from all clusters** - Traffic mirroring and stealing work across all workload clusters. If a request arrives at any cluster, the developer's local app receives it.

- **Consistent environment** - Environment variables, files, and database connections all come from one source (the Default cluster), ensuring predictable behavior.

- **Database branching** - Branches are created once in the Default cluster and used consistently, regardless of which cluster handles traffic.

- **Single session** - No need to manage multiple sessions or switch contexts. The complexity is hidden behind one session ID.

### Configuration

The developer's mirrord configuration file does not need any multi-cluster-specific settings:

```json
{
  "target": "deployment/myapp"
}
```

The operator determines single-cluster or multi-cluster mode based on its installation configuration, not the developer's config file. This keeps the developer experience simple and consistent.

---

## The Parent-Child Session Model

In single-cluster mirrord, we have a Custom Resource (CR) called `MirrordClusterSession`. This represents a single mirrord session with a single agent in a single cluster. When the CLI connects, the operator creates this CR, spawns an agent, and the session is active.

In multi-cluster, we need something more. We introduce a new CR called `MirrordMultiClusterSession`. This is like a "parent" session that coordinates multiple "child" sessions.

### Why Do We Need a Parent Session?

The CLI expects to interact with ONE session. It doesn't want to manage multiple session IDs, multiple WebSocket connections, or worry about which cluster does what. From the CLI's perspective, it's just one mirrord session.

But internally, we need a session in each workload cluster. Each cluster needs its own `MirrordClusterSession` so that cluster's operator can spawn an agent and handle requests like the usually does.

The parent `MirrordMultiClusterSession` presents one session ID to the CLI, while internally managing child sessions across multiple clusters.

This is different from single-cluster where one `MirrordClusterSession` is enough. In multi-cluster, the parent session acts as a coordinator of multiple child sessions, each of which looks like a normal single-cluster session to its respective operator.

### The Naming Convention

Child sessions follow a simple naming pattern. If the parent session is called `mc-4dbe55ab68bba16c`, then the child sessions are named by appending the cluster name: `mc-4dbe55ab68bba16c-remote-1` for the remote-1 cluster, `mc-4dbe55ab68bba16c-remote-2` for remote-2, and so on.

This naming makes it easy to identify which parent a child belongs to, and makes cleanup straightforward since we can find all children by their prefix.

---

## The MultiClusterSessionManager

The `MultiClusterSessionManager` is the controller that manages `MirrordMultiClusterSession` resources. It lives in the Primary cluster's operator and is responsible for the entire lifecycle of multi-cluster sessions.

In single-cluster mirrord, session management is handled by the `BaseController`. The `MultiClusterSessionManager` follows similar patterns but adds the complexity of coordinating across clusters.

### The Reconciliation Loop

The controller runs a "reconciliation" function whenever something changes. This function looks at the current state of a session and decides what to do next.

The session goes through several phases. It starts as "Initializing" when just created - this is when we need to create child sessions in all workload clusters and potentially create database branches. It moves to "Pending" once we've started creating child sessions but not all are ready yet. It becomes "Ready" when all child sessions are active. It can become "Failed" if something goes wrong, or "Terminating" when being deleted.

### CR and In-Memory State

We maintain session state in two places: the `MirrordMultiClusterSession` CR and an in-memory `DashMap`. The CR is our source of truth - it survives operator restarts and is visible via kubectl. The in-memory map is a performance cache that avoids slow Kubernetes API calls when routing WebSocket messages. Every time the controller reconciles a session, it syncs the in-memory map from the CR, ensuring they stay consistent. When cleaning up, we always read from the CR status (not memory) because if the operator restarted, the memory would be empty but the CR still knows which child sessions exist.

---

## The MultiClusterRouter

The MultiClusterRouter is one of the most important components in multi-cluster mirrord. It's what makes multi-cluster feel like single-cluster to the CLI and intproxy.

### What It Does

In single-cluster mirrord, the operator has a component called `AgentRouter`. The AgentRouter manages communication between the CLI/intproxy and the agent. It receives messages from the client, forwards them to the agent, and sends responses back. It presents an interface called `AgentConnection` which has a sender channel (to send messages to the agent) and a receiver channel (to receive messages from the agent).

The MultiClusterRouter does the same thing, but for multiple agents across multiple clusters. It presents the same `AgentConnection` interface, so the rest of the operator code doesn't need to know it's dealing with multi-cluster. This is intentional as we want to reuse as much existing code as possible.

### How It Routes Messages

When the router receives a message from the client, it needs to decide where to send it. This decision is based on the type of operation.

Stateful operations - file reads/writes, environment variable requests, outgoing TCP/UDP connections, DNS lookups - go ONLY to the Default cluster. This ensures consistency. If the app reads an environment variable, it always gets the same answer because it always comes from the same cluster.

Traffic operations - things like mirror and steal requests - go to ALL workload clusters. This makes sense because traffic can arrive at any cluster, so all agents need to know about traffic interception rules.

The code that makes this decision is simple. We have a function `is_stateful_operation` that checks the message type:

```rust
pub fn is_stateful_operation(msg: &ClientMessage) -> bool {
    matches!(
        msg,
        ClientMessage::FileRequest(_)
            | ClientMessage::GetEnvVarsRequest(_)
            | ClientMessage::GetAddrInfoRequest(_)
            | ClientMessage::GetAddrInfoRequestV2(_)
            | ClientMessage::Vpn(_)
    )
}

pub fn is_outgoing_traffic(msg: &ClientMessage) -> bool {
    matches!(
        msg,
        ClientMessage::TcpOutgoing(_) | ClientMessage::UdpOutgoing(_)
    )
}
```

Both stateful operations and outgoing traffic go to the Default cluster only. Outgoing traffic (TCP/UDP) is routed to one cluster to prevent duplicate responses — all clusters share the same external services, so one agent can handle all outgoing requests. Everything else (traffic mirroring/stealing, control messages) is broadcast to all workload clusters.

### Merging Responses

Messages come back from multiple agents. The router merges all these responses into a single stream that goes back to the client. The client doesn't know responses are coming from multiple sources - it just sees a stream of messages like it would in single-cluster.

### Comparison with Single-Cluster

In single-cluster, the `AgentRouter` talks to one agent. In multi-cluster, the `MultiClusterRouter` talks to multiple operators (which each have their own agent). But both present the same `AgentConnection` interface. This means the `LayerHandler` and other components that use `AgentConnection` work unchanged in multi-cluster.

---

## Connection Health and Ping-Pong

Each operator needs to know if the connection to its agent is still alive. It does this by sending `Ping` messages every few seconds. The agent responds with `Pong`. If the operator doesn't receive a pong for 60 seconds, it assumes something is wrong and terminates the agent.

### The Problem in Multi-Cluster

In multi-cluster, there are multiple operators (one per cluster), each with its own agent. The CLI sends ping messages, and the Primary operator's router decides where to send them.

If pings only go to one cluster (the Default/Primary cluster), the other clusters never receive any pings. Their operators wait, receive nothing for 60 seconds, and then kill their agents. This is why remote agents were disappearing after exactly 60 seconds.

### The Solution

Ping messages are broadcast to ALL clusters. Every cluster receives the ping, every agent responds with pong, and every operator knows its connection is alive. The CLI receives multiple pong responses (one per cluster), but that's fine - it just means all clusters are healthy.

### WebSocket Keep-Alive

The Primary operator also sends low-level WebSocket ping frames to each remote operator every 30 seconds. This keeps the network connection itself alive and helps detect if a remote cluster becomes unreachable.

---

## The Session Lifecycle

Let's walk through what happens from session creation to cleanup, comparing with single-cluster where relevant.

### Step-by-Step Example

Here is the full flow of a multi-cluster session from start to finish. The developer runs `mirrord exec ./myapp` targeting `deployment/myapp`, with the Primary cluster configured to orchestrate two workload clusters (`cluster-a` and `cluster-b`), where `cluster-a` is the Default cluster.

**Session Creation**

1. The developer runs `mirrord exec ./myapp`. The CLI connects to the Primary cluster's operator.
2. The CLI detects multi-cluster mode from the operator's configuration. It does **not** resolve the target locally — target resolution happens later on each workload cluster.
3. If the developer has database branching configured, the CLI creates branch CRs first, waits for them to be ready, and collects the branch names. See the Database Branching step-by-step below. If no branching is configured, this step is skipped.
4. The CLI builds `ConnectParams` containing branch names, SQS configuration, and other session parameters. It opens a request to the Primary operator to create a multi-cluster session.
5. The Primary operator creates a `MirrordMultiClusterSession` CR with phase `Initializing` and returns the session ID to the CLI.

**Initialization**

1. The `MultiClusterSessionManager` controller sees the new CR and starts reconciliation. For an `Initializing` session, it spawns a background task.
2. The background task iterates over the configured workload clusters (`cluster-a` and `cluster-b`). For each cluster, it connects to that cluster's operator using the configured authentication method (bearer token, EKS IAM, or mTLS).
3. On each workload cluster, the Primary operator creates a child `MirrordClusterSession` CR. The child session name follows the pattern `mc-<parent-id>-<cluster-name>` (e.g. `mc-4dbe55ab68bba16c-cluster-a`). The child spec includes the target (`deployment/myapp`), namespace, and `ConnectParams`. Branch names and SQS config are only forwarded to the Default cluster (`cluster-a`).
4. Each remote cluster's operator processes the child CR as a normal single-cluster session — it resolves the target (finds the pod for `deployment/myapp`), spawns an agent, and marks the child session as `Ready`.

**Session Becomes Ready**

1. As each child session becomes ready, the Primary operator updates the parent CR's status with the child session name, cluster, and readiness.
2. Once all child sessions are ready, the parent CR's phase changes from `Initializing` to `Ready`.
3. The CLI sees the session is ready and establishes a WebSocket connection to the Primary operator.
4. The Primary operator creates a `MultiClusterRouter` for this session, which presents the same `AgentConnection` interface as single-cluster.

**Active Session**

1. Messages flow through the `MultiClusterRouter`. Stateful operations (env vars, file reads, DNS, outgoing connections) go only to the Default cluster (`cluster-a`). Traffic operations (mirror/steal) are broadcast to all workload clusters (`cluster-a` and `cluster-b`).
2. A background heartbeat task updates the `connectedTimestamp` on the parent CR every 10 seconds.
3. Ping messages are broadcast to all clusters so every agent knows the connection is alive.

**Session Cleanup**

1. The developer's application exits. The WebSocket closes and the heartbeat task stops.
2. The `connectedTimestamp` stops being updated. The controller reconciles and calculates the time since the last heartbeat.
3. After 60 seconds (the default TTL) without a heartbeat, the controller deletes the parent CR.
4. Deleting the CR triggers the Kubernetes finalizer. The finalizer reads child session information from the CR status (not from memory, in case the operator restarted).
5. For each child session, the Primary operator connects to the respective remote cluster and deletes the child `MirrordClusterSession` CR. The remote operator cleans up the agent.
6. After all children are deleted, the finalizer is removed from the parent CR and Kubernetes completes the deletion.

### CLI Creates a Session

The user runs `mirrord exec`. The CLI sends a request to the Primary operator asking to create a multi-cluster session. The request includes the target alias (like "deployment.myapp") and the mirrord configuration.

In single-cluster, the CLI sends a similar request, and the operator immediately creates a `MirrordClusterSession` and starts spawning an agent.

In multi-cluster, the operator creates a `MirrordMultiClusterSession` CR with phase "Initializing" and returns the session ID to the CLI. The actual work happens asynchronously.

### Initialization

The controller sees the new CR and starts the reconciliation process. For an "Initializing" session, it spawns a background task to do the actual work.

Note: if the developer is using database branching, the branches are already created before session creation. The CLI creates branch CRs on the Primary cluster, the `DbBranchSyncController` syncs them to the Default cluster, and once ready the CLI receives the branch names back. The CLI then sends these branch names via `ConnectParams` when creating the session — the same flow as single-cluster. See the Database Branching section for details.

The controller creates child sessions in each workload cluster. For each cluster, it connects to that cluster's operator and creates a `MirrordClusterSession` CR. The child session spec includes the target (which pod or deployment) and the namespace. Branch names and SQS configuration are passed via `ConnectParams` when the primary connects to each remote operator — the same mechanism the CLI uses in single-cluster.

We then wait for each child session to become ready, meaning its agent has spawned and is connected.

### Parent Status Update

As each child session becomes ready, we update the parent CR's status. The status contains a map of all child sessions with their names, readiness, and any errors. Once all children are ready, we set the parent's phase to "Ready".

In single-cluster, there's no parent status to update - the single session's status is enough.

### CLI Connects via WebSocket

Once the session is Ready, the CLI establishes a WebSocket connection to the Primary operator. The operator looks up the session in the in-memory map and creates a `MultiClusterRouter` to handle message routing.

In single-cluster, the operator creates an `AgentRouter` instead. But both return an `AgentConnection` interface, so the code that handles the WebSocket doesn't need to know the difference.

### Active Session

While the session is active, the MultiClusterRouter routes messages to the appropriate clusters. Stateful operations go to the Default cluster. Traffic operations go to all workload clusters.

A background task periodically updates the `connectedTimestamp` in the CR. This happens every 10 seconds. This timestamp is our "heartbeat" - it tells the controller the session is still in use.

In single-cluster, a similar mechanism exists but uses an in-memory reference count (`UseGuard`). In multi-cluster, we use CR timestamps because they survive operator restarts and work in HA setups with multiple operator replicas.

### Client Disconnects

When the application exits, the WebSocket closes. The heartbeat task stops updating `connectedTimestamp`.

### TTL Expiration

The controller continues reconciling the session. On each reconciliation, it calculates how long since the last heartbeat. If more than 60 seconds have passed (the default TTL), it deletes the session CR.

In single-cluster, cleanup happens when the last `UseGuard` reference is dropped. In multi-cluster, we can't rely on in-memory references because of HA and restarts, so we use TTL-based cleanup instead.

### Cleanup via Finalizer

Deleting the CR triggers the finalizer. Kubernetes finalizers ensure cleanup code runs BEFORE the resource is actually deleted.

The cleanup process reads the child session information from the CR status - not from memory. This is important because if the operator restarted, the in-memory state would be empty, but the CR still knows which child sessions exist.

For each child session in the status, we connect to that cluster's operator and delete the child `MirrordClusterSession`. The remote cluster's operator then cleans up its agent.

After all children are deleted, the finalizer is removed from the CR, and Kubernetes completes the parent deletion.

In single-cluster, cleanup is simpler - just delete the one session and its agent.

---

## Database Branching in Multi-Cluster

In single-cluster mode, this is straightforward: the CLI creates a branch CR, the operator creates a temporary database, and the developer's application connects to it. In multi-cluster mode, this becomes more complex because the database can live on a different cluster than where the developer connects.

### Step-by-Step Example (Primary != Default)

This is the most common multi-cluster database branching scenario. The developer connects to the Primary cluster, but the application that uses the database runs on the Default cluster (`cluster-a`). The developer has PostgreSQL branching configured in their `mirrord.json`.

**Branch Creation**

1. The developer runs `mirrord exec ./myapp`. The CLI reads the `mirrord.json` configuration and sees that database branching is enabled for PostgreSQL.
2. The CLI calls `prepare_pg_branch_dbs()`. This creates a `PgBranchDatabase` CR on the Primary cluster with the branch spec (source database, owner, etc.).
3. The `DbBranchSyncController` on the Primary cluster detects the new `PgBranchDatabase` CR.
4. The sync controller creates a copy of this CR on the Default cluster (`cluster-a`).
5. The Default cluster's PostgreSQL branching controller detects the new CR and starts creating the branch — it spins up a temporary database pod and copies the schema/data from the source etc..
6. Once the branch is ready, the Default cluster's controller updates the CR status to `Ready` with the branch name (e.g. `postgres-test-pg-branch-5www5`).
7. The sync controller detects the status change on the Default cluster's CR and copies it back to the Primary cluster's CR.
8. The CLI's `await_condition()` polling loop sees the Primary CR status change to `Ready`.
9. The CLI extracts the branch name from the CR status.

**Passing Branch Names to the Session**

1. The CLI builds `ConnectParams` containing `pg_branch_names: ["postgres-test-pg-branch-5www5"]`.
2. The CLI opens a WebSocket connection to the Primary operator with the `ConnectParams` encoded in the URL query string.
3. The Primary operator creates the `MirrordMultiClusterSession` CR and starts creating child sessions.
4. When creating the child session on the Default cluster (`cluster-a`), the Primary operator forwards the `ConnectParams` including the branch names. Non-default clusters do not receive branch names.

**Environment Variable Interception**

1. The developer's application starts and reads `DATABASE_URL` from the environment.
2. The `MultiClusterRouter` on Primary routes this `GetEnvVarsRequest` to the Default cluster only (stateful operation).
3. The Default cluster's operator reads the real environment variables from the target pod, e.g. `DATABASE_URL=postgres://db.prod:5432/mydb`.
4. The operator checks the session's branch configuration and sees there is a PostgreSQL branch named `postgres-test-pg-branch-5www5`.
5. The operator modifies the response: `DATABASE_URL=postgres://db.prod:5432/postgres-test-pg-branch-5www5`.
6. The application receives the modified connection string and connects to the branch database instead of production — completely transparently.

**Branch Cleanup**

1. The developer's application exits. The session is cleaned up (see Session Lifecycle above).
2. The branch's TTL starts counting down. While the session was active, the TTL was continuously extended.
3. After the TTL expires (e.g. 30 minutes), the Default cluster's branching controller deletes the branch CR and cleans up the temporary database pod.
4. The `DbBranchSyncController` detects the deletion on the Default cluster and deletes the corresponding CR on the Primary cluster.

> **Note:** If Primary **is** the Default cluster some steps are skipped because the branching controllers run directly on Primary and process the CRs locally, just like single-cluster. The sync controller does not run.

### How Database Branching Works in Single-Cluster

In single-cluster mirrord, the CLI has direct access to the Kubernetes cluster where the pod it's connected to runs. When a developer configures database branching in their `mirrord.json` file, the CLI creates a CR such as `PgBranchDatabase` for PostgreSQL or `MysqlBranchDatabase` for MySQL etc... The operator watches for these CRs and responds by creating the branch. It spins up a temporary database pod and updates the CR's status to indicate the branch is ready. The CLI waits for this status change before proceeding. Once the branch is ready, the operator knows to intercept environment variable requests from the application and modify database connection strings to point to the branch instead of the production database.

### The Multi-Cluster Challenge

In multi-cluster mode, the developer's CLI connects to the Primary cluster, but we don't want to create the same branch on all child clusters so instead we only do that on the Default cluster. Also the Primary cluster can be a management-only cluster that orchestrates sessions but doesn't host application workloads or resources. This creates a fundamental problem: the CLI cannot directly create branch CRs on the Default cluster because the developer only has access to the Primary cluster.

### Configuration-Based Controller Deployment

Branching controllers only run on the cluster designated to handle stateful operations (the Default cluster). The CLI creates branch CRs on the Primary cluster, and how they are processed depends on the cluster configuration.

If Primary is the Default cluster, the branching controllers run locally on Primary and process the CRs directly. The CLI creates CRs, Primary's branching controllers create the actual database branches, and everything happens on one cluster just like single-cluster mode.

If Primary is not the Default cluster, the branching controllers do not run on Primary. Instead, a synchronization controller on Primary mirrors the CRs to the Default cluster, where the actual branching controllers run and create the database branches.

The decision of which controllers to run is made at startup time based on configuration. When the operator starts, it loads the multi-cluster configuration and compares `cluster_name` (this cluster's name) with `default_cluster` (where stateful operations happen):

- **Single-cluster mode**: Branching controllers run and handle everything locally. No sync controller is needed.
- **Multi-cluster, Primary == Default**: Branching controllers run on Primary. The `DbBranchSyncController` does not run.
- **Multi-cluster, Primary != Default**: Branching controllers do not run on Primary. Instead, the `DbBranchSyncController` runs to sync CRs to the Default cluster where the actual branching controllers handle database creation.

### The Database Branch Sync Controller

When Primary is not the Default cluster, the Primary cluster runs a controller called `DbBranchSyncController`. This controller watches for branch CRs on Primary and synchronizes them to the Default cluster. The controller supports PostgreSQL, MySQL, and MongoDB branches.

When the controller sees a new branch CR on the Primary cluster, it creates a copy of that CR on the Default cluster. The Default cluster's branching controller (which runs only on Default) processes this CR and creates the actual database branch.

The synchronization is bidirectional for status. Once the Default cluster's operator creates the branch and updates the CR's status to indicate readiness, the sync controller copies that status back to the Primary cluster's CR. This allows the CLI, which is watching the Primary cluster's CR, to know when the branch is ready even though the actual creation happened on a remote cluster.

### CLI

From the CLI's perspective, multi-cluster database branching is identical to single-cluster. The CLI creates branch CRs on the cluster it has access to and waits for the status to become Ready or Failed. After creating the CRs, the CLI enters a waiting loop. It watches the CR status and waits for either a "Ready" or "Failed" phase. The waiting has a configurable timeout. If the branch doesn't become ready within this timeout, the CLI fails.

### Passing Branch Names to Sessions

Once the branches are ready, the CLI passes the branch names to the Primary operator via `ConnectParams`, the same way it works in single-cluster. The Primary operator forwards these params when creating the child session on the Default cluster. Branch names are not stored in the session CRs — they flow through the connection parameters just like in single-cluster mode.

### Environment Variable Interception

In multi-cluster mode, this request is handled by the `MultiClusterRouter`. Because reading environment variables is a stateful operation (we want consistent answers regardless of which cluster handles the request), the router sends this request only to the Default cluster. This is the same cluster where the database branch was provisioned and where the session has the branch configuration in its ephemeral state.

The Default cluster's operator receives the request and handles it with branch awareness. It reads the actual environment variables from the target pod, which might include something like `DATABASE_URL=postgres://db.prod:5432/mydb`. Before returning this to the application, the operator checks if there are any branch overrides configured for this session. If the session has a PostgreSQL branch named `branch_abc123`, the operator modifies the response to return `DATABASE_URL=postgres://db.prod:5432/branch_abc123` instead.

The application receives this modified connection string and connects to the branch database, completely unaware that anything special happened. From the application's perspective, it simply read an environment variable and got a database URL. The fact that this URL points to a temporary branch rather than the production database is invisible to the application code.

### Branch Lifecycle and Cleanup

Database branches have a time-to-live (TTL) that controls how long they exist. In a typical configuration, a branch might have a TTL of 30 minutes, meaning it will be automatically deleted 30 minutes after the session that created it ends.

While the session is active, the branch's TTL is continually extended. The branching controller watches for active sessions and extends the TTL as long as the session is alive. This prevents branches from expiring while a developer is still using them.

When the session ends (either because the developer's application exited or because of a disconnection), the TTL extension stops. The branch continues to exist for its remaining TTL, giving the developer time to reconnect if needed. Once the TTL expires, the Default cluster's branching controller deletes the branch and cleans up any associated resources.

In multi-cluster mode, we also need to handle cleanup of the Primary cluster's CR. The `DbBranchSyncController` handles this through bidirectional deletion synchronization. If the Default cluster deletes a branch CR (due to TTL expiration), the sync controller detects this and deletes the corresponding CR on the Primary cluster. Similarly, if someone deletes the Primary cluster's CR, the sync controller deletes the copy on the Default cluster. This ensures that both clusters stay consistent and that orphaned CRs don't accumulate.

---

## SQS Queue Splitting in Multi-Cluster

SQS queue splitting works across clusters. In single-cluster, the operator creates a temporary SQS queue, patches the target workload to consume from it, and filters messages based on the developer's configuration. In multi-cluster, the same thing happens on each workload cluster — the operator on each cluster creates its own temporary queue and patches the local workload.

The primary operator passes the SQS split configuration (queue IDs and message filters) to each child session via `ConnectParams`. It also passes the `sqs_output_queues` map, which tells each cluster the temporary queue names created by other clusters. This allows the output queue routing to work across cluster boundaries.

Each workload cluster needs AWS credentials with SQS permissions to create and manage SQS queues. The operator uses the default AWS credential chain, so credentials can come from IRSA (`sa.roleArn`), the node's IAM instance profile, EKS Pod Identity, or environment variables — whatever is available in the cluster. This is independent of the multi-cluster auth type — even if the cluster uses bearer token auth for the primary operator's connection, it still needs its own AWS credentials for SQS operations.

---

## Summary

Multi-cluster mirrord adds several new components on top of single-cluster:

The **Envoy** orchestrates multi-cluster sessions, running on the Primary cluster. It coordinates child sessions across multiple workload clusters.

The **MirrordMultiClusterSession** is the Custom Resource that represents the parent session, tracking child sessions in each cluster. It stores configuration, branch names, and status.

The **MultiClusterSessionManager** is the controller that manages parent sessions. It handles initialization, creates child sessions, and coordinates cleanup.

The **MultiClusterRouter** routes messages between the client and multiple workload operators. It sends stateful operations to the Default cluster and broadcasts traffic operations to all workloads.

The key insight is that we reuse as much single-cluster code as possible. The MultiClusterRouter presents the same `AgentConnection` interface as the single-cluster AgentRouter. Child `MirrordClusterSession` resources are normal sessions from each cluster's perspective. The complexity of multi-cluster is hidden in the Primary cluster's Envoy, while everything else works like single-cluster.
