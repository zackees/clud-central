# CLUD-CENTRAL — High-Level Design

**Python backend + npm frontend**

## 1) Purpose

Provide a central control plane for many clud agents:

- **Observe**: live agent status, logs, and health.
- **Guide**: trigger high-level intents (start/stop/fix/launch).
- **Develop**: optionally broker access to an agent's VS Code Server (heavy, on-demand).
- **Chat-ops**: Telegram bot for quick bind/status/tail/commands.
- **Progressive unlock**: the first `clud --track` on any host brings CLUD-CENTRAL online for that user.

## 2) Core UX Principles

- **Claude-parity by default**: if you never pass `--track`, nothing changes.
- **Instant unlock**: `clud --track` (client command) auto-spawns a persistent sentinel process on 127.0.0.1:7565 (if not already running) and registers the agent with CLUD-CENTRAL; the web app and bot become available.
- **Tiles first**: primary UI is a grid of terminals (xterm.js), one tile per agent; optional second tile per agent for a cwd shell.
- **VS Code is a power-tool**: on-demand; expect ~20s cold start; clear progress & kill.

## 3) Canonical Runtime Decisions

- **Local sentinel bind**: 127.0.0.1:7565 (always; single persistent sentinel per host; multiplexes all tracked agents).
- **Cross-machine comms**: sentinel → outbound to CLUD-CENTRAL; no inbound to developer hosts.
- **One controlling chat per agent**: Telegram chat ↔ active agent binding (read-only observers optional, later).

### The Sentinel Process

The **sentinel** is a persistent, always-on local daemon that acts as the intermediary between tracked `clud` agents and CLUD-CENTRAL:

- **Auto-spawned**: When you run `clud --track` for the first time, it spawns the sentinel (if not already running).
- **Persistent**: Stays running across multiple `clud --track` invocations; manages the lifecycle of all tracked agents on the host.
- **Multiplexer**: Owns and multiplexes PTYs for all tracked agents and optional cwd shells.
- **Outbound-only**: Maintains a persistent WebSocket/gRPC connection to CLUD-CENTRAL; no inbound ports.
- **Single per host**: Only one sentinel runs per developer machine, regardless of how many agents are tracked.

## 4) Architecture (conceptual)

```
[Operator]──Telegram──┐
                      │
                  [CLUD-CENTRAL]
                 /      |       \
       Web UI (tiles)  Registrar  Tools Mediator (VS Code broker)
               |           |                 |
               |      Outbound WS            |
               v           |                 v
            (browser) <── [Sentinel] ──> [clud --track instances]
                  xterm.js  127.0.0.1:7565   per-agent PTY + task
                            (persistent,
                             multiplexes PTYs)
```

**Flow:**
1. User runs `clud --track <command>` (the client)
2. Client probes 127.0.0.1:7565 → auto-spawns sentinel if absent
3. Sentinel registers the agent, spawns its PTY, and streams to CLUD-CENTRAL
4. CLUD-CENTRAL fans out to Web UI tiles and Telegram bot

## 5) Components & Responsibilities

### A. Web App (Frontend; npm)

- **Stack**: React + xterm.js + WebSocket client; CSS Grid (or react-grid-layout) for tiling.
- **Tiles**: one xterm per agent (stdout/stderr); optional "Open Shell" tile bound to agent's cwd.
- **State**: minimal—subscribe to streams; persist layout locally per operator (and server-side as a small JSON doc for restore).
- **Build**: `packages/web/` built by npm; artifacts (JS/CSS) embedded into the Python package (wheel) at `clud_central/static/`.

### B. Control Plane (Backend; Python)

- **Framework**: FastAPI/Starlette (async) or Flask-SocketIO (eventlet/gevent). Prefer FastAPI + uvicorn for asyncio parity.
- **Registrar**: authoritative list of agents (id/host/pid/cwd/caps/status/heartbeats).
- **Session & Auth**: operator identity (Telegram), scoped sessions for terminal attach & VS Code actions.
- **Events bus (in-proc)**: fan-out agent events/telemetry to Web UI sessions and Telegram bridge.
- **Storage**: SQLite for dev; Postgres optional later (agents, bindings, events); no heavy schema design yet.
- **Packaging**: pyproject.toml with extras `web` (serves static) and `bot` (Telegram).

### C. Telegram Bridge (Python)

- **Bind & drive**: `/bind`, `/status`, `/tail <N|time>`, `/run <allowlisted>`, `/kill`, `/launch <path>`, `/vscode`.
- **Policy**: allow-listed shell actions; cwd jail; output caps with "send file" fallback.
- **Tone**: concise summaries; buttons for next actions (extend tail, open code, stop).

### D. VS Code Mediator

- **Launch policy**: on-demand; single instance per agent; idle timeout; clear kill.
- **Routing**: central issues short-lived session token and coordinates access (implementation left abstract here).
- **Visibility**: show "warming" → "ready" states in both Web UI and Telegram.

### E. Sentinel ↔ Central Transport (Abstract)

**Pattern**: persistent outbound channel(s) (WebSocket/gRPC) carrying:

- registration, heartbeats, events/logs,
- multiplexed PTY frames for agent tiles and cwd shells,
- control intents down (exec/stop/open-shell/open-code).

**Sentinel responsibilities:**
- Maintains a single persistent connection to CLUD-CENTRAL per host
- Multiplexes all tracked agents' PTY streams over this connection
- Receives control intents from Central and dispatches them to the appropriate agent
- Handles agent lifecycle: spawn, monitor, cleanup on exit
- Survives individual agent crashes; exits when idle (configurable)

## 6) Terminal & PTY Strategy

- **Per-agent console PTY**: read-mostly; optionally enable stdin pass-through if needed.
- **CWD shell PTY (optional)**: interactive; idle TTL; allow-list; audit command lines.
- **Windows**: use pywinpty (ConPTY/winpty). POSIX: pty/os.openpty.
- **Streaming**: binary frames; 10–20ms coalescing; scrollback caps; pause rendering off-screen.

## 7) Security & Governance

- **No inbound to hosts**: sentinel binds to loopback only (127.0.0.1:7565); all remote paths originate at Central via outbound connection.
- **Scoped sessions**: short-lived tokens for terminal/VS Code; revoke on role change or timeout.
- **Directory sandbox**: cwd jail for shells/exec; no `..` traversal.
- **Audit**: append-only event log (who/when/what); optional retention settings.
- **Secrets**: store bot/API secrets via OS keyring or env (dev); no plaintext in repo.
- **Sentinel isolation**: each tracked agent runs in its own PTY; sentinel enforces resource limits and idle timeouts.

## 8) Packaging & Build (Python + npm)

### Monorepo layout:

```
clud-central/
  pyproject.toml
  clud_central/
    __init__.py
    app/            # FastAPI/Flask app
    bot/            # Telegram bridge
    static/         # built assets from packages/web
    templates/
  packages/web/     # npm project (React + xterm.js)
```

### Build pipeline:

1. `npm ci && npm run build` in `packages/web/`
2. copy artifacts to `clud_central/static/`
3. `uv build` (or poetry build) → wheel/sdist includes static via package_data
4. CLI entrypoints: `clud-central serve`, `clud-central bot`.

## 9) Operations & Observability

- **Dev**: single process, SQLite, .env secrets; hot reload for web frontend.
- **Prod (later)**: uvicorn behind reverse proxy; Postgres; optional Redis for fan-out; Prometheus metrics.
- **Health**: heartbeat freshness per agent; session counts; PTY throughput; VS Code warmup durations.

## 10) Rollout Plan (phases)

1. **MVP**: registration/heartbeats; tile per agent (console PTY read); Telegram `/bind` + `/status`; on-demand VS Code (just show state).
2. **Shells + Tail**: optional cwd shell with TTL & audit; Telegram tails & allow-listed runs.
3. **Scale & polish**: persisted layouts; RBAC; Postgres; metrics; log retention.

## 11) Risks & Mitigations

- **Browser render pressure (many tiles)**: coalesce frames; pause hidden; cap scrollback.
- **PTY floods (noisy builds)**: server-side throttling/backpressure; per-tile rate caps.
- **VS Code bloat**: clear idle timeout; single instance per agent; "pre-warm once per host" (optional).

---

## Technology Stack Details

### PTY Handling in Python

- **Unix**: Use the `pty` module with `openpty()` to spawn shells inside a pseudo-terminal. Functions like `spawn()` connect the child process's terminal and copy data between them.
- **Windows**: Use `pywinpty` for ConPTY/winpty support, allowing creation and communication with Windows console processes.

### Web Terminal Projection

The PTY acts as a "tunnel" to project each agent's interactive console into the browser:

1. **Client**: User runs `clud --track` → client ensures sentinel is running on 127.0.0.1:7565.
2. **Sentinel (Backend)**: Each agent's shell runs inside its own PTY. The sentinel reads from the PTY's master file descriptor and forwards output over WebSocket to CLUD-CENTRAL.
3. **CLUD-CENTRAL**: Receives multiplexed PTY streams from sentinel and fans out to connected web clients.
4. **Frontend**: xterm.js instances in the browser connect via WebSocket to CLUD-CENTRAL. Keystrokes are sent back through CLUD-CENTRAL → sentinel → PTY.
5. **Communication**: WebSocket provides full-duplex connection for streaming terminal data continuously.

### Grid Layout

- Multiple xterm.js instances arranged in a tiled grid (CSS Grid or react-grid-layout).
- Each tile connects to its own PTY stream over a separate WebSocket.
- Inspiration from tmux, dvtm, Zellij for tiling terminal behavior.

---

## Shared Tech Choices (CLUD + CLUD-CENTRAL)

- **Python toolchain**: uv or Poetry for fast, reproducible builds.
- **Async runtime**: uvloop (optional) with FastAPI; or Flask-SocketIO if you prefer eventlet/gevent.
- **Terminal stack**: pty (POSIX) / pywinpty (Windows); xterm.js on the frontend.
- **WebSockets**: websockets/Starlette WS (FastAPI) or Socket.IO (Flask-SocketIO)—choose one stack per repo for simplicity.
- **CI**: build npm assets first, then Python wheels; run minimal e2e PTY tests on Linux and Windows.
- **Security defaults**: loopback daemon; outbound-only to Central; short-lived session tokens; cwd jail; command allow-lists; audit trail.
