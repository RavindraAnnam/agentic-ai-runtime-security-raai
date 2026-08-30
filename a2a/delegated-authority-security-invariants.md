# Delegated Authority Security Invariants and Adversarial Conformance Vectors for Multi-Agent Systems

**Author:** Ravindra Annam  
**Status:** Draft for Technical Co-Review  
**Version:** 0.2
**Date:** August 2026  

> This note proposes candidate runtime security properties for delegated authority in multi-agent systems. These are not requirements of the current A2A protocol specification. The document is intended for technical co-review against the A2A authorization model and the ERDL design discussed in A2A Discussion #2031.

---

## Abstract

Multi-agent systems increasingly delegate tasks across independent agents, skills, tools, APIs, and protected resources. Authentication and conventional access control establish important security boundaries, but they do not by themselves guarantee that the effective authority associated with a delegated task remains bounded throughout a multi-hop execution chain.

This note proposes five runtime security invariants for delegated authority:

1. Authority Non-Amplification
2. Authority Provenance and Temporal Continuity
3. Constraint Inheritance / Narrow-Only Delegation
4. Enforcement-Boundary Revocation
5. Capability-Boundary Non-Amplification

It also proposes eight adversarial conformance vectors for testing these properties across:

**Principal → Agent → Agent → Skill → Tool/API → Protected Resource**

The proposal is implementation-neutral. It defines security properties rather than prescribing a specific policy engine, token format, authorization architecture, or enforcement mechanism.

---

# 1. Problem Statement

Consider a multi-agent workflow:

**Principal P → Agent A → Agent B → Skill S → Tool T → Protected Resource R**

Principal P authorizes Agent A to perform Task T.

Agent A subsequently delegates some or all of that task to Agent B.

Agent B may possess broader standing privileges than Agent A. Agent B may also invoke a Skill or Tool whose underlying technical privileges exceed the authority associated with the originating task.

This creates an important distinction:

> **Standing authority is not necessarily effective task authority.**

For example, Principal P authorizes Agent A to approve a refund of no more than $500.

Agent A delegates the task to Agent B.

Agent B normally possesses authority to approve refunds up to $5,000.

The delegation from A to B should not automatically transform a $500-authorized task into a $5,000-authorized task merely because B possesses broader standing privileges.

The relevant security question is therefore not only:

> Is Agent B authenticated and permitted to access this service?

It is also:

> Is this particular action permitted by the effective authority derived from the originating authorization and its delegation chain?

This note proposes security properties for answering the second question.

---

# 2. Authority Model

For Task T, effective authority may be viewed conceptually as the authority currently available to an actor for that specific task after applicable authorization and delegation constraints have been evaluated.

Let:

- `SA(A)` = standing authority of Agent A
- `EA(A,T)` = effective authority available to Agent A for Task T
- `C(T)` = constraints associated with Task T
- `D(T)` = delegation lineage associated with Task T

An agent may possess standing authority greater than the authority legitimately available for a particular delegated task.

Therefore, in the general case:

`SA(A) != EA(A,T)`

For delegated execution, effective authority available downstream should remain bounded by authority legitimately conveyed through the delegation chain.

---

# 3. Security Invariants

## INV-01 — Authority Non-Amplification

For a delegation from Agent A to Agent B for Task T:

`EA(B,T) ⊆ EA(A,T)`

A delegatee MAY receive equal or narrower authority.

A delegation MUST NOT silently increase the effective authority available to the task.

A delegatee's independent standing privileges MUST NOT automatically become available to the delegated task.

### Example

Agent A has effective task authority:

`refund <= $500`

Agent B has standing authority:

`refund <= $5,000`

Agent A delegates the task to Agent B.

Without an independent legitimate re-authorization event, Agent B's effective authority for the delegated task remains:

`refund <= $500`

and not:

`refund <= $5,000`

### Security Objective

A principal or upstream agent should not acquire authority indirectly by selecting a delegatee possessing privileges unavailable to the originating task.

### Delegation Depth and Loop Safety

Authority non-amplification alone does not prevent unbounded or cyclic delegation.

A delegation chain MUST NOT exceed the configured maximum delegation depth. Implementations MUST detect and reject delegation loops or repeated chain elements.

The delegation-depth constraint is independent of the authority ceiling: a delegation may remain within the originating authority scope and still be invalid because the permitted delegation depth has been exceeded.

Accordingly, a conforming implementation MUST validate both:

1. effective-authority containment; and
2. delegation-chain depth and loop safety.

This property is implementation-neutral. Chain depth and ordering may be represented through sequence numbers, lineage records, signed delegation objects, or an equivalent mechanism.

---

## INV-02 — Authority Provenance and Temporal Continuity

Security-sensitive actions SHOULD retain sufficient provenance to reconstruct the authority lineage that permitted the action.

For consequential operations, provenance SHOULD be capable of representing:

**Originating Principal  
→ Task/Intent  
→ Authorization State  
→ Delegation Chain  
→ Executing Agent  
→ Skill  
→ Tool/API  
→ Protected Resource  
→ Resulting Action**

Provenance should capture more than the identity of the final actor.

A security investigation should be able to determine:

1. Who originated the request?
2. What task or intent was authorized?
3. What authority was granted?
4. Which constraints applied?
5. How was authority delegated?
6. Which agent ultimately executed the operation?
7. Which skill, tool, or API produced the effect?
8. Which protected resource was affected?
9. Was the relevant authorization still valid when the operation occurred?

### Temporal Ordering

Valid lineage alone is insufficient if delegation evidence can be replayed or applied outside its legitimate execution sequence.

Authority-bearing delegation events MUST preserve valid temporal ordering. A delegation establishing authority for a task MUST precede acceptance or exercise of authority derived from that delegation.

Conceptually:

`DELEGATE.timestamp < TASK_ACCEPT.timestamp < AUTHORITY_EXERCISE.timestamp`

A replayed, stale, expired, or temporally inconsistent delegation record MUST NOT by itself establish current effective authority.

The exact representation of timestamps, sequence numbers, freshness proofs, or replay protection remains implementation-defined.

### Security Objective

Prevent an authenticated downstream action from becoming disconnected from the authority, constraints, and intent that originally permitted the workflow.

---

## INV-03 — Constraint Inheritance / Narrow-Only Delegation

Applicable authorization constraints MUST propagate downstream by default.

Examples include:

- action scope;
- resource scope;
- purpose;
- temporal restrictions;
- transaction or value limits;
- autonomy limits;
- geographic or environmental restrictions;
- approval requirements; and
- other authorization conditions.

A downstream delegation MAY narrow inherited constraints.

A downstream delegation MUST NOT silently remove or broaden them.

The default principle is:

> **inherit-by-default, narrow-only**

### Example

Original authorization:

`resource = Account-123`

`purpose = refund`

`amount <= $500`

`valid_until = 18:00 UTC`

A downstream delegation MAY narrow this to:

`resource = Account-123`

`purpose = refund`

`amount <= $250`

`valid_until = 17:30 UTC`

It MUST NOT silently transform the authorization into:

`amount <= $5,000`

or remove the resource, purpose, or temporal restriction.

### Security Objective

Constraint propagation should preserve the security meaning of the originating authorization throughout delegation.

---

## INV-04 — Enforcement-Boundary Revocation

Revocation MUST be evaluated at the relevant enforcement boundary before derived authority is exercised.

Consider:

**Principal P → Agent A → Agent B → Agent C**

If P revokes the authorization from which A's authority was derived, downstream authority derived solely from that authorization should no longer remain independently usable.

Conceptually:

`Revoke(P→A) ⇒ Invalidate(unexercised derived authority A→B→C)`

Revocation propagation across distributed or disconnected agents MAY be eventually consistent. However, an enforcement boundary MUST evaluate the currently available revocation state before permitting a security-sensitive side effect.

This invariant deliberately does not prescribe a specific freshness mechanism, propagation protocol, cache lifetime, revocation token, or synchronization architecture.

The normative security property is evaluation at the enforcement boundary; the mechanism by which sufficiently current revocation state reaches that boundary remains implementation-defined.

This does not imply reversal of actions that have already legitimately completed.

It applies to authority that remains pending or unexercised when its upstream authorization becomes invalid.

### Security Objective

A stale downstream delegation should not survive merely because the immediate delegatee has not independently observed the upstream revocation.

---

## INV-05 — Capability-Boundary Non-Amplification

Effective task authority MUST NOT increase merely because execution crosses into a downstream skill, tool, API, service identity, or protected-resource capability possessing broader standing privileges.

Consider:

**Agent A → Agent B → Skill S → Tool T → Protected Resource R**

If Agent B's effective authority for Task T permits only `read(R)`, invoking Skill S or Tool T MUST NOT make `write(R)` or `delete(R)` available to that task merely because the downstream component technically possesses those privileges.

The governing distinction is:

> **Credential Capability ≠ Authorized Task Authority**

Conceptually, authority must remain bounded across the capability path:

`EA(Resource,T) ⊆ EA(Tool,T) ⊆ EA(Skill,T) ⊆ EA(Agent,T)`

where each term represents authority legitimately exercisable for the originating task rather than the component's independent standing capability.

Selecting a more privileged downstream capability MUST NOT create a new authorization root.

If broader authority is required, it MUST originate from an independently legitimate authorization event that is explicitly represented in the authority provenance.

### Security Objective

Prevent agent-to-skill, skill-to-tool, tool-to-API, or tool-to-resource transitions from laundering broader technical capability into broader effective task authority.

# 4. Enforcement Boundary

These invariants intentionally do not prescribe a particular enforcement technology.

Implementations may use mechanisms such as:

- runtime policy evaluation;
- constrained credentials;
- capabilities;
- authorization services;
- policy enforcement points;
- signed authorization artifacts;
- task-scoped tokens;
- trusted execution components; or
- other architectures.

The important security property is:

> **A security-sensitive side effect should not execute unless the effective authority required for that action remains valid at the relevant enforcement boundary.**

This is especially important when execution crosses:

**Agent → Skill → Tool/API → Protected Resource**

because the agent may itself remain within its apparent permissions while selecting a downstream capability that possesses broader technical privileges.

---

# 5. Adversarial Conformance Vectors

The following negative tests are intended to function as security canaries.

A conforming implementation should prevent the unauthorized side effect in each scenario.

---

## AV-01 — Direct Authority Amplification

### Setup

Principal P grants Agent A effective authority L2 for Task T.

Agent A delegates T to Agent B.

Agent B independently possesses standing authority L3.

### Attack

Agent B attempts an L3 action while operating under Task T.

### Expected Result

**DENY**

### Invariant

INV-01 — Authority Non-Amplification

### Failure Condition

Agent B's independent L3 standing privilege becomes available merely because B received an L2 delegated task.

---

## AV-02 — Transitive Authority Amplification

### Setup

**P → A(L2) → B(L2) → C**

Agent C possesses independent L3 privileges.

### Attack

C attempts an L3 operation using authority derived from the original L2 task.

### Expected Result

**DENY**

### Invariants

INV-01 and INV-02

### Failure Condition

Each individual delegation appears locally valid while the resulting end-to-end effective authority exceeds the originating authority.

---

## AV-03 — Privileged Delegate Laundering

### Setup

Agent A is not authorized to perform privileged action X.

Agent B independently possesses authority to perform X.

### Attack

Instead of attempting X directly, A delegates a task to B that causes B to execute X.

### Expected Result

**DENY**

### Invariant

INV-01

### Failure Condition

Agent A effectively acquires unavailable authority by selecting a more privileged delegatee.

---

## AV-04 — Downstream Constraint Removal

### Setup

Principal P grants:

`purpose = refund`

`resource = Order-123`

`amount <= $500`

Agent A delegates the task to B while preserving those constraints.

### Attack

B delegates to C but removes the amount restriction or changes it to:

`amount <= $5,000`

### Expected Result

**DENY**

### Invariant

INV-03 — Constraint Inheritance / Narrow-Only Delegation

### Failure Condition

A downstream delegation silently weakens an inherited authorization constraint.

---

## AV-05 — Revoked Ancestor

### Setup

**P → A → B → C**

Agent C possesses unexercised authority derived from this chain.

Principal P subsequently revokes the authorization granted to A.

### Attack

C attempts to exercise the previously delegated authority.

### Expected Result

**DENY**

### Invariant

INV-04 — Enforcement-Boundary Revocation

### Failure Condition

C continues operating because its immediate delegation from B appears valid even though the originating authority has been revoked.

---

## AV-06 — Capability Boundary Laundering

### Setup

The workflow is:

**P → Agent A → Agent B → Skill S → Tool T → Protected Resource R**

Agent B is not authorized to directly perform privileged action X against R.

B is permitted to invoke Skill S.

Skill S can invoke Tool T.

Tool T technically possesses sufficient privilege to perform X against R.

### Attack

B invokes allowed Skill S.

S invokes Tool T.

T attempts privileged action X against R.

Each local interaction may appear permitted:

- B may invoke S;
- S may invoke T;
- T technically possesses access to R.

However, the originating task does not authorize X.

### Expected Result

**DENY**

### Primary Invariant

INV-05 — Capability-Boundary Non-Amplification

### Supporting Invariants

INV-01 — Authority Non-Amplification  
INV-02 — Authority Provenance and Temporal Continuity  
INV-03 — Constraint Inheritance / Narrow-Only Delegation

### Failure Condition

Effective authority remains apparently constrained at the agent layer but expands through a downstream skill or tool boundary.

### Security Significance

This vector tests whether effective authority is enforced end-to-end rather than solely at agent-to-agent boundaries.

---

## AV-07 — Delegation Depth / Loop Violation

### Setup

The implementation permits a maximum of three agent-to-agent delegation edges.

The following delegation chain is valid:

**A → B → C → D**

### Attack

D attempts to delegate the same derived authority to E, creating a fourth agent-to-agent delegation edge and exceeding the configured delegation-depth limit.

Alternatively, a delegation attempts to introduce a repeated chain element such as:

**A → B → C → B**

### Expected Result

**DENY**

### Invariant

INV-01 — Authority Non-Amplification / Delegation Depth and Loop Safety

### Required Evidence

The decision record SHOULD identify the delegation chain, configured depth limit or detected loop, violated invariant, and resulting DENY decision.

### Failure Condition

The implementation permits an arbitrarily deep or cyclic delegation chain merely because each individual delegation remains within the originating authority ceiling.

## AV-08 — Temporal Delegation Replay

### Setup

Agent A previously issued a valid delegation to Agent B for Task T.

The delegation was accepted and used within its legitimate execution sequence.

### Attack

A prior delegation record is replayed, reordered, or reused such that its temporal relationship to the current task acceptance or authority exercise is no longer valid.

For example, the presented evidence would imply:

`TASK_ACCEPT.timestamp < DELEGATE.timestamp`

or relies on delegation state that is stale or no longer valid for the current execution.

### Expected Result

**DENY / RE-AUTHORIZE**

### Invariant

INV-02 — Authority Provenance and Temporal Continuity

### Required Evidence

The decision record SHOULD identify the relevant delegation event, task acceptance or authority-exercise event, temporal inconsistency, violated invariant, and resulting decision.

### Failure Condition

The implementation treats valid historical lineage as sufficient authority without validating whether that lineage is temporally valid for the current execution.


# 6. Candidate Conformance Assertions

The proposed invariants may be translated into testable assertions.

### C-01

A delegatee MUST NOT exercise task authority exceeding the delegator's effective authority for that task.

### C-02

Effective authority MUST remain bounded across every hop of a delegation chain.

### C-03

Applicable authorization constraints MUST propagate downstream unless explicitly narrowed.

### C-04

A downstream component MUST NOT silently remove or weaken inherited authorization constraints.

### C-05

Revocation MUST be evaluated at the relevant enforcement boundary before unexercised downstream authority derived solely from the revoked authorization is exercised.

### C-06

Security-sensitive actions SHOULD retain sufficient provenance to reconstruct the originating principal, task/intent, authorization state, delegation path, executing agent, downstream capability, and affected resource.

### C-07

Security-sensitive side effects MUST be protected by enforcement that evaluates whether the effective authority required for the action remains valid at the relevant enforcement boundary.

### C-08

Selection of a more privileged agent, skill, tool, or API MUST NOT by itself increase the effective authority of the originating task.

---

# 7. Relationship to the Current A2A Authorization Model

This proposal is intended to complement, rather than redefine, the current A2A authorization model.

A2A already establishes important concepts and security mechanisms for agent interoperability, including authenticated interactions, authorization, Agent Cards, Tasks, contexts, skills, protocol extensions, and enterprise security practices.

The protocol also supports collaborative agent interactions and task delegation.

This proposal focuses on a narrower runtime-security question:

> **After authorization has been established, what security properties should remain true when execution and authority cross multiple delegation and capability boundaries?**

The relevant path may extend beyond the A2A interaction itself:

**Principal → Agent A → Agent B → Skill → Tool/API → Protected Resource**

The proposed invariants therefore do not replace A2A authentication or authorization.

Instead, they define candidate runtime properties for preserving the security meaning of authorization across delegated execution:

1. authority does not silently amplify;
2. constraints propagate downstream and may only be narrowed;
3. authority provenance and valid temporal ordering remain continuous;
4. dependent downstream authority responds to upstream revocation; and
5. downstream capabilities cannot be used to obtain effective authority unavailable to the originating task.

The appropriate realization may ultimately be through an A2A extension, ERDL, implementation-specific authorization semantics, verification vectors, security guidance, or another mechanism.

The conformance vectors intentionally remain implementation-neutral so that the security properties can be evaluated independently from the mechanism chosen to enforce them.

---

# 8. Design Questions for Co-Review

## Q1 — Effective Authority Representation

What minimum state must accompany, or remain associated with, a delegated task so that effective authority can be evaluated deterministically?

Potential representations include:

- an authority envelope;
- inherited constraints;
- authorization-chain state;
- capability state; or
- another representation.

---

## Q2 — Standing Privilege vs. Delegated Authority

If a delegatee independently possesses broader authority, under what circumstances may that standing authority legitimately become available to the delegated task?

If re-authorization is permitted, the system should clearly distinguish:

**legitimate new authorization**

from:

**silent authority amplification**

---

## Q3 — Constraint Composition

Where authority is derived from multiple sources, how should constraints compose?

A conservative default would be intersection or narrowing rather than union or expansion.

---

## Q4 — Revocation Semantics

How should transitive revocation operate across:

- active actions;
- pending actions;
- queued actions;
- cached authorization decisions;
- disconnected agents; and
- long-running Tasks?

---

## Q5 — Capability Boundary

Should authority conformance terminate at the receiving agent?

Or should verification extend through:

**Agent → Skill → Tool/API → Protected Resource**

where the actual security-sensitive effect occurs?

---

## Q6 — Unavailable Authority State

What should occur when current upstream authority or revocation state cannot be established?

For high-impact operations, should the implementation fail closed rather than assume that previously valid delegated authority remains valid?

---
## Q7 — Revocation Freshness

What freshness guarantee is sufficient for revocation evaluation at an enforcement boundary when delegation spans independently operated or temporarily disconnected agents?

---

## Q8 — Capability Boundary Semantics

Should Agent → Skill → Tool/API → Protected Resource transitions be represented as another delegation axis governed by the same authority invariants, or as a distinct capability-boundary relationship with equivalent non-amplification semantics?

# 9. Conformance Matrix

| Invariant | Primary Vectors | Expected Decision | Minimum Verifiable Evidence |
|---|---|---|---|
| INV-01 — Authority Non-Amplification | AV-01, AV-02, AV-03, AV-07 | DENY | Effective-authority ceiling, delegation chain, depth/loop state |
| INV-02 — Provenance and Temporal Continuity | AV-02, AV-08 | DENY / RE-AUTHORIZE | Authority lineage, event ordering, authorization state |
| INV-03 — Narrow-Only Delegation | AV-04 | DENY | Inherited constraints and attempted downstream constraints |
| INV-04 — Enforcement-Boundary Revocation | AV-05 | DENY | Revoked ancestor, derived authority lineage, boundary decision |
| INV-05 — Capability-Boundary Non-Amplification | AV-06 | DENY | Task authority, downstream capability, requested effect, boundary decision |

# 10. Proposed Verification Method

Each invariant can be associated with one or more negative verification vectors.

A simple methodology is:

1. Construct a valid baseline authorization and delegation chain.
2. Introduce exactly one controlled authority violation.
3. Request the protected action.
4. Require deterministic rejection.
5. Record the violated invariant and relevant authority provenance.
6. Confirm that the unauthorized side effect did not occur.

### Attributable Rejection Requirement

Rejection alone is insufficient evidence of conformance.

A conforming implementation MUST produce an attributable decision indicating why the vector was rejected.

At minimum, the decision evidence SHOULD identify:

- the resulting decision;
- the matched or violated invariant;
- the originating task or authorization context;
- the relevant authority provenance;
- the enforcement boundary at which the violation was detected; and
- a machine-readable or otherwise deterministic reason for the rejection.

For example:

    decision: DENY
    matched_invariant: INV-05
    originating_task: T1
    boundary: Agent-B → Skill-S → Tool-T
    requested_action: delete(Resource-X)
    effective_authority: read(Resource-X)
    reason: CAPABILITY_BOUNDARY_AUTHORITY_AMPLIFICATION

An implementation that indiscriminately denies all operations MUST NOT be considered conformant merely because every negative vector results in DENY.

Conformance therefore requires both:

1. the expected security decision; and
2. evidence attributing that decision to the authority property exercised by the vector.
The negative vector then acts as a security canary.

A regression exists if a later implementation permits a vector that a conforming implementation previously rejected.

This makes the proposal usable both as an architectural security model and as a basis for executable security verification.

---

# 11. Conclusion

Multi-agent authorization must preserve security semantics beyond initial authentication and access control.

As execution moves through:

**Principal → Agent → Agent → Skill → Tool/API → Protected Resource**

the effective authority associated with the originating task should remain bounded, attributable, constrained, and revocable.

The proposed model centers on five invariants:

1. **Authority Non-Amplification**
2. **Authority Provenance and Temporal Continuity**
3. **Constraint Inheritance / Narrow-Only Delegation**
4. **Enforcement-Boundary Revocation**
5. **Capability-Boundary Non-Amplification**

The eight adversarial conformance vectors test whether these properties survive realistic multi-hop execution and whether rejection can be attributed to the violated authority property.

The core principle is:

> **Delegation may transfer or narrow authority, but it must not silently create authority that the originating task did not possess.**

This draft is submitted for technical co-review and refinement against the ERDL delegation model, A2A authorization architecture, and the proposed verification-vector methodology.

---

## References

1. Agent2Agent (A2A) Protocol Specification — A2A Project.
2. A2A Discussion #2031 — “RFC: Extending A2A Agent Cards with behavioral rules (ERDL).”
3. A2A Enterprise-Ready guidance.
