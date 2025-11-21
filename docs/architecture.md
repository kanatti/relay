# Relay Architecture

## Overview

Relay consists of two main components: a daemon (`relayd`) and a CLI (`relay`). This separation provides clean boundaries between execution logic and user interface.

```
┌─────────────────┐     ┌─────────────────┐
│   relay CLI     │     │   AI Agent      │
│  (human UX)     │     │  (HTTP client)  │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │    HTTP/gRPC API      │
         └───────────┬───────────┘
                     ▼
            ┌─────────────────┐
            │     relayd      │
            │    (daemon)     │
            ├─────────────────┤
            │ • Policy Engine │
            │ • Session Mgmt  │
            │ • Sandbox Exec  │
            │ • Stream Mgmt   │
            │ • Audit Logger  │
            └─────────────────┘
```

---

## `relayd` — The Daemon

The **execution brain**. Runs as a long-lived background process.

### Responsibilities

| Component | Purpose |
|-----------|---------|
| **API Server** | HTTP/gRPC endpoints for sessions, commands, queries |
| **Policy Engine** | Loads `relay.toml`, enforces allow/deny decisions |
| **Session Manager** | Creates/tracks sessions, maintains state |
| **Sandbox Executor** | Spawns subprocesses with isolation (cwd, env, limits) |
| **Stream Manager** | Captures stdout/stderr, fans out to clients |
| **Audit Logger** | Writes JSONL logs for all command events |

### Why a Daemon?

- **Single enforcement point** — all requests (human or agent) pass through the same policy gates
- **Session persistence** — holds state across multiple commands; clients can reconnect
- **Streaming** — manages subprocess lifecycle; clients just consume events
- **Centralized audit** — one place logs everything

---

## `relay` — The CLI

A **thin client** for humans. Talks to `relayd` over HTTP/gRPC.

### Commands

```bash
relay run "kubectl get pods"      # execute command, stream output
relay simulate "rm -rf /"         # dry-run, show policy decision
relay plan "cat foo | grep bar"   # show execution plan for pipeline
relay timeline                    # view command history
relay graph                       # view command dependency graph
relay changes --session s1        # view filesystem diff
relay shell                       # interactive REPL
```

### What It Doesn't Do

- No execution logic
- No policy enforcement
- No subprocess management

It's purely UX — formatting, colors, interactive prompts.

---

## Why the Split?

| Concern | Benefit |
|---------|---------|
| **Agent access** | Agents connect directly to `relayd` via API — no shell parsing |
| **Consistency** | Humans and agents get identical policy enforcement |
| **Stateful sessions** | Daemon holds session state; CLI is stateless |
| **Observability** | Single audit log for all activity |
| **Deployability** | `relayd` can run remotely; CLI connects over network |

---

## Alternative: Embedded Mode

For simpler local use, `relay` could spawn an embedded `relayd` on-demand (single-binary mode). This is a future consideration — MVP assumes the split architecture.
