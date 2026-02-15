# Threat Model

## Execution Envelope Framework — v0.1

---

## Scope

This threat model defines what the Execution Envelope framework is designed to prevent, what it explicitly does not address, and the assumptions under which its guarantees hold.

---

## Assumed Adversarial Conditions

The framework assumes the following may occur in a real deployment:

| Condition | Implication |
|---|---|
| The LLM may generate actions that exceed its authorized scope | Capability enforcement must be structural, not trust-based |
| The LLM may misinterpret the task goal | Envelope bounds must hold even under misinterpretation |
| Strategy code may contain bugs or be maliciously modified | Credential isolation and network isolation must be physical facts |
| An attacker may attempt prompt injection to expand the agent's scope | The agent cannot modify its own Envelope; forbidden effects cannot be overridden |
| The agent may attempt to recover from a circuit-break itself | Recovery paths must be explicitly outside agent control |
| A compromised agent may attempt to access external systems directly | Network egress rules must be enforced at infrastructure level |

---

## Threats Addressed

### T1: Unauthorized Resource Access
**Threat:** Agent accesses resources (databases, APIs, accounts) not authorized for the current task.
**Mitigation:** Capability list is exhaustive; anything not listed is implicitly denied. Scope constraints limit access within granted capabilities.

### T2: Budget Overrun
**Threat:** Agent consumes unbounded resources (orders, API calls, compute time, money).
**Mitigation:** Budget limits are enforced per-request by the Gateway. Circuit breakers halt execution when thresholds are reached.

### T3: Irreversible Harmful Action
**Threat:** Agent takes an action that cannot be undone (fund withdrawal, data deletion, large position).
**Mitigation:** Forbidden effects list is checked independently of capabilities and cannot be overridden. High-risk actions require human approval.

### T4: Governance Bypass
**Threat:** Agent modifies its own Envelope, Gateway configuration, or circuit breaker state to expand its own permissions.
**Mitigation:** Gateway and Envelope configuration are immutable during execution and managed outside the agent's control domain.

### T5: Credential Theft
**Threat:** Agent reads or exfiltrates API keys or credentials to enable direct external access.
**Mitigation:** Credentials are held exclusively by the Gateway process; agent has no read access.

### T6: Direct External Access
**Threat:** Agent connects directly to external systems, bypassing the Gateway.
**Mitigation:** Agent network egress is restricted at infrastructure level; only the Gateway's internal address is reachable.

### T7: Unauthorized Recovery
**Threat:** Agent self-recovers from a circuit-break state to resume execution.
**Mitigation:** Recovery is explicitly restricted to manual operator action or new Envelope issuance.

### T8: Envelope Tampering
**Threat:** Envelope is modified in transit between issuance and Gateway activation.
**Mitigation:** Envelopes are cryptographically signed; Gateway verifies signature before activation.

---

## Threats NOT Addressed

The following are out of scope for this framework. They require independent mitigation.

| Threat | Reason out of scope |
|---|---|
| LLM semantic errors (hallucination, misunderstanding) | Framework bounds behavior, not correctness |
| Strategy algorithm producing incorrect signals | Correctness is a quality concern, not a governance concern |
| Financial losses within permitted limits | Losses within Envelope bounds are permitted by design |
| Data accuracy of inputs provided to the agent | Data quality is a separate concern |
| Collusion between the human approver and a malicious agent | Assumes the approval chain is trusted |
| Compromise of the AuthZ service that issues Envelopes | Assumes the issuing authority is trusted |
| Side-channel attacks on the Gateway process itself | Assumes Gateway process integrity |

---

## Trust Assumptions

The framework's guarantees hold only when the following are true:

1. **The Gateway process is not compromised.** The Gateway is the root of the enforcement trust chain.
2. **The AuthZ service is not compromised.** Envelope signatures are only meaningful if the signing key is secure.
3. **Infrastructure isolation is correctly configured.** Network egress rules and credential isolation must be implemented correctly.
4. **Envelope templates are correctly authored.** A template that grants overly broad capabilities produces overly broad Envelopes.
5. **The approval process is not circumvented.** Human approvers must review Envelopes before signing off.

---

## Security Properties (When Assumptions Hold)

| Property | Guarantee |
|---|---|
| **Containment** | Agent actions are bounded by the Envelope regardless of agent behavior |
| **Non-bypassability** | No path exists for the agent to reach external systems except through the Gateway |
| **Auditability** | Every agent action and Gateway decision is logged in a tamper-evident log |
| **Revocability** | Execution can be halted at any time by circuit breaker or operator |
| **Ephemerality** | Envelope permissions expire; they are not persistent grants |
