- Feature Name: `multi_cluster`
- Start Date: 2026-01-09
- Last Updated: 2026-01-21
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

Multi-cluster authentication uses ServiceAccounts for remote clusters and Secrets for the primary cluster. There are two authentication methods:

1. **Bearer Token** - Uses ServiceAccount tokens that are automatically refreshed via the TokenRequest API. You only need to setup a ServiceAccount and generate an initial token during setup. After that, the operator automatically refreshes the token before expiration.

2. **mTLS (mutual TLS)** - For clusters that require client certificate authentication. The you can provide `clientCertData` and `clientKeyData`.

Also you can provide `caData` (the server's CA certificate) if it's required.

### Remote Cluster Setup

For each remote cluster that will participate in multi-cluster sessions, you must create a ServiceAccount with appropriate permissions. This is a one-time setup per cluster.

Apply the following manifests to each remote cluster (final permissions TBD):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mirrord
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mirrord-primary-access
  namespace: mirrord
  labels:
    app.kubernetes.io/name: mirrord-operator
    app.kubernetes.io/component: multi-cluster
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mirrord-primary-access
  labels:
    app.kubernetes.io/name: mirrord-operator
    app.kubernetes.io/component: multi-cluster
rules:
  # Session CRD management
  - apiGroups: ["mirrord.metalbear.co"]
    resources: ["mirrordclustersessions"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["mirrord.metalbear.co"]
    resources: ["mirrordclustersessions/status"]
    verbs: ["get", "update", "patch"]
  # Token refresh for credential rotation
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
  # Target workload resolution
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  # Operator target API access
  - apiGroups: ["operator.metalbear.co"]
    resources: ["targets", "copytargets", "targets/port-locks"]
    verbs: ["get", "list", "proxy"]
  - apiGroups: ["operator.metalbear.co"]
    resources: ["copytargets"]
    verbs: ["create"]
  - apiGroups: ["operator.metalbear.co"]
    resources: ["mirrordoperators"]
    verbs: ["get"]
  # Queue registry sync from primary cluster (SQS splitting)
  - apiGroups: ["queues.mirrord.metalbear.co"]
    resources: ["mirrordworkloadqueueregistries"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mirrord-primary-access
  labels:
    app.kubernetes.io/name: mirrord-operator
    app.kubernetes.io/component: multi-cluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mirrord-primary-access
subjects:
  - kind: ServiceAccount
    name: mirrord-primary-access
    namespace: mirrord
```

After creating the ServiceAccount, generate an initial token:

```bash
kubectl create token mirrord-primary-access -n mirrord --duration=24h
```

This initial token is only needed for the first setup. Once the operator starts, it automatically refreshes tokens using the TokenRequest API before they expire, matching the original token's lifetime.

### Primary Cluster Helm Configuration

Install the operator on the primary cluster with multi-cluster configuration. You can either:

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
    defaultCluster: "us-east-1"

    # Set to true if primary cluster has no workloads (management-only)
    primaryIsManagementOnly: true

    # Cluster credentials (optional - can also create secrets manually)
    # If provided, Helm will create the secrets automatically
    clusters:
      # Bearer token authentication
      us-east-1:
        server: "https://api.us-east-1.example.com:6443"
        caData: "LS0tLS1CRUdJTi..."  # Base64-encoded CA certificate
        bearerToken: "eyJhbGciOiJS..."  # Initial token from `kubectl create token`

      # mTLS authentication
      eu-west-1:
        server: "https://api.eu-west-1.example.com:6443"
        caData: "LS0tLS1CRUdJTi..."  # Base64-encoded CA certificate
        clientCertData: "LS0tLS1CRUdJTi..."  # Base64-encoded client certificate
        clientKeyData: "LS0tLS1CRUdJTi..."  # Base64-encoded client private key
```

### Manual Secret Creation

Alternatively, you can create cluster secrets manually. Secrets must be labeled with `mirrord.metalbear.co/remote-cluster=true` for the operator to discover them.

For Bearer Token authentication:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-us-east-1
  namespace: mirrord
  labels:
    mirrord.metalbear.co/remote-cluster: "true"
type: Opaque
stringData:
  server: "https://api.us-east-1.example.com:6443"
  caData: "LS0tLS1CRUdJTi..."
  bearerToken: "eyJhbGciOiJS..."
```

For mTLS authentication:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-eu-west-1
  namespace: mirrord
  labels:
    mirrord.metalbear.co/remote-cluster: "true"
type: Opaque
stringData:
  server: "https://api.eu-west-1.example.com:6443"
  caData: "LS0tLS1CRUdJTi..."
  clientCertData: "LS0tLS1CRUdJTi..."
  clientKeyData: "LS0tLS1CRUdJTi..."
```

### Token Refresh

When using Bearer Token authentication, the operator automatically refreshes tokens before they expire. The refresh process:

1. On startup, the operator reads all cluster secrets and builds an in-memory connection for each cluster
2. It parses the JWT token to determine its original lifetime (from `exp` and `iat` claims).
3. We monitor the token expiration and requests new tokens using the TokenRequest API
4. The refresh happens with a buffer before expiration to ensure uninterrupted connectivity
5. New tokens maintain the same lifetime as the original token

This means you only need to generate tokens once during initial setup. The operator handles all subsequent renewals automatically.

---

## Developer Experience

From a developer's perspective, multi-cluster mirrord is completely transparent. Developers do not need to know which cluster is primary or which is the default. They use mirrord exactly the same way as single-cluster.

### How Developers Use It

The workflow is identical to single-cluster:

1. **Authenticate with the Primary cluster** - The developer's local machine is bound to it through their kubeconfig.

2. **Run mirrord** - Execute `mirrord exec ./myapp` as usual.

3. **The operator handles the rest** - Based on the operator's configuration, it determines whether to use single-cluster or multi-cluster mode.

The CLI automatically detects multi-cluster from the operator's CRD and behaves accordingly. No special flags or configuration are needed from the developer.

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

In single-cluster mirrord, we have one CRD called `MirrordClusterSession`. This represents a single mirrord session with a single agent in a single cluster. When the CLI connects, the operator creates this CRD, spawns an agent, and the session is active.

In multi-cluster, we need something more. We introduce a new CRD called `MirrordMultiClusterSession`. This is like a "parent" session that coordinates multiple "child" sessions.

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

The `MultiClusterSessionManager` is the controller that manages `MirrordMultiClusterSession` CRDs. It lives in the Primary cluster's operator and is responsible for the entire lifecycle of multi-cluster sessions.

In single-cluster mirrord, session management is handled by the `BaseController`. The `MultiClusterSessionManager` follows similar patterns but adds the complexity of coordinating across clusters.

### The Reconciliation Loop

The controller runs a "reconciliation" function whenever something changes. This function looks at the current state of a session and decides what to do next.

The session goes through several phases. It starts as "Initializing" when just created - this is when we need to create child sessions in all workload clusters and potentially create database branches. It moves to "Pending" once we've started creating child sessions but not all are ready yet. It becomes "Ready" when all child sessions are active. It can become "Failed" if something goes wrong, or "Terminating" when being deleted.

---

## CRD and In-Memory State

We maintain session state in two places: the `MirrordMultiClusterSession` CRD and an in-memory `DashMap`. The CRD is our source of truth - it survives operator restarts and is visible via kubectl. The in-memory map is a performance cache that avoids slow Kubernetes API calls when routing WebSocket messages. Every time the controller reconciles a session, it syncs the in-memory map from the CRD, ensuring they stay consistent. When cleaning up, we always read from the CRD status (not memory) because if the operator restarted, the memory would be empty but the CRD still knows which child sessions exist.

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
    matches!(msg,
        ClientMessage::FileRequest(_)
        | ClientMessage::GetEnvVarsRequest(_)
        | ClientMessage::TcpOutgoing(_)
        | ClientMessage::UdpOutgoing(_)
        | ClientMessage::GetAddrInfoRequest(_)
        | ClientMessage::Ping
    )
}
```

If it's a stateful operation, we send to the Default cluster only. Otherwise, we broadcast to all workload clusters.

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

## Database Branching in Multi-Cluster

### Create Once, Use Everywhere

We create database branches ONLY on the Default cluster. The Primary operator's Envoy handles this during session initialization. Before creating any child sessions, it connects to the Default cluster and creates the necessary `PgBranchDatabase` or `MysqlBranchDatabase` CRDs there.

Once the branches are created, we get back the branch names - for example, `branch_abc123`. These names are stored in the parent `MirrordMultiClusterSession` CRD's spec, in fields like `pg_branch_database_names` and `mysql_branch_database_names`.

### Why Store Branch Names in the Parent?

We store them in the parent because the parent is the authoritative record of the entire multi-cluster session. If we need to know what branches were created, we look at the parent. This also makes cleanup easier - when the parent session is deleted, we know exactly which branches need to be cleaned up.

### Passing Branch Names to the Child Session

When we create the child `MirrordClusterSession` in the Default cluster, we include the branch names in that child's spec. This is crucial.

The child session's operator sees these branch names and stores them in what we call `SessionEphemeralState`. This is per-session data that the agent uses during its lifetime.

When the agent receives a `GetEnvVarsRequest` from the application (asking for environment variables), it doesn't just return the pod's actual environment. It checks the ephemeral state for branch overrides. If there's a database branch configured, it modifies `DATABASE_URL` to point to the branch instead of the original database.

### The Flow in Detail

Let's say the app reads `DATABASE_URL` which normally points to `postgres://db.prod:5432/mydb`.

The app calls `os.Getenv("DATABASE_URL")`. The mirrord layer intercepts this and sends a `GetEnvVarsRequest` to the operator. Because this is a stateful operation, the MultiClusterRouter sends it ONLY to the Default cluster.

The Default cluster's agent receives the request. It reads the pod's actual environment and sees `DATABASE_URL=postgres://db.prod:5432/mydb`. But the agent also has the branch name from SessionEphemeralState. It modifies the response to `DATABASE_URL=postgres://db.prod:5432/branch_abc123`.

The app receives this modified URL and connects to the branch, not production.

---

## The Session Lifecycle

Let's walk through what happens from session creation to cleanup, comparing with single-cluster where relevant.

### CLI Creates a Session

The user runs `mirrord exec`. The CLI sends a request to the Primary operator asking to create a multi-cluster session. The request includes the target alias (like "deployment.myapp") and the mirrord configuration.

In single-cluster, the CLI sends a similar request, and the operator immediately creates a `MirrordClusterSession` and starts spawning an agent.

In multi-cluster, the operator creates a `MirrordMultiClusterSession` CRD with phase "Initializing" and returns the session ID to the CLI. The actual work happens asynchronously.

### Initialization

The controller sees the new CRD and starts the reconciliation process. For an "Initializing" session, it spawns a background task to do the actual work.

First, if the configuration includes database branching, we create the branches on the Default cluster. We connect to that cluster's operator, create the branch CRDs, and wait for them to be ready. The resulting branch names are stored in the parent CRD.

Next, we create child sessions in each workload cluster. For each cluster, we connect to its operator and create a `MirrordClusterSession` CRD. The child session spec includes the target (which pod or deployment), the namespace, and - for the Default cluster only - the database branch names.

We then wait for each child session to become ready, meaning its agent has spawned and is connected.

### Parent Status Update

As each child session becomes ready, we update the parent CRD's status. The status contains a map of all child sessions with their names, readiness, and any errors. Once all children are ready, we set the parent's phase to "Ready".

In single-cluster, there's no parent status to update - the single session's status is enough.

### CLI Connects via WebSocket

Once the session is Ready, the CLI establishes a WebSocket connection to the Primary operator. The operator looks up the session in the in-memory map and creates a `MultiClusterRouter` to handle message routing.

In single-cluster, the operator creates an `AgentRouter` instead. But both return an `AgentConnection` interface, so the code that handles the WebSocket doesn't need to know the difference.

### Active Session

While the session is active, the MultiClusterRouter routes messages to the appropriate clusters. Stateful operations go to the Default cluster. Traffic operations go to all workload clusters.

A background task periodically updates the `connectedTimestamp` in the CRD. This happens every 10 seconds. This timestamp is our "heartbeat" - it tells the controller the session is still in use.

In single-cluster, a similar mechanism exists but uses an in-memory reference count (`UseGuard`). In multi-cluster, we use CRD timestamps because they survive operator restarts and work in HA setups with multiple operator replicas.

### Client Disconnects

When the application exits, the WebSocket closes. The heartbeat task stops updating `connectedTimestamp`.

### TTL Expiration

The controller continues reconciling the session. On each reconciliation, it calculates how long since the last heartbeat. If more than 60 seconds have passed (the default TTL), it deletes the session CRD.

In single-cluster, cleanup happens when the last `UseGuard` reference is dropped. In multi-cluster, we can't rely on in-memory references because of HA and restarts, so we use TTL-based cleanup instead.

### Cleanup via Finalizer

Deleting the CRD triggers the finalizer. Kubernetes finalizers ensure cleanup code runs BEFORE the resource is actually deleted.

The cleanup process reads the child session information from the CRD status - not from memory. This is important because if the operator restarted, the in-memory state would be empty, but the CRD still knows which child sessions exist.

For each child session in the status, we connect to that cluster's operator and delete the child `MirrordClusterSession`. The remote cluster's operator then cleans up its agent.

After all children are deleted, the finalizer is removed from the CRD, and Kubernetes completes the parent deletion.

In single-cluster, cleanup is simpler - just delete the one session and its agent.

---

## Summary

Multi-cluster mirrord adds several new components on top of single-cluster:

The **Envoy** orchestrates multi-cluster sessions, running on the Primary cluster. It coordinates child sessions across multiple workload clusters.

The **MirrordMultiClusterSession** CRD is the parent session that tracks child sessions in each cluster. It stores configuration, branch names, and status.

The **MultiClusterSessionManager** is the controller that manages parent sessions. It handles initialization, creates child sessions, and coordinates cleanup.

The **MultiClusterRouter** routes messages between the client and multiple workload operators. It sends stateful operations to the Default cluster and broadcasts traffic operations to all workloads.

The key insight is that we reuse as much single-cluster code as possible. The MultiClusterRouter presents the same `AgentConnection` interface as the single-cluster AgentRouter. Child `MirrordClusterSession` resources are normal sessions from each cluster's perspective. The complexity of multi-cluster is hidden in the Primary cluster's Envoy, while everything else works like single-cluster.
