- Feature Name: session_monitor
- Start Date: 2026-03-13
- Last Updated: 2026-03-13
- RFC PR: [metalbear-co/rfcs#0000](https://github.com/metalbear-co/rfcs/pull/0000)
- RFC reference:
  - [Notion: mirrord Session Monitor Product Spec](https://www.notion.so/32215e39139481b289cacc87a9cdf257)
  - [metalbear-co/rfcs#8 (Dashboard Backend)](https://github.com/metalbear-co/rfcs/pull/8)

## Summary
[summary]: #summary

Add an HTTP + WebSocket server to the mirrord internal proxy (intproxy) that streams real-time session events to a browser-based Session Monitor UI. Every `mirrord exec` session gets a localhost URL where developers can observe traffic, file operations, DNS queries, environment variables, and more. The intproxy already sees all messages between the layer and agent, making it the natural place to tap into the data stream. The Session Monitor works for all users (OSS and Teams) with no operator dependency.

## Motivation
[motivation]: #motivation

mirrord is a "black box" to its users. When a developer runs `mirrord exec`, they see their application start, but have no visibility into what mirrord is doing behind the scenes. This creates problems:

1. **Debugging is blind** - When something doesn't work (wrong file served, missing env var, traffic not arriving), developers have no way to see what mirrord intercepted, what it forwarded remotely, and what fell back to local. The only option is adding `-v` flags and reading log output.

2. **No session awareness** - Developers don't know if their session is healthy, how much traffic is flowing, or which remote resources they're accessing.

3. **Configuration is guesswork** - Users set up mirrord configs (file filters, port subscriptions, outgoing filters) without feedback on whether the config is doing what they intended.

4. **No growth surface** - mirrord has no UI surface where we can show the value of Teams features to OSS users. The admin dashboard only reaches paying customers who have the operator.

### Use Cases

1. **Developer debugging** - "My app isn't getting the right config. Let me check the Session Monitor to see which files are being read remotely vs locally."

2. **Traffic inspection** - "I'm stealing traffic on port 8080 but nothing's arriving. Let me check if the port subscription is active and if any connections are coming in."

3. **Environment debugging** - "My app is connecting to the wrong database. Let me check which env vars mirrord fetched from the remote pod."

4. **DNS debugging** - "Service discovery isn't working. Let me check what DNS queries mirrord is resolving and what IPs they map to."

5. **Performance investigation** - "My app feels slow with mirrord. Let me check if remote file reads or DNS are adding latency."

6. **Teams discovery (business)** - An OSS developer using the Session Monitor sees "3 other developers are targeting this service" (locked, requires Teams). They click "Start Free Trial" to unlock it.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Basic Usage

When a developer runs mirrord, the Session Monitor URL is printed to stdout:

```
$ mirrord exec --target deploy/checkout -- node server.js
⚡ Session Monitor: http://localhost:48291
⚡ mirrord is running on target deploy/checkout-service-7b9f...
```

Opening that URL in a browser shows a real-time dashboard of the mirrord session:

- **Session header**: target name, mode (steal/mirror), connection state, elapsed time
- **Traffic panel**: TCP connections, bytes in/out, HTTP method breakdown
- **File operations panel**: remote reads, writes, local fallbacks, errors, with file paths
- **DNS panel**: resolved hostnames, query counts, resolved IPs
- **Port subscriptions**: local/remote port mappings, mode per port
- **Environment variables**: which vars were fetched remotely, their keys (values redacted by default)
- **Outgoing connections**: external services the app connects to (host:port, connection count)
- **Event log**: scrollable, filterable stream of everything mirrord is doing

### Multi-Session Support

If a developer runs multiple mirrord sessions (different terminals, different targets), each session gets its own intproxy and its own Session Monitor port. The URLs are independent:

```
Terminal 1: mirrord exec --target deploy/checkout -- node server.js
⚡ Session Monitor: http://localhost:48291

Terminal 2: mirrord exec --target deploy/payments -- python app.py
⚡ Session Monitor: http://localhost:52107
```

If a single session forks child processes (e.g., Node.js worker threads), all child layers connect to the same intproxy and appear as sub-sessions in the same Session Monitor.

### Configuration

The Session Monitor is enabled by default. It can be controlled via the mirrord config:

```json
{
  "session_monitor": {
    "enabled": true,
    "port": 0,
    "open_browser": false
  }
}
```

- `enabled` (default: `true`): Set to `false` to disable the Session Monitor entirely. No HTTP server is started.
- `port` (default: `0`): Port to listen on. `0` means random available port. Set a fixed port for scripting or bookmarking.
- `open_browser` (default: `false`): If `true`, automatically opens the Session Monitor in the default browser on session start.

Environment variable override: `MIRRORD_SESSION_MONITOR=false` disables it (useful in CI).

### Interaction with Existing Features

The Session Monitor is purely observational. It does not modify mirrord's behavior. Disabling it has zero effect on file ops, traffic mirroring/stealing, env fetching, DNS, or any other feature.

The only interaction is with the intproxy's lifecycle:
- The intproxy already runs as a background process (reparented to init)
- The HTTP server binds alongside the existing layer TCP listener
- When all layers disconnect and the idle timeout expires, the HTTP server shuts down with the intproxy

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Architecture Overview

```
                                  ┌──────────────────────────────┐
                                  │        Browser               │
                                  │   Session Monitor UI         │
                                  │   (React static assets)      │
                                  └──────────┬───────────────────┘
                                             │ HTTP GET / (static)
                                             │ WS /ws (events)
                                             │ GET /api/state (snapshot)
                                             ▼
┌──────────────┐    TCP (bincode)    ┌──────────────────────────────┐
│  Layer        │ ◄────────────────► │        Intproxy              │
│  (LD_PRELOAD) │                    │                              │
│              │                    │  ┌────────────┐              │
│  Layer 2     │ ◄──────────────►   │  │ Monitor    │ HTTP+WS     │
│  (forked)    │                    │  │ Server     │ :48291      │
│              │                    │  └────────────┘              │
└──────────────┘                    │                              │
                                    │  ┌────────────┐              │
                                    │  │ FilesProxy │              │
                                    │  │ IncomingPrx│              │
                                    │  │ OutgoingPrx│              │
                                    │  │ SimpleProxy│              │
                                    │  └─────┬──────┘              │
                                    └────────┼─────────────────────┘
                                             │ TCP (bincode)
                                             ▼
                                    ┌──────────────────┐
                                    │     Agent        │
                                    │  (K8s cluster)   │
                                    └──────────────────┘
```

### Component: MonitorServer

A new component added to the intproxy, alongside the existing background tasks (FilesProxy, IncomingProxy, OutgoingProxy, SimpleProxy, PingPong).

**Location:** `mirrord/intproxy/src/monitor.rs` (new file)

**Responsibilities:**
1. Bind an HTTP server on `localhost:<port>`
2. Serve static frontend assets (embedded in binary or from a directory)
3. Accept WebSocket connections at `/ws`
4. Maintain a `MonitorState` struct that aggregates session data
5. Receive `MonitorEvent` messages from the intproxy main loop
6. Broadcast events to all connected WebSocket clients

### MonitorState

```rust
/// Aggregated state of all sessions managed by this intproxy instance.
pub struct MonitorState {
    /// Per-layer session information.
    sessions: HashMap<LayerId, SessionInfo>,
    /// Aggregated traffic statistics.
    traffic: TrafficStats,
    /// File operation statistics.
    file_ops: FileOpsStats,
    /// DNS query log.
    dns: DnsStats,
    /// Port subscriptions across all layers.
    ports: Vec<PortSubscription>,
    /// Environment variables fetched (keys only, values redacted).
    env_vars: Vec<EnvVarInfo>,
    /// Outgoing connection stats.
    outgoing: OutgoingStats,
    /// Rolling event log (bounded ring buffer).
    events: VecDeque<MonitorEvent>,
    /// Session start time.
    started_at: Instant,
    /// Target information from config.
    target: TargetInfo,
    /// mirrord config (sanitized, no secrets).
    config: SanitizedConfig,
}

pub struct SessionInfo {
    layer_id: LayerId,
    process: ProcessInfo,  // pid, name, cmdline
    connected_at: Instant,
    is_active: bool,
}

pub struct TrafficStats {
    tcp_connections: u64,
    bytes_in: u64,
    bytes_out: u64,
    http_requests: HashMap<String, u64>,  // method -> count
    http_status_codes: HashMap<u16, u64>, // status -> count
}

pub struct FileOpsStats {
    remote_reads: u64,
    remote_writes: u64,
    local_fallbacks: u64,
    errors: u64,
    /// Most recently accessed files (bounded).
    recent_files: VecDeque<FileAccessRecord>,
}

pub struct FileAccessRecord {
    path: String,
    operation: FileOperation, // Read, Write, Stat, Unlink, etc.
    remote: bool,
    timestamp: Instant,
    latency: Option<Duration>,
}

pub struct DnsStats {
    total_queries: u64,
    unique_hostnames: HashSet<String>,
    recent_queries: VecDeque<DnsQueryRecord>,
}

pub struct DnsQueryRecord {
    hostname: String,
    resolved: Vec<String>, // IP addresses
    timestamp: Instant,
    latency: Option<Duration>,
}

pub struct OutgoingStats {
    connections: HashMap<String, u64>, // "host:port" -> count
    bytes_sent: u64,
    bytes_received: u64,
}

pub struct EnvVarInfo {
    key: String,
    source: EnvVarSource, // Remote, Override, Excluded
    // Value intentionally omitted for security
}
```

### MonitorEvent

Events emitted by the intproxy main loop and broadcast to WebSocket clients:

```rust
#[derive(Serialize)]
#[serde(tag = "type")]
pub enum MonitorEvent {
    /// A new layer connected.
    SessionConnected {
        layer_id: u64,
        process: ProcessInfoDto,
        timestamp: u64,
    },
    /// A layer disconnected.
    SessionDisconnected {
        layer_id: u64,
        timestamp: u64,
    },
    /// File operation observed.
    FileOp {
        path: String,
        operation: String, // "read", "write", "stat", "unlink", "mkdir", etc.
        remote: bool,
        #[serde(skip_serializing_if = "Option::is_none")]
        latency_ms: Option<u64>,
        timestamp: u64,
    },
    /// DNS query resolved.
    DnsQuery {
        hostname: String,
        resolved: Vec<String>,
        #[serde(skip_serializing_if = "Option::is_none")]
        latency_ms: Option<u64>,
        timestamp: u64,
    },
    /// Incoming TCP connection accepted.
    IncomingConnection {
        port: u16,
        source: String,
        timestamp: u64,
    },
    /// HTTP request intercepted (incoming).
    HttpRequest {
        method: String,
        path: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        status: Option<u16>,
        #[serde(skip_serializing_if = "Option::is_none")]
        latency_ms: Option<u64>,
        timestamp: u64,
    },
    /// Outgoing connection initiated.
    OutgoingConnection {
        remote_address: String,
        port: u16,
        protocol: String, // "tcp" or "udp"
        timestamp: u64,
    },
    /// Port subscription changed.
    PortSubscription {
        port: u16,
        mode: String, // "mirror" or "steal"
        action: String, // "subscribe" or "unsubscribe"
        timestamp: u64,
    },
    /// Environment variable fetched.
    EnvVar {
        key: String,
        source: String, // "remote", "override", "excluded"
        timestamp: u64,
    },
    /// Process forked.
    ProcessForked {
        parent_layer_id: u64,
        child_layer_id: u64,
        timestamp: u64,
    },
    /// Error occurred.
    Error {
        message: String,
        context: String,
        timestamp: u64,
    },
    /// Agent log message.
    AgentLog {
        level: String,
        message: String,
        timestamp: u64,
    },
}
```

### Integration with Intproxy Main Loop

The intproxy's `run_inner()` method (in `mirrord/intproxy/src/lib.rs`) uses a `tokio::select!` loop to poll background tasks. The MonitorServer integrates as follows:

```rust
// In IntProxy::run_inner()

// Start monitor server if enabled
let monitor = if config.session_monitor.enabled {
    let (monitor_tx, monitor_server) = MonitorServer::new(
        config.session_monitor.port,
        config.target.clone(),
        config.sanitized(),
    ).await?;
    // Print URL to stderr (stdout is used for intproxy address)
    eprintln!("⚡ Session Monitor: http://localhost:{}", monitor_server.port());
    Some((monitor_tx, monitor_server))
} else {
    None
};

// In the main select! loop, add a branch for monitor server:
loop {
    tokio::select! {
        // ... existing branches for agent, layer, ping_pong ...

        // Monitor server events (client connect/disconnect, etc.)
        msg = monitor_server.next(), if monitor.is_some() => {
            // Handle monitor client lifecycle
        }
    }
}
```

**Event emission points** - The intproxy emits `MonitorEvent` by calling `monitor_tx.send()` at these locations:

| Location | Event | Trigger |
|----------|-------|---------|
| `IntProxy::handle_layer_message()` | `FileOp` | Layer sends `FileRequest` |
| `IntProxy::handle_layer_message()` | `DnsQuery` | Layer sends `GetAddrInfoRequest` |
| `IntProxy::handle_layer_message()` | `EnvVar` | Layer sends `GetEnvVarsRequest` |
| `IntProxy::handle_layer_message()` | `OutgoingConnection` | Layer sends `OutgoingRequest::Connect` |
| `IntProxy::handle_layer_message()` | `PortSubscription` | Layer sends `IncomingRequest::PortSubscribe` |
| `IntProxy::handle_agent_message()` | `HttpRequest` | Agent sends `TcpSteal::HttpRequest` |
| `IntProxy::handle_agent_message()` | `IncomingConnection` | Agent sends `Tcp::NewConnection` |
| `IntProxy::handle_agent_message()` | `AgentLog` | Agent sends `DaemonMessage::LogMessage` |
| `IntProxy::handle_agent_message()` | `Error` | Agent sends error responses |
| `IntProxy::new_layer_connected()` | `SessionConnected` | Layer sends `NewSession` |
| `IntProxy::layer_disconnected()` | `SessionDisconnected` | Layer TCP connection closes |
| `IntProxy::handle_layer_forked()` | `ProcessForked` | Layer sends `LayerForked` |

**Latency tracking** - For file ops and DNS, the intproxy records the timestamp when the request is sent to the agent, then calculates the delta when the response arrives. This requires a small `PendingRequests` map keyed by `(LayerId, MessageId)`.

### HTTP Server Endpoints

Using `axum` (already used by the license-server, consistent with the codebase):

```
GET /                  -> Serve index.html (static assets)
GET /assets/*          -> Serve static JS/CSS assets
GET /api/state         -> JSON snapshot of current MonitorState
GET /api/config        -> Sanitized mirrord config
WS  /ws                -> WebSocket event stream
```

**WebSocket protocol:**
- On connect: server sends a `MonitorEvent::Snapshot` with the full current `MonitorState`
- After that: server sends individual `MonitorEvent` messages as they occur
- Client can send `{ "type": "ping" }` to keep connection alive
- All messages are JSON-serialized (not bincode, since this is browser-facing)

### Static Asset Embedding

The Session Monitor frontend (React app) is compiled to static assets and embedded in the mirrord CLI binary at build time using `include_dir` or `rust-embed`:

```rust
#[derive(RustEmbed)]
#[folder = "monitor-ui/dist/"]
struct MonitorAssets;
```

**Build integration:**
- The frontend lives in a new directory: `mirrord/monitor-ui/` (React + Vite)
- CI builds the frontend first (`npm run build`), then the Rust binary embeds the output
- Total embedded size: ~200-300KB gzipped (React + a few components, no heavy dependencies)
- If the assets are missing at compile time (e.g., dev build without frontend), the monitor server still starts but returns a plaintext "Session Monitor UI not built" message at `GET /`

### Multi-Session Data Model

The intproxy already tracks multiple layers via `LayerId`. The MonitorState uses the same model:

```
IntProxy instance
  ├── Layer 1 (pid 1234, "node server.js")     ← main process
  ├── Layer 2 (pid 1235, "node worker.js")      ← forked child
  └── Layer 3 (pid 1236, "node worker.js")      ← forked child
```

The Session Monitor UI shows all layers as sub-sessions within a single view. Stats are aggregated across all layers by default, with optional per-layer filtering in the UI.

Each intproxy instance serves one Session Monitor. Multiple `mirrord exec` invocations create separate intproxy instances with separate monitors on different ports.

### Security Considerations

- **Localhost only**: The HTTP server binds to `127.0.0.1`, never `0.0.0.0`. Only local processes can access it.
- **No secrets in events**: Environment variable values are never included in MonitorEvents. Only keys and source (remote/override/excluded) are exposed. File contents are never streamed, only paths and operation types.
- **No control plane**: The Session Monitor is read-only. It cannot modify the session, restart agents, or drop sockets. Control features are reserved for Teams (future, via operator API).
- **No auth**: Same security model as Vite dev server, Storybook, React DevTools. If you can access localhost, you can see the monitor.

### Performance Impact

- **CPU**: The intproxy already processes every message. Emitting MonitorEvents adds a `serde_json::to_string` call per event and a `tokio::sync::broadcast::send`. Estimated overhead: <1% CPU.
- **Memory**: MonitorState maintains bounded collections (ring buffers for recent events/files/DNS, capped at 1000 entries). Estimated overhead: <1MB.
- **Network**: WebSocket traffic stays on localhost. No external network impact.
- **When disabled**: Zero overhead. No HTTP server is started, no event tracking occurs.

The event channel uses `tokio::sync::broadcast` with a bounded capacity (e.g., 4096). If no WebSocket clients are connected, events are dropped (no buffering). If a slow client falls behind, it receives a `Lagged` error and can re-sync via `GET /api/state`.

## Drawbacks
[drawbacks]: #drawbacks

1. **Binary size increase**: Embedding the React frontend adds ~200-300KB to the mirrord CLI binary. This is small relative to the current binary size (~30MB) but non-zero.

2. **Build complexity**: The CI pipeline needs to build the frontend (Node.js) before the Rust binary. This adds a build step and a Node.js dependency to CI.

3. **Port conflicts**: Binding a random localhost port is usually fine, but in constrained environments (containers with limited port ranges, strict firewall rules), it could fail. Mitigation: the feature is disableable.

4. **Maintenance burden**: A new frontend (React app) to maintain, even if small. Design changes, dependency updates, security patches.

5. **Feature creep risk**: Once a UI exists, there will be pressure to add more features to it (config editing, session control, etc.). The RFC scope intentionally limits this to read-only observation.

6. **axum dependency**: Adding axum to the intproxy pulls in a non-trivial dependency tree (hyper, tower, etc.). However, the intproxy already depends on hyper (for HTTP gateway in IncomingProxy), so the incremental cost is mainly axum itself and tokio-tungstenite for WebSocket.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why the intproxy?

The intproxy is the only component that sees ALL messages between every layer and the agent. It already aggregates data from multiple layers (forked processes). Adding the monitor here means:
- Zero changes to the layer (LD_PRELOAD library, hardest to modify)
- Zero changes to the agent (runs in customer clusters, hardest to deploy)
- Zero changes to the protocol (no new message types between layer and agent)

### Why not the layer?

The layer runs inside the user's process via LD_PRELOAD. Adding an HTTP server there would:
- Compete for the process's event loop
- Risk interfering with the application's own port bindings
- Require changes to the delicate hook/detour system
- Not support multi-layer aggregation (each layer only sees its own operations)

### Why not a separate sidecar process?

A separate process would need IPC to receive events from the intproxy. The intproxy already runs as a separate process. Adding the HTTP server directly to it avoids an extra process and IPC complexity.

### Why not the IDE extension?

The VS Code and IntelliJ extensions could host a webview panel. However:
- Not all users use an IDE (CLI users, CI pipelines)
- Requires separate implementations for VS Code and IntelliJ
- The IDE extension can open the Session Monitor URL in a webview as a future enhancement (Phase 4)

### Why not a TUI (terminal UI)?

A terminal UI (like k9s) would conflict with the user's own terminal output (their application's stdout/stderr). A browser-based UI runs in a separate window and doesn't interfere.

### Impact of not doing this

mirrord remains a black box. Developers continue to rely on verbose logging for debugging. We have no UI surface to reach OSS users for Teams upsell. The admin dashboard continues to only serve paying customers.

## Prior art
[prior-art]: #prior-art

### Tilt (localhost:10350)

Tilt starts a local web UI at `localhost:10350` showing resource status, build logs, and health checks. It's the closest model to what we're proposing.

**What works well:** Always available, zero setup, auto-opens browser (configurable).
**What we'd do differently:** Tilt focuses on build/deploy status. We focus on session-level detail (traffic, files, DNS). Tilt also doesn't have an upsell surface since it's fully open source.

### Vite Dev Server

Vite's dev server overlay shows build errors and HMR status directly in the browser. It demonstrates that embedding a small UI server in a dev tool is well-accepted.

### React DevTools

Chrome extension with a panel showing component tree, state, and performance. Shows that developers are willing to use a separate UI for debugging their runtime.

### Telepresence Dashboard

Telepresence (now owned by Gravitee) has a cloud-hosted dashboard for intercept management. Requires account signup and internet connectivity. Our approach is local-first with no cloud dependency.

### Grafana Agent / Prometheus

These tools run local HTTP servers for health checks and metrics scraping (`/metrics` endpoint). Shows that running an HTTP server alongside a monitoring agent is standard practice.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Event granularity**: Should every individual `read()` syscall be a MonitorEvent, or should we batch/aggregate? High-throughput applications could generate thousands of file reads per second. Proposal: aggregate at the intproxy level (increment counters), emit individual events only for "interesting" operations (first access to a new file, errors, DNS, connections).

2. **Value redaction policy**: Should env var values ever be shown? Some are harmless (e.g., `NODE_ENV=production`), others are secrets (e.g., `DATABASE_URL`). Proposal: never show values by default, add opt-in config `session_monitor.show_env_values = true`.

3. **Frontend repo location**: Options are (a) `mirrord/monitor-ui/` in the main mirrord repo, (b) separate `mirrord-monitor` repo, (c) in the existing `operator/dashboard/` directory. Recommendation: (a) in the main repo, since it ships with the CLI binary and needs to be embedded at build time.

4. **axum vs alternatives**: The license-server uses axum. The intproxy doesn't currently have an HTTP framework. Should we use axum for consistency, or a lighter-weight alternative like `warp` or raw `hyper`? The intproxy already depends on hyper.

5. **CI environments**: Should the Session Monitor be disabled by default in CI? Proposal: auto-detect CI via standard env vars (`CI=true`, `GITHUB_ACTIONS`, `JENKINS_URL`, etc.) and disable unless explicitly enabled.

6. **Existing `--json` flag interaction**: The mirrord CLI has a `--json` flag for machine-readable output. How does the Session Monitor URL get communicated in JSON mode? Proposal: include it in the JSON output as `"session_monitor_url": "http://localhost:48291"`.

## Future possibilities
[future-possibilities]: #future-possibilities

1. **Teams features in Session Monitor**: When the operator is available, the Session Monitor can fetch additional data (remote sessions, session history, target topology) and display them alongside the local session data. Locked features for non-Teams users become the upsell surface.

2. **Session control** (Teams): Stop session, restart agent, drop sockets, pause stealing. Requires new operator API endpoints and authentication.

3. **IDE integration**: VS Code command "mirrord: Open Session Monitor" that opens the URL in a webview panel. IntelliJ tool window integration. Both just open the existing localhost URL.

4. **Config suggestions**: Based on observed patterns (frequently accessed remote files, unused port subscriptions), suggest config improvements. "Add `/app/config.yaml` to your local filter to save 200ms per request."

5. **Session recording**: Save the event stream to a file for post-mortem debugging or sharing with teammates. `mirrord exec --record session.json`.

6. **Metrics export**: Expose a `/metrics` Prometheus endpoint for integration with existing monitoring stacks.

7. **Multi-session aggregator**: A separate page/tool that discovers all running intproxy instances on localhost and shows a unified view of all active mirrord sessions.

8. **Analytics**: With user consent, send anonymized usage telemetry (features used, session duration, error rates) to help prioritize development. Fully opt-in.
