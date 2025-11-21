# Proxy Mode

Relay's proxy mode provides visibility and control over HTTP(S) traffic originating from within sandboxed commands. This solves the "inner layer" problem — when a subprocess (e.g., a Python script) spawns multiple threads making concurrent network calls.

## The Problem

```
relay run "python agent.py"
    └── python spawns 50 threads
        └── each thread calls requests.post(opensearch)
```

Relay sees **1 command**. The 50 parallel HTTP calls are invisible — they happen inside the subprocess.

## The Solution

Relay spawns a local HTTP proxy and forces all subprocess traffic through it via environment variables.

```
┌─────────────────────────────────────────────┐
│              relay sandbox                  │
│                                             │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │ python.py   │ ───▶ │  relay-proxy     │  │──▶ external
│  │ (50 threads)│      │  :9999           │  │    targets
│  └─────────────┘      ├──────────────────┤  │
│                       │ • rate limiting  │  │
│  HTTP_PROXY=:9999     │ • per-target RPS │  │
│  HTTPS_PROXY=:9999    │ • circuit breaker│  │
│                       │ • request logging│  │
│                       └──────────────────┘  │
└─────────────────────────────────────────────┘
```

Every HTTP(S) call from the subprocess routes through Relay's proxy — full visibility and control.

---

## Capabilities

| Control | Description |
|---------|-------------|
| **Global RPS** | Token bucket rate limit across all targets |
| **Per-target RPS** | Separate rate limits per host/domain |
| **Concurrency** | Max in-flight requests (global and per-target) |
| **Circuit breaker** | Trip on error rate or latency threshold |
| **Request logging** | Every request written to audit log |
| **Host blocklist** | Deny calls to certain hosts/domains |

---

## Configuration

```toml
[proxy]
enabled = true
bind = "127.0.0.1:9999"

global_max_rps = 50
global_max_concurrent = 10

[proxy.targets.opensearch]
hosts = ["opensearch.internal", "*.es.amazonaws.com"]
max_rps = 10
max_concurrent = 3
circuit_breaker = { error_rate = 0.5, window_secs = 30 }

[proxy.targets.postgres]
hosts = ["db.internal"]
max_rps = 20
max_concurrent = 5

[proxy.targets.default]
max_rps = 20
max_concurrent = 5

[proxy.blocklist]
hosts = ["metadata.google.internal", "169.254.169.254"]  # block cloud metadata
```

### Configuration Fields

**`[proxy]`**
- `enabled` — activate proxy mode
- `bind` — address for proxy server
- `global_max_rps` — total requests/sec across all targets
- `global_max_concurrent` — total in-flight requests

**`[proxy.targets.<name>]`**
- `hosts` — list of host patterns (supports wildcards)
- `max_rps` — requests/sec limit for this target
- `max_concurrent` — max in-flight requests for this target
- `circuit_breaker.error_rate` — error ratio (0.0–1.0) to trip breaker
- `circuit_breaker.window_secs` — time window for error rate calculation

**`[proxy.blocklist]`**
- `hosts` — requests to these hosts are rejected

---

## Circuit Breaker States

```
         success
    ┌───────────────┐
    ▼               │
┌────────┐    ┌─────┴─────┐    ┌────────┐
│ CLOSED │───▶│   OPEN    │───▶│ HALF   │
└────────┘    └───────────┘    │ OPEN   │
  errors        timeout        └────┬───┘
  exceed        expires             │
  threshold                    success → CLOSED
                               failure → OPEN
```

- **CLOSED** — normal operation, requests pass through
- **OPEN** — target tripped, all requests fail fast
- **HALF-OPEN** — allow limited requests to test recovery

---

## HTTPS Handling

Two modes:

### 1. MITM Mode (full inspection)
- Relay generates a CA certificate
- Subprocess must trust the CA
- Full request/response visibility
- Required for body inspection or response-based rate limiting

### 2. CONNECT Passthrough (host-level only)
- No certificate generation
- Rate limiting based on host only (no body inspection)
- Simpler, no trust setup required

```toml
[proxy]
https_mode = "passthrough"  # or "mitm"
ca_cert_path = "./relay-ca.pem"  # for mitm mode
```

---

## Environment Injection

When proxy mode is enabled, Relay injects into the sandbox environment:

```bash
HTTP_PROXY=http://127.0.0.1:9999
HTTPS_PROXY=http://127.0.0.1:9999
http_proxy=http://127.0.0.1:9999
https_proxy=http://127.0.0.1:9999
NO_PROXY=localhost,127.0.0.1
```

Both uppercase and lowercase variants for compatibility.

---

## Logging

Every proxied request is logged:

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "session_id": "s1",
  "command_id": "c3",
  "request": {
    "method": "POST",
    "host": "opensearch.internal",
    "path": "/_search",
    "size_bytes": 1024
  },
  "response": {
    "status": 200,
    "size_bytes": 4096,
    "latency_ms": 45
  },
  "target": "opensearch",
  "rate_limit": {
    "remaining_rps": 7,
    "concurrent": 2
  }
}
```

---

## Limitations

| Limitation | Notes |
|------------|-------|
| **Non-HTTP traffic** | Raw TCP, gRPC-over-HTTP/2, WebSockets not fully supported. Needs eBPF for full coverage. |
| **Proxy-ignoring tools** | Some tools bypass proxy env vars (e.g., `curl` without `-x`). Document for agent authors. |
| **Performance overhead** | Additional hop adds ~1-2ms latency per request |
| **HTTPS inspection** | MITM mode requires CA trust setup |

---

## Implementation Notes

### Rust Crates
- **hyper** — HTTP proxy server
- **tower** — middleware (rate limiting, circuit breaker)
- **rustls** — TLS termination for MITM mode
- **governor** — token bucket rate limiting

### Architecture

```
relay-proxy
├── server.rs         # hyper server, connection handling
├── middleware/
│   ├── rate_limit.rs # token bucket per-target
│   ├── circuit.rs    # circuit breaker state machine
│   └── logging.rs    # request/response audit
├── targets.rs        # host matching, target resolution
└── config.rs         # proxy config parsing
```

---

## Future Considerations

- **Adaptive rate limiting** — adjust limits based on observed target latency
- **Request priorities** — queue low-priority requests when near limit
- **gRPC support** — HTTP/2 aware proxying
- **Metrics export** — Prometheus endpoint for proxy stats
