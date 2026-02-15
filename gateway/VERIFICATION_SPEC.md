# Gateway Verification Specification

## Execution Envelope Framework — v0.1

---

## Overview

The **Gateway** (also called Runtime Proxy) is the enforcement layer of the Execution Envelope framework. It is the single authorized path through which an agent may produce real-world side effects.

**The Gateway's invariant:**

> If an agent can reach external systems through any path other than the Gateway, the Execution Envelope architecture has failed.

This document specifies the Gateway's required behaviors: envelope validation, per-request enforcement, circuit breaking, audit logging, and independence requirements.

---

## 1. Independence Requirements

These properties must hold before the Gateway can be considered part of a valid Execution Envelope architecture.

### 1.1 Control Domain Separation

The Gateway and its configuration must run **outside the agent's control domain**. Specifically:

- The agent may **not** modify Gateway source code or configuration
- The agent may **not** restart or stop the Gateway process
- The agent may **not** modify the active Envelope after it has been verified
- The agent may **not** issue capability tokens to itself
- Gateway process lifecycle is managed by the platform team, not the agent

If any of the above conditions are violated, the enforcement guarantee is void.

### 1.2 Credential Isolation

- Exchange API keys, database credentials, and all external authentication material are held **exclusively** by the Gateway
- These credentials are injected via secrets manager at Gateway startup; they are never written to environment variables readable by the agent process
- The agent has no mechanism to read, copy, or transmit credentials

### 1.3 Network Isolation

- The agent process runs in a network-restricted environment (container or VM)
- The agent's only permitted outbound destination is the Gateway's internal address
- Direct connections to external systems (exchanges, APIs, databases) are blocked at the network layer (firewall rules / egress policy)
- This is enforced at the infrastructure level, not by agent code

### 1.4 Immutability of Active Envelope

Once an Envelope has been verified and execution has begun:

- The Envelope document is treated as immutable
- The Gateway holds the canonical copy
- Any request by the agent to modify, extend, or replace the active Envelope must be rejected and logged

---

## 2. Envelope Validation (Pre-Execution)

Before permitting any execution, the Gateway must perform a full envelope validation pass.

### 2.1 Schema Validation

```
MUST: envelope conforms to envelope.schema.json
MUST: all required fields present
MUST: envelope_id is unique (not previously used)
MUST: version is supported by this Gateway version
```

### 2.2 Integrity Verification

```
MUST: envelope signature is valid (signed by trusted AuthZ service)
MUST: envelope has not been modified since signing
MUST: envelope has not expired (meta.expires_at > now)
```

### 2.3 Template Conformance

If `template_ref` is present:

```
MUST: all capabilities are subset of template's allowed_capability_types
MUST: all budget values are within template's budget_ceilings
MUST: all forbidden entries include the template's forbidden_baseline
MUST: all required_postconditions from template are present
MUST: all required_circuit_breakers from template are present
```

### 2.4 Approval Verification

```
IF approval.required == true:
  MUST: approval.approved_by is a recognized approver identity
  MUST: approval.approved_at is present and within acceptable window
  MUST: approval.approval_token is valid and unexpired

IF approval.required == false:
  MUST: envelope satisfies all auto-approve conditions in template
  IF NOT: reject with APPROVAL_REQUIRED error
```

### 2.5 Validation Result

```
PASS  → Gateway activates the envelope; execution may begin
FAIL  → Gateway rejects; logs VALIDATION_FAILED with reason; no execution
```

---

## 3. Per-Request Enforcement

For every action request submitted by the agent, the Gateway runs the following check sequence **in order**. Any failure results in immediate rejection.

```
CHECK 1: Forbidden effects
  Is the requested action type in envelope.forbidden?
  → REJECT with FORBIDDEN_EFFECT

CHECK 2: Capability authorization
  Is there a capability entry that covers this action type?
  → REJECT with CAPABILITY_NOT_GRANTED

CHECK 3: Scope conformance
  Does the request conform to the scope of the matched capability?
  (symbol in allowlist, method allowed, host in allowlist, etc.)
  → REJECT with SCOPE_VIOLATION

CHECK 4: Rate limit
  Does this request exceed the per-minute/per-hour rate of the matched capability?
  → REJECT with RATE_LIMIT_EXCEEDED

CHECK 5: Budget checks
  Would executing this request cause any budget value to be exceeded?
  (total actions, domain-specific limits, commission, etc.)
  → REJECT with BUDGET_EXCEEDED

CHECK 6: Envelope expiry
  Has the envelope expired (meta.expires_at <= now)?
  → REJECT with ENVELOPE_EXPIRED

CHECK 7: Circuit breaker state
  Is the Gateway currently in a halted state?
  → REJECT with CIRCUIT_BREAKER_ACTIVE

ALL CHECKS PASS → Execute the action via the Gateway's own credentials
```

---

## 4. Circuit Breaker

The Gateway maintains runtime state across the execution session and monitors circuit breaker triggers defined in `envelope.circuit_breaker.triggers`.

### 4.1 Trigger Evaluation

After every completed action, the Gateway evaluates all circuit breaker conditions against current runtime state:

- Realized P&L
- Open position size
- Order count (session, per-minute)
- Consecutive error count
- Time remaining in session

### 4.2 Trigger Actions

| Action | Behavior |
|---|---|
| `halt_and_cancel` | Reject all new requests; submit cancel for all open orders (within envelope capabilities); emit alert |
| `halt_only` | Reject all new requests; leave open orders as-is; emit alert |
| `alert_only` | Continue execution; emit high-priority alert |

### 4.3 Recovery

Recovery from a halted state is governed by `envelope.circuit_breaker.recovery`:

| Recovery mode | Allowed recovery path |
|---|---|
| `manual_only` | Human operator must explicitly release the halt via out-of-band tooling |
| `new_envelope_required` | A new Envelope must be issued, verified, and activated; the halted envelope is invalidated |

**The agent itself may never trigger recovery.** Any recovery request originating from the agent process must be rejected and logged.

---

## 5. Audit Log

The Gateway must produce an append-only audit log for every execution session.

### 5.1 Required Log Events

| Event | Required fields |
|---|---|
| `ENVELOPE_RECEIVED` | envelope_id, received_at |
| `VALIDATION_PASS` | envelope_id, validated_at, template_ref |
| `VALIDATION_FAIL` | envelope_id, reason, failed_check |
| `REQUEST_RECEIVED` | envelope_id, request_id, action_type, timestamp |
| `REQUEST_APPROVED` | envelope_id, request_id, check_results_summary, budget_state_snapshot |
| `REQUEST_REJECTED` | envelope_id, request_id, rejection_reason, failed_check |
| `EXTERNAL_CALL_MADE` | envelope_id, request_id, target_system, request_hash, response_status |
| `CIRCUIT_BREAKER_TRIGGERED` | envelope_id, trigger_condition, action_taken, runtime_state_snapshot |
| `POSTCONDITION_CHECKED` | envelope_id, condition_id, result, detail |
| `SESSION_COMPLETE` | envelope_id, completed_at, summary_stats |

### 5.2 Log Integrity Requirements

- Logs are written to an append-only store outside the agent's write access
- Each log entry includes a monotonic sequence number
- Each log entry is signed or hash-chained to detect tampering
- Logs must be retained for a minimum period defined by organizational policy

### 5.3 Audit Retrieval

Logs must support retrieval by:

- `envelope_id`
- time range
- action type
- rejection reason

---

## 6. Postcondition Verification

After execution completes (or is halted), the Gateway runs postcondition checks defined in `envelope.postconditions`.

### 6.1 Check Registry

Each `check` identifier in a postcondition references a registered verifier function in the Gateway. Unknown check identifiers cause execution to fail at validation time (see §2).

### 6.2 Result Handling

| Condition | `critical` | Outcome |
|---|---|---|
| Check passes | any | Log `POSTCONDITION_PASS` |
| Check fails | `true` | Log `POSTCONDITION_FAIL`; session marked as FAULTED |
| Check fails | `false` | Log `POSTCONDITION_WARN`; session marked as COMPLETED_WITH_WARNINGS |

### 6.3 Standard Check Implementations (v0.1)

| Check ID | Verifies |
|---|---|
| `all_orders_logged` | Every order submitted to Gateway appears in audit log |
| `all_risk_checks_passed` | No CHECK 1–6 rejections were overridden during session |
| `no_open_positions_at_session_end` | Position book shows zero open contracts |
| `realized_loss_within_daily_limit` | Realized P&L >= -max_daily_loss_usd |
| `audit_log_complete` | Audit log sequence is unbroken; no gaps |

---

## 7. Error Codes Reference

| Code | Meaning |
|---|---|
| `VALIDATION_FAILED` | Envelope did not pass pre-execution validation |
| `APPROVAL_REQUIRED` | Envelope requires human approval; none present |
| `FORBIDDEN_EFFECT` | Requested action is in the forbidden list |
| `CAPABILITY_NOT_GRANTED` | No capability entry covers this action type |
| `SCOPE_VIOLATION` | Action is within a capability type but outside its scope |
| `RATE_LIMIT_EXCEEDED` | Request exceeds per-capability rate limit |
| `BUDGET_EXCEEDED` | Request would cause a budget limit to be exceeded |
| `ENVELOPE_EXPIRED` | Envelope's expiry time has passed |
| `CIRCUIT_BREAKER_ACTIVE` | Gateway is in a halted state |
| `RECOVERY_FROM_AGENT_DENIED` | Agent attempted to trigger circuit breaker recovery |
| `ENVELOPE_MODIFICATION_DENIED` | Agent attempted to modify the active envelope |

---

## 8. What the Gateway Does Not Guarantee

The Gateway enforces behavioral legality within the Envelope. It does **not** guarantee:

- That the agent's strategy logic is correct
- That the agent's decisions will produce profitable or desirable outcomes
- That market conditions will not cause losses within permitted limits
- That the Envelope was correctly authored (this is the template governance layer's responsibility)
- Semantic correctness of any output produced by the agent

These require independent quality assurance mechanisms.
