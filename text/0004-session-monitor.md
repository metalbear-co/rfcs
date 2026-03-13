- Feature Name: session_monitor
- Start Date: 2026-03-13
- Last Updated: 2026-03-13
- RFC PR: [metalbear-co/rfcs#20](https://github.com/metalbear-co/rfcs/pull/20)
- RFC reference:
  - [Notion: mirrord Session Monitor Product Spec](https://www.notion.so/32215e39139481b289cacc87a9cdf257)
  - [metalbear-co/rfcs#8 (Dashboard Backend)](https://github.com/metalbear-co/rfcs/pull/8)

## Summary
[summary]: #summary

Add a local session monitoring system to mirrord with two components: (1) each intproxy exposes a Unix socket at `~/.mirrord/sessions/` that streams real-time session events, and (2) a new `mirrord ui` command that discovers all active sessions via that directory, aggregates them into a single web UI served over localhost with TLS and CSRF protection. Additionally, new CLI commands (`mirrord status`, `mirrord kill`) provide non-interactive access to the same data for AI agents and scripting. The system works for all users (OSS and Teams) with no operator dependency.

## Motivation
[motivation]: #motivation

mirrord is a "black box" to its users. When a developer runs `mirrord exec`, they see their application start, but have no visibility into what mirrord is doing behind the scenes. This creates problems:

1. **Debugging is blind** - When something doesn't work (wrong file served, missing env var, traffic not arriving), developers have no way to see what mirrord intercepted, what it forwarded remotely, and what fell back to local. The only option is adding `-v` flags and reading log output.

2. **No session awareness** - Developers don't know if their session is healthy, how much traffic is flowing, or which remote resources they're accessing.

3. **Configuration is guesswork** - Users set up mirrord configs (file filters, port subscriptions, outgoing filters) without feedback on whether the config is doing what they intended.

4. **No growth surface** - mirrord has no UI surface where we can show the value of Teams features to OSS users. The admin dashboard only reaches paying customers who have the operator.

5. **No programmatic access** - AI coding agents and scripts cannot query mirrord session state. There's no `mirrord status` or way to kill a session by ID.

### Use Cases

1. **Developer debugging** - "My app isn't getting the right config. Let me open `mirrord ui` to see which files are being read remotely vs locally."

2. **Multi-session overview** - "I have 3 mirrord sessions running in different terminals. Let me see them all in one place."

3. **Traffic inspection** - "I'm stealing traffic on port 8080 but nothing's arriving. Let me check if the port subscription is active."

4. **Environment debugging** - "My app is connecting to the wrong database. Let me check which env vars mirrord fetched."

5. **Session management** - "I forgot to close a mirrord session in another terminal. Let me kill it from the UI or via `mirrord kill`."

6. **AI agent integration** - An AI coding agent runs `mirrord status --json` to check session health, or `mirrord kill <id>` to clean up after a test run.

7. **Teams discovery (business)** - An OSS developer using `mirrord ui` sees "3 other developers are targeting this service" (locked, requires Teams). They click "Start Free Trial."

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Starting the UI

The Session Monitor is a separate command, not auto-launched:

```
$ mirrord ui
🔒 Session Monitor: https://localhost:59281?token=a8f3b2c1...
   Opening browser...
```

This starts a local web server that discovers and connects to all active mirrord sessions on the machine. The URL includes a one-time token for CSRF protection.

### Viewing Sessions

The UI shows all active sessions in one view:

```
┌──────────────────────────────────────────────────────────────────┐
│  mirrord Session Monitor          v3.165.0 (OSS)   2 sessions   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SESSION 1: deploy/checkout-service (steal)          ● Running   │
│  PID 1234 · node server.js · 12m 34s · ~/projects/checkout      │
│  Ports: :80→:3000 (steal), :8080→:8080 (steal)                  │
│  Traffic: 147 connections, 2.3 MB in/out                         │
│  Files: 23 remote reads, 5 writes, 2 local fallbacks            │
│  DNS: 12 queries (8 unique hosts)                                │
│  [View Details]  [Kill Session]                                  │
│                                                                  │
│  SESSION 2: deploy/payments-service (mirror)         ● Running   │
│  PID 5678 · python app.py · 3m 12s · ~/projects/payments        │
│  Ports: :5000→:5000 (mirror)                                     │
│  Traffic: 42 connections, 890 KB in/out                          │
│  [View Details]  [Kill Session]                                  │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│  📰 What's New                                                   │
│  • v3.165.0: HTTP filter improvements for steal mode             │
│  • v3.164.0: Go 1.23 support                                    │
│  [Full Changelog →]                                              │
│                                                                  │
│  ⭐ mirrord for Teams                                            │
│  See who's targeting your services · Session history · Control   │
│  [Start Free Trial →]                                            │
└──────────────────────────────────────────────────────────────────┘
```

Clicking "View Details" on a session expands it to show:
- Real-time event log (file ops, DNS, network, errors)
- Traffic stats breakdown (HTTP methods, status codes, latency)
- File operations detail (paths, read/write, remote vs local)
- DNS query log (hostname, resolved IPs, latency)
- Port subscriptions detail
- Environment variables fetched (keys only, values redacted by default)
- Outgoing connections (which external services the app talks to)
- mirrord config for this session

### CLI Commands

The same session data is available via CLI for scripting and AI agents:

```bash
# List all active local sessions
$ mirrord status
SESSION ID                            TARGET                     MODE    PID    ELAPSED
a8f3b2c1-4d5e-6f7a-8b9c-0d1e2f3a4b5c deploy/checkout-service    steal   1234   12m 34s
c2d3e4f5-6a7b-8c9d-0e1f-2a3b4c5d6e7f deploy/payments-service    mirror  5678   3m 12s

# JSON output for programmatic use
$ mirrord status --json
[{"session_id":"a8f3b2c1...","target":"deploy/checkout-service","mode":"steal",...}]

# Kill a session by ID
$ mirrord kill a8f3b2c1
Session a8f3b2c1 killed.

# Kill all local sessions
$ mirrord kill --all
Killed 2 sessions.
```

If the operator is available, `mirrord status` also shows cluster-wide sessions:

```bash
$ mirrord status --cluster
LOCAL SESSIONS:
  a8f3b2c1  deploy/checkout-service  steal   1234  12m 34s

CLUSTER SESSIONS (requires mirrord for Teams):
  🔒 3 other sessions active on this cluster
  [Upgrade to see details: https://app.metalbear.com/...]
```

### Configuration

```json
{
  "session_monitor": {
    "enabled": true
  }
}
```

- `enabled` (default: `true`): Set to `false` to disable the Unix socket for this session. The intproxy will not create a socket file.

Environment variable override: `MIRRORD_SESSION_MONITOR=false` disables the socket (useful in CI).

The `mirrord ui` command has its own flags:

```bash
mirrord ui                        # Start UI, auto-open browser
mirrord ui --no-open              # Start UI, don't open browser
mirrord ui --port 8080            # Use specific port
mirrord ui --sessions-dir /path   # Custom sessions directory
```

### Interaction with Existing Features

The session monitoring system is purely observational for v1. It does not modify mirrord's behavior, except for the `mirrord kill` command which terminates a session's intproxy process and its associated layers.

The intproxy lifecycle is extended slightly:
- On startup, the intproxy creates a Unix socket at `~/.mirrord/sessions/<session-id>.sock`
- On shutdown, it removes the socket file
- If the intproxy crashes, stale socket files are cleaned up by `mirrord ui` or `mirrord status` (they detect dead sockets and remove them)

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Architecture Overview

The system has two layers: session sockets (per-intproxy) and the UI server (standalone).

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ mirrord exec │   │ mirrord exec │   │ mirrord exec │
│  session 1   │   │  session 2   │   │  session 3   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Intproxy 1  │   │  Intproxy 2  │   │  Intproxy 3  │
│              │   │              │   │              │
│ Unix Socket  │   │ Unix Socket  │   │ Unix Socket  │
│ ~/.mirrord/  │   │ ~/.mirrord/  │   │ ~/.mirrord/  │
│ sessions/    │   │ sessions/    │   │ sessions/    │
│ <id1>.sock   │   │ <id2>.sock   │   │ <id3>.sock   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       └──────────┬───────┴──────────────────┘
                  │  (Unix socket connections)
                  ▼
         ┌────────────────┐
         │  mirrord ui    │
         │                │
         │  Discovers all │
         │  sockets in    │
         │  ~/.mirrord/   │     ┌─────────────┐
         │  sessions/     │────▶│   Browser    │
         │                │     │ https://     │
         │  HTTP+WS+TLS   │     │ localhost:   │
         │  :59281        │     │ 59281?token= │
         └────────────────┘     └─────────────┘

         ┌────────────────┐
         │ mirrord status │  (reads same sockets)
         │ mirrord kill   │  (sends kill signal)
         └────────────────┘
```

### Component 1: Session Socket (in intproxy)

Each intproxy instance creates a Unix domain socket at `~/.mirrord/sessions/<session-id>.sock` on startup.

**Session ID**: A UUID generated by the intproxy at startup (e.g., `a8f3b2c1-4d5e-6f7a-8b9c-0d1e2f3a4b5c`).

**Socket directory**: `~/.mirrord/sessions/` is created with `0700` permissions (user-only access), following the Docker socket model. This ensures only the current user can access their own sessions.

**Metadata file**: Alongside each socket, a JSON metadata file is written:

```
~/.mirrord/sessions/a8f3b2c1.sock      # Unix socket
~/.mirrord/sessions/a8f3b2c1.json      # Metadata
```

Metadata file contents:

```json
{
  "session_id": "a8f3b2c1-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "target": "deploy/checkout-service",
  "mode": "steal",
  "pid": 1234,
  "process_name": "node",
  "process_args": ["server.js"],
  "working_dir": "/Users/han/projects/checkout",
  "started_at": "2026-03-13T14:30:00Z",
  "mirrord_version": "3.165.0",
  "is_operator": false,
  "config_path": "/Users/han/projects/checkout/.mirrord/mirrord.json"
}
```

**Socket protocol**: The Unix socket speaks a simple JSON-lines protocol:

```
Client connects to socket
  → Server sends: {"type":"hello","session_id":"a8f3b2c1...","state":{...full MonitorState...}}
  → Server streams: {"type":"event","data":{...MonitorEvent...}}\n
  → Server streams: {"type":"event","data":{...MonitorEvent...}}\n
  ...

Client can send commands:
  → {"type":"kill"}\n
  → Server sends: {"type":"ack","command":"kill"}\n
  → Intproxy initiates graceful shutdown
```

**MonitorState** (sent on connect):

```rust
pub struct MonitorState {
    session_id: String,
    target: TargetInfo,
    mode: String,
    started_at: String,         // ISO 8601
    mirrord_version: String,
    is_operator: bool,
    layers: Vec<LayerInfo>,     // connected processes
    traffic: TrafficStats,
    file_ops: FileOpsStats,
    dns: DnsStats,
    ports: Vec<PortSubscription>,
    env_vars: Vec<EnvVarInfo>,
    outgoing: OutgoingStats,
    recent_events: Vec<MonitorEvent>,  // last N events
}
```

**MonitorEvent** (streamed after hello):

```rust
#[derive(Serialize)]
#[serde(tag = "type")]
pub enum MonitorEvent {
    LayerConnected {
        layer_id: u64,
        pid: u32,
        process_name: String,
        timestamp: u64,
    },
    LayerDisconnected {
        layer_id: u64,
        timestamp: u64,
    },
    FileOp {
        path: String,
        operation: String,  // "read", "write", "stat", "unlink", "mkdir", etc.
        remote: bool,
        #[serde(skip_serializing_if = "Option::is_none")]
        latency_ms: Option<u64>,
        timestamp: u64,
    },
    DnsQuery {
        hostname: String,
        resolved: Vec<String>,
        #[serde(skip_serializing_if = "Option::is_none")]
        latency_ms: Option<u64>,
        timestamp: u64,
    },
    IncomingConnection {
        port: u16,
        source: String,
        timestamp: u64,
    },
    HttpRequest {
        method: String,
        path: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        status: Option<u16>,
        #[serde(skip_serializing_if = "Option::is_none")]
        latency_ms: Option<u64>,
        timestamp: u64,
    },
    OutgoingConnection {
        remote_address: String,
        port: u16,
        protocol: String,
        timestamp: u64,
    },
    PortSubscription {
        port: u16,
        mode: String,
        action: String,
        timestamp: u64,
    },
    EnvVar {
        key: String,
        source: String,
        timestamp: u64,
    },
    ProcessForked {
        parent_layer_id: u64,
        child_layer_id: u64,
        timestamp: u64,
    },
    Error {
        message: String,
        context: String,
        timestamp: u64,
    },
    AgentLog {
        level: String,
        message: String,
        timestamp: u64,
    },
}
```

**Event emission points in the intproxy:**

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
| `IntProxy::new_layer_connected()` | `LayerConnected` | Layer sends `NewSession` |
| `IntProxy::layer_disconnected()` | `LayerDisconnected` | Layer TCP connection closes |
| `IntProxy::handle_layer_forked()` | `ProcessForked` | Layer sends `LayerForked` |

**Latency tracking**: The intproxy records timestamps when requests are sent to the agent and computes the delta when responses arrive. A `HashMap<(LayerId, MessageId), Instant>` tracks pending requests.

**Cleanup**: On intproxy exit (normal or crash), the socket and metadata files should be removed. The intproxy registers a shutdown hook to delete both files. For crash scenarios, the `mirrord ui` and `mirrord status` commands detect stale sockets (connection refused) and clean them up.

### Component 2: `mirrord ui` Command

A new subcommand that runs a web server aggregating all local sessions.

**Startup sequence:**

1. Scan `~/.mirrord/sessions/` for `*.json` metadata files
2. For each, attempt to connect to the corresponding `*.sock`
3. Clean up stale entries (socket exists but connection refused)
4. Generate a TLS self-signed certificate (in-memory, ephemeral)
5. Generate a random access token
6. Start HTTPS server on `127.0.0.1:<port>`
7. Print URL with token: `https://localhost:59281?token=<token>`
8. Open browser (unless `--no-open`)
9. Watch `~/.mirrord/sessions/` directory for new/removed sockets (via `notify` crate or polling)
10. For each connected session socket, maintain a tokio task that reads events and forwards to WebSocket clients

**Security model (following Aviram's requirements):**

- **Unix socket permissions**: `~/.mirrord/sessions/` has `0700`. Only the user can access their sessions. Same model as Docker's `/var/run/docker.sock`.
- **TLS**: Self-signed certificate generated at startup. Prevents network sniffing on localhost (relevant on shared machines).
- **CSRF token**: The URL includes `?token=<random>`. The server sets this as a cookie on first access. Subsequent requests must include the cookie. This prevents a malicious webpage from making requests to the localhost server (since it won't have the cookie).
- **Localhost binding**: `127.0.0.1` only, never `0.0.0.0`.

**HTTP endpoints:**

```
GET  /                     → Serve React frontend (index.html)
GET  /assets/*             → Static JS/CSS assets
GET  /api/sessions         → List all active sessions (JSON)
GET  /api/sessions/:id     → Session detail + current state (JSON)
POST /api/sessions/:id/kill → Kill session (terminates intproxy + layers)
GET  /api/version          → mirrord version, OSS/Teams status
GET  /api/changelog        → Latest changelog entries (fetched from GitHub releases API, cached)
WS   /ws                   → WebSocket: aggregated events from all sessions
WS   /ws/:id               → WebSocket: events from a specific session
```

**WebSocket protocol (browser-facing):**

```
Client connects to /ws
  → Server sends: {"type":"sessions","data":[...list of all sessions with state...]}
  → Server streams: {"type":"event","session_id":"a8f3b2c1","data":{...MonitorEvent...}}\n
  → Server streams: {"type":"session_added","data":{...session metadata...}}\n
  → Server streams: {"type":"session_removed","session_id":"a8f3b2c1"}\n
  ...
```

**Static asset embedding**: The React frontend is embedded in the mirrord CLI binary using `rust-embed`. The `mirrord ui` command serves these assets. If running a development build without embedded assets, it can optionally proxy to a Vite dev server (configurable via `--dev` flag).

### Component 3: CLI Commands

**`mirrord status`**:

```rust
// Pseudocode
fn mirrord_status(json: bool, cluster: bool) {
    let sessions_dir = home_dir().join(".mirrord/sessions");
    let sessions = discover_sessions(&sessions_dir);

    if json {
        println!("{}", serde_json::to_string(&sessions)?);
    } else {
        print_table(&sessions);
    }

    if cluster {
        // If operator is available, also query cluster sessions
        match operator_client.get_active_sessions() {
            Ok(cluster_sessions) => print_cluster_sessions(&cluster_sessions),
            Err(_) => println!("🔒 Cluster sessions require mirrord for Teams"),
        }
    }
}
```

**`mirrord kill`**:

```rust
fn mirrord_kill(session_id: Option<String>, all: bool) {
    let sessions_dir = home_dir().join(".mirrord/sessions");

    if all {
        for session in discover_sessions(&sessions_dir) {
            kill_session(&session);
        }
    } else if let Some(id) = session_id {
        let session = find_session(&sessions_dir, &id);
        kill_session(&session);
    }
}

fn kill_session(session: &SessionInfo) {
    // Connect to Unix socket and send kill command
    let mut stream = UnixStream::connect(&session.socket_path)?;
    write!(stream, r#"{{"type":"kill"}}"#)?;
    // Wait for ack
    // The intproxy will gracefully shut down, terminating all layers
}
```

### Multi-Session Data Flow

```
Session 1 intproxy ──Unix Socket──┐
                                  │
Session 2 intproxy ──Unix Socket──┼──► mirrord ui ──HTTPS+WS──► Browser
                                  │
Session 3 intproxy ──Unix Socket──┘

mirrord status ────Unix Socket────► Session N intproxy (read-only query)
mirrord kill ──────Unix Socket────► Session N intproxy (kill command)
```

The `mirrord ui` process maintains a `HashMap<SessionId, SessionConnection>` where each `SessionConnection` is a tokio task reading from the Unix socket and forwarding events to the WebSocket broadcast channel.

When a new socket appears in `~/.mirrord/sessions/`, `mirrord ui` connects to it automatically. When a socket disappears (session ended), the corresponding task is cleaned up and a `session_removed` event is sent to WebSocket clients.

### Performance Impact

**On the intproxy (per session):**
- CPU: Serializing MonitorEvents to JSON adds ~1% overhead (serde_json per event)
- Memory: MonitorState bounded at ~1MB (ring buffers for recent events)
- I/O: Unix socket writes are negligible (local IPC)
- When no client is connected: events are dropped (broadcast channel with no receivers)

**On `mirrord ui`:**
- Aggregates events from all sessions. Memory scales linearly with number of active sessions.
- WebSocket broadcasts to browser clients add minimal overhead.

**When disabled (`session_monitor.enabled = false`):**
- Zero overhead. No socket file created, no event tracking in intproxy.

### Changelog / What's New Feature

The `mirrord ui` server includes a `/api/changelog` endpoint that:

1. Fetches the latest releases from `https://api.github.com/repos/metalbear-co/mirrord/releases` (public API, no auth needed)
2. Caches the response for 1 hour
3. Returns the latest 5 releases with title, date, and body (markdown)

The UI shows a "What's New" section with the latest release notes, linking to the full changelog. This keeps users informed and engaged.

### Version and License Display

The UI header shows:
- mirrord version (compiled into the binary)
- OSS vs Teams status (detected by checking if operator is configured/reachable)
- If Teams: subscription tier and license info (fetched from operator)
- If OSS: "Upgrade to Teams" CTA

## Drawbacks
[drawbacks]: #drawbacks

1. **Two-process model**: `mirrord ui` is a separate process from the intproxy. Users must explicitly run it. This is intentional (following Aviram's architecture), but some users may expect auto-launch.

2. **Unix socket limitation**: Unix sockets don't work on Windows. For Windows support (future), we'd need named pipes or TCP with authentication. macOS and Linux are the primary targets.

3. **Self-signed TLS**: Browsers show a certificate warning on first access. Users must click through "Advanced > Proceed" once. This is the trade-off for localhost TLS without a CA.

4. **Binary size increase**: Embedding the React frontend adds ~200-300KB. Small relative to the ~30MB mirrord binary.

5. **Build complexity**: CI needs Node.js to build the frontend before embedding in the Rust binary.

6. **Stale socket cleanup**: If an intproxy crashes without cleaning up its socket, stale files remain. Mitigated by detection and cleanup in `mirrord status` and `mirrord ui`.

7. **Security surface**: Even with TLS + CSRF + Unix permissions, a local attacker with the same UID could connect to the Unix sockets. This is the same threat model as Docker.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why Unix sockets + separate UI server (vs embedded HTTP per intproxy)?

An earlier draft of this RFC proposed embedding an HTTP server directly in each intproxy. The Unix socket + `mirrord ui` architecture is better because:

- **Multi-session aggregation**: One UI shows all sessions. No need to remember different ports for different sessions.
- **Security**: Unix socket permissions (0700) provide OS-level access control. The `mirrord ui` server adds TLS + CSRF on top. An embedded HTTP server per intproxy would need its own auth.
- **Separation of concerns**: The intproxy stays focused on proxying. The UI server is a separate concern.
- **CLI integration**: `mirrord status` and `mirrord kill` use the same Unix sockets, no HTTP needed for CLI commands.
- **Resource efficiency**: Only one HTTP server running (the `mirrord ui` process), not one per session.

### Why not the layer?

The layer runs inside the user's process via LD_PRELOAD. Adding any IPC there would:
- Compete for the process's event loop
- Risk interfering with the application's own socket usage
- Not support multi-layer aggregation

### Why not a TUI?

A terminal UI (like k9s) is hostile to AI coding agents, which work best with structured CLI output. The browser UI is for humans, the CLI commands (`--json`) are for agents. A TUI falls between these and serves neither well.

However, CLI commands like `mirrord status` and `mirrord kill` provide all the functionality an AI agent needs without a TUI.

### Why TLS + CSRF + token?

Localhost isn't automatically safe. A malicious website could make requests to `http://localhost:59281` via JavaScript (CSRF attack). The token cookie prevents this. TLS prevents local network sniffing on shared machines. This follows the same security model as Jupyter Notebook's localhost server.

### Impact of not doing this

mirrord remains a black box. No growth surface for OSS-to-Teams conversion. AI agents can't interact with mirrord sessions programmatically.

## Prior art
[prior-art]: #prior-art

### Docker CLI + Docker Desktop

Docker uses a Unix socket (`/var/run/docker.sock`) for the Docker daemon API. Docker Desktop provides a GUI that connects to the same socket. `docker ps`, `docker kill` are CLI equivalents. Our architecture mirrors this pattern exactly: Unix sockets for IPC, separate UI process, CLI commands for scripting.

### Tilt (localhost:10350)

Tilt embeds an HTTP server directly (the approach we considered but rejected). Works well for single-resource views. Less ideal for multi-session aggregation.

### Jupyter Notebook

Jupyter runs an HTTPS localhost server with token-based authentication. Users access it via `https://localhost:8888?token=...`. This is the exact security model we're adopting for `mirrord ui`.

### Telepresence Dashboard

Cloud-hosted dashboard requiring account signup. Our approach is local-first with no cloud dependency, which is better for developer trust and offline use.

### Grafana Agent

Runs a local HTTP server for metrics scraping. Shows that local HTTP servers for monitoring are standard practice in the dev tools ecosystem.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Event granularity**: Should every individual `read()` syscall generate a MonitorEvent, or should we batch/aggregate? High-throughput applications could generate thousands of file reads per second. Proposal: aggregate counters in the intproxy, emit individual events only for "interesting" operations (first access to a new file path, errors, DNS queries, new connections).

2. **Env var value redaction**: Should env var values ever be shown? Proposal: never by default, with opt-in via `mirrord ui --show-env-values` for debugging.

3. **Frontend location in repo**: Options: (a) `mirrord/monitor-ui/` in the main mirrord repo (recommended, since it ships with the CLI binary), (b) separate `mirrord-monitor` repo.

4. **Self-signed cert UX**: The browser TLS warning is friction. Alternatives: (a) accept the warning (Jupyter does this), (b) use HTTP with just CSRF token (simpler, less secure), (c) generate and install a local CA cert on first run (more complex, better UX). Recommendation: start with (a), consider (c) later.

5. **Socket protocol versioning**: How to handle protocol changes between mirrord versions? If a user runs `mirrord ui` v3.166 but has an active session from `mirrord exec` v3.165, the socket protocol must be compatible. Proposal: include a `protocol_version` field in the hello message and handle gracefully.

6. **Windows support**: Unix sockets are not available on Windows. Defer Windows support or use named pipes?

7. **Session directory on shared machines**: If multiple users share a machine, `~/.mirrord/sessions/` is per-user (different home dirs). Is there a use case for a system-wide sessions directory?

## Future possibilities
[future-possibilities]: #future-possibilities

1. **Teams features in UI**: When the operator is available, the UI can show remote sessions (who else is targeting this service), session history, and target topology. Locked for OSS users as upsell surface.

2. **Session control actions** (Teams): Via the operator API, the UI could offer actions like restart agent, drop sockets, and pause stealing. These would be additional commands sent over the Unix socket and forwarded to the operator.

3. **IDE integration**: VS Code command "mirrord: Open Session Monitor" that either opens the `mirrord ui` URL in a webview or starts `mirrord ui` if not running. IntelliJ equivalent.

4. **Config suggestions**: Based on observed patterns (files read remotely that could be local, ports not receiving traffic), suggest mirrord config improvements directly in the UI.

5. **Session recording**: `mirrord exec --record session.json` saves the event stream to a file. `mirrord ui --replay session.json` replays it in the UI for post-mortem debugging.

6. **MCP server**: Expose the session data as an MCP (Model Context Protocol) server that AI coding agents can connect to directly, rather than parsing CLI output.

7. **Metrics export**: `/metrics` Prometheus endpoint on the `mirrord ui` server for integration with existing monitoring stacks.

8. **Multi-machine aggregation**: A future `mirrord ui --remote` mode that connects to session sockets on remote machines (via SSH tunneling), enabling team-wide session visibility without the operator.
