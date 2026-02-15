# Framework Principles

## Execution Envelope — v0.1

---

These principles are not guidelines. They are the conditions under which the framework's guarantees hold. A system that violates any of these is not an Execution Envelope–compliant system.

---

## P1: The Envelope Precedes the Code

No agent action is permitted without a valid, verified Execution Envelope.

The Envelope defines the permissible action space before any code runs. Code is an implementation detail operating within that space. The Envelope is the authority.

*Corollary: A system where code executes and the Envelope is consulted afterwards is not Envelope-Centric.*

---

## P2: Enforcement Is Not Optional

If an agent can reach external systems through any path other than the enforcement layer (Gateway), the framework has failed.

Enforcement must be a **physical and network-layer fact**, not a convention or a behavioral expectation. "The agent shouldn't do X" is not enforcement. "The agent cannot do X because the path doesn't exist" is enforcement.

---

## P3: The Control Plane Is Independent

The Envelope definition, the enforcement logic, and the Gateway process must operate **outside the agent's control domain**.

An agent that can:
- modify its own Envelope
- alter Gateway check logic
- restart the Gateway process
- issue its own capability tokens

...has no effective governance layer. Control plane independence is not a deployment detail; it is a correctness requirement.

---

## P4: Envelopes Are Ephemeral Capability Instances

An Execution Envelope is not a long-term permission grant. It is a scoped, time-bounded authorization for a single task execution.

Envelopes expire. Expired envelopes are invalid. The agent's permissions do not persist between executions.

*This is the key distinction from IAM roles and static policy grants, which are long-lived identity-based permissions.*

---

## P5: The Agent Is a Parameter Instantiator, Not a Rule Author

The agent may instantiate Envelope parameters within bounds defined by human-authored templates. The agent may not create new capability types, extend scope beyond template bounds, or override forbidden effects.

The trust hierarchy is:

```
Organization (defines templates)
    ↓
Verifier + Approver (validates instances)
    ↓
Agent (instantiates parameters within bounds)
```

Power flows downward. The agent operates at the bottom of this hierarchy.

---

## P6: Compliance ≠ Correctness

The Execution Envelope guarantees **behavioral legality** — that the agent's actions fall within the defined permissible space.

It does **not** guarantee that legally-bounded behavior produces correct, desirable, or profitable outcomes. Quality assurance requires independent mechanisms: testing, validation, human review, reference checks.

A system operator who believes an Execution Envelope eliminates the need for quality assurance has misunderstood the framework's scope.

---

## P7: Circuit Breaking Is the Agent of Last Resort

When runtime state indicates that continued execution is dangerous (loss limits exceeded, error rate too high, session end approaching), the Gateway must halt execution immediately.

The agent may not self-recover from a halted state. Recovery is an out-of-band human action, or requires issuance of a new Envelope. This ensures that the decision to resume is always a human decision.

---

## P8: Every Action Is Auditable

The Gateway must produce a complete, tamper-evident audit log of every action request, every enforcement decision, every circuit breaker event, and every postcondition check.

Auditability is not optional. A system where agent behavior cannot be fully reconstructed after the fact does not meet the framework's accountability requirements.
