# Relay — Concept, Origins, and MVP Design (Exploratory & Cutting‑Edge)

## 1. Introduction

Relay is a **safe, composable, agent‑friendly command execution layer**. It acts as a **controlled shell** for AI agents, letting them run real CLI commands with safety, oversight, policy gates, sandboxing, and full traceability — without losing the natural composability of the Unix shell.

Relay preserves the power of CLI while reintroducing safety, structure, and visibility typically associated with MCP.

It’s experimental, forward‑looking, and built for the emerging world where agents must interact with real infrastructure.

---

## 2. Origin of the Idea

The concept came from observing major limitations in today’s agent tooling landscapes:

### **2.1 The MCP Limitation**

MCP is safe, structured, typed — but loses one thing that makes CLI magical:
**composability.**

* No pipes
* No streaming
* No chaining
* No scripting
* No emergent tool‑building

Agents can’t combine simple tools to solve complex tasks. They’re limited to predefined RPC‑style tool calls.

### **2.2 The CLI Limitation**

CLI is infinitely powerful… **but unsafe for agents**.

* Arbitrary command execution
* No guardrails
* No policies
* No confirmations
* Full system access
* Fragile quoting/escaping

### **2.3 The Insight**

There is a **missing layer**:
A **safe shell** that preserves CLI expressiveness while giving MCP‑level safety.

Thus Relay was born — **a safety‑net CLI**.

Relay = the bridge between structured agents and real-world system commands.

---

## 3. What Relay Is

### **Relay is:**

* A **controlled, policy‑aware shell** for agents.
* A **sandbox** for executing real commands safely.
* A **session‑oriented command runner** with streaming output.
* A **trace and visualization layer** for command history.
* An **inspection, simulation, and planning surface** for agents.

### **Relay is not:**

* A replacement for bash.
* A replacement for MCP.
* A full VM/Firecracker isolation layer (not initially).

Relay complements existing agent ecosystems.

---

## 4. MVP Core Pillars

Relay MVP is intentionally sharp and compact. It has four pillars.

### ### **4.1 Pillar 1 — Policy & Safety Layer**

Relay’s most important feature.

Each allowed command/binary has a policy describing:

* allowed/denied args
* max args
* whether confirmation is required
* whether destructive patterns are blocked

**Example (`relay.toml`):**

```toml
[commands.kubectl]
allowed = true
args_allow = ["get", "describe", "logs"]
args_deny  = ["delete", "apply", "exec"]
require_confirm = true

[commands.git]
allowed = true
args_deny = ["push", "reset", "clean", "rebase"]

[global]
default_allowed = false
log_dir = "./relay-logs"
```

This gives agents **predictable, explainable safety boundaries**.

---

### **4.2 Pillar 2 — Sandboxed Execution**

Not full virtualization, but enough to create strong boundaries:

* force working directory into `relay-root`
* controlled env variables
* deny sensitive env vars
* timeouts
* output length limits

**Config:**

```toml
[sandbox]
working_dir = "./relay-root"
env_pass_through = ["PATH", "AWS_PROFILE"]
env_block = ["AWS_SECRET_ACCESS_KEY", "KUBECONFIG"]
```

Tight, simple, safe.

---

### **4.3 Pillar 3 — Sessions + Streaming (Agent Ready)**

Relay exposes sessions so agents can:

* maintain context
* run multiple commands sequentially
* stream output
* receive incremental updates
* cancel commands dynamically

**API shape:**

* `POST /sessions`
* `POST /sessions/{id}/commands`
* streaming stdout/stderr events
* `DELETE /sessions/{id}/commands/{cid}` to cancel

This mimics a real TTY without fully emulating one.

---

### **4.4 Pillar 4 — Tracing + Visualizations**

Relay logs all command events in JSON for auditing.

**Example log:**

```json
{
  "timestamp": "2025-11-20T20:10:00Z",
  "session_id": "s1",
  "command": "kubectl",
  "args": ["get", "pods"],
  "exit_code": 0,
  "duration_ms": 532,
  "stdout_snippet": "NAME   READY..."
}
```

Visual tools:

### **`relay timeline`**

```
Session s1
  20:10 kubectl get pods      [OK]
  20:11 kubectl logs foo       [OK]

Session s2
  20:12 aws s3 ls              [DENIED]
```

### **`relay graph`**

```
[kubectl get pods] --> [kubectl logs pod/foo]
```

Simple, text‑based, helpful.

---

## 5. Experimental & Exploratory Features (MVP‑Friendly)

Relay should feel **cutting‑edge**, a glimpse into the future of agentic ops.

### **5.1 Pipeline Planner**

Relay understands shell pipelines:

Input:

```
kubectl get pods | grep api | wc -l
```

Relay returns a **plan**:

```json
{
  "steps": [
    {"id": 1, "cmd": "kubectl", "args": ["get", "pods"]},
    {"id": 2, "cmd": "grep", "args": ["api"], "input_from": 1},
    {"id": 3, "cmd": "wc", "args": ["-l"], "input_from": 2}
  ]
}
```

Agents see if each step is allowed.

---

### **5.2 Simulation Mode**

```
relay simulate "kubectl delete pod foo | grep bar"
```

Output:

```
[1] kubectl delete pod foo → DENIED
[2] grep bar               → ALLOWED (but upstream blocked)
```

Agents can reason about safety *before* execution.

---

### **5.3 File-System Diff Sandbox**

Relay takes a shallow file snapshot before/after command:

* filenames
* sizes
* timestamps

Then shows:

```
relay changes --session s5 --cmd 003

Created:
  logs/output.txt

Modified:
  configs/app.yaml
```

This helps agents understand side‑effects.

---

## 6. Overall Architecture

Relay consists of four components.

### **6.1 relayd (daemon)**

* HTTP/gRPC API
* Command planner
* Sandbox executor
* Logging engine
* Policy enforcer

### **6.2 relay (CLI)**

* Human UX
* Commands: `run`, `simulate`, `timeline`, `graph`, `changes`, `shell` (attach)
* Dev mode & debugging tools

### **6.3 Policy/Config Engine**

* Load & validate `relay.toml`
* Provide allow/deny decisions and explanations

### **6.4 Log Store**

* MVP: JSONL files
* Future: SQLite/Postgres integration

---

## 7. Why Relay Matters

The world is shifting to **agentic ops**. Agents need:

* real‑world access
* real commands
* real infrastructure interaction
* but with **absolute safety**

Pure CLI is too dangerous.
Pure MCP is too limited.

Relay is the missing execution layer — a controlled, explainable, composable environment where agents can act.

Relay enables:

* safer automation
* observable decision paths
* hybrid human+agent workflows
* progressive adoption of AI ops

---

## 8. Experimental Vision Beyond MVP

These are future‑facing but consistent with the philosophy:

* full Firecracker/MicroVM isolation
* TUI/web visual debugger
* replayable command sessions
* continuous safety learning (adaptive policies)
* agent‑specific personalities (builder, operator, planner)
* auto‑summaries for every session
* event‑driven audit feeds

Relay becomes a **developer-first agentic shell**.

---

## 9. Final Summary

Relay is:

* Easy to remember
* Easy to type
* Relatable
* Practical yet futuristic
* A perfect name for a safety‑net CLI for agents

The MVP is strong, cutting-edge, experimental but grounded in real needs. It focuses on:

* policy
* sandboxing
* sessions
* streaming
* traceability

And with exploratory features like planning, simulation, and diff tracking, Relay feels **modern**, **agent‑native**, and **visionary**.

---

Relay isn’t just a tool — it’s a foundational layer for the agentic future.

