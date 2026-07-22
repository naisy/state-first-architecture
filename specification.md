# State-First Architecture (SFA) Technical Specification

## 1. Purpose and Scope

This document is the normative specification that defines the common concepts and execution semantics of State-First Architecture (SFA).

SFA is a design approach that models system flow control as explicit `State`, `Event`, `Context`, `Transition`, `Guard`, `Action`, and `Role` boundaries, and does not allow hidden control logic outside the state transition model.

This document specifies:

- the components of a State machine
- the rules for applying Events to Transitions
- the execution order of a Transition
- termination conditions for AUTO transitions
- Projection from external input to Canonical Events
- Role boundaries and runtime ownership
- Event outcomes and failure handling
- common principles for the State machine lifecycle
- the scope of static and runtime verification

This document does not prescribe application-specific screens, key bindings, persistence formats, communication protocols, game rules, or business rules. Those belong to the domain specification of each system that uses SFA.

### 1.1 Normative Terms

The key words in this document are to be interpreted as follows:

- **MUST**: required for SFA conformance
- **MUST NOT**: prohibited for SFA conformance
- **SHOULD**: recommended; deviation is permitted only when there is a reasonable justification
- **MAY**: optional

---

## 2. Core Principles of SFA

### 2.1 Explicit State

State that affects control decisions MUST be represented explicitly as State or Context.

Booleans, sequence values, timeout waits, ownership, pending status, and similar control information MUST NOT be retained as implicit variables that cannot be observed from the state transition model.

Local calculations, temporary variables, and rendering-only values do not all need to become State. SFA concerns control state that affects system flow, acceptable Events, exclusivity, ownership, retries, and completion decisions.

### 2.2 Canonical Event

Events accepted by a State machine MUST be defined as Canonical Events independent of the original input source.

Raw input such as keyboard input, HTTP responses, messages, timers, Worker notifications, and AI decisions is converted into Canonical Events through Projection.

### 2.3 Declared Transition

State changes MUST occur through declared Transitions.

A Role, UI, Worker, Controller, or Action MUST NOT bypass the defined flow by directly jumping across control States without a Transition.

### 2.4 Pure Guard / Controlled Action

A Guard evaluates whether a Transition may be applied using the current State, Event, and explicit Context as input. A Guard MUST NOT modify Context.

An Action is responsible for Context changes or external side effects associated with a selected Transition. The scope that an Action may modify MUST be explicit.

### 2.5 Explicit Role and Ownership

Every State MUST identify the Role that owns it.

For a process spanning multiple steps, or a process in which multiple control actors compete, runtime ownership, preemption rules, completion conditions, invalidation conditions, and release conditions MUST be explicit.

### 2.6 Observable Outcome

The result of Event handling MUST be observable. An implementation SHOULD distinguish application, discard, rejection, unhandled input, and failure rather than representing all of them as “nothing happened.”

---

## 3. Core Metamodel

### 3.1 State Machine Instance

A State machine instance contains at least the following:

```text
MachineInstance = {
  current_state,
  context,
  definition,
  role,
  lifecycle_identity
}
```

- `current_state`: the current State ID
- `context`: explicit data read by Guards and updated by Actions
- `definition`: State and Transition definitions
- `role`: the owner of the instance or current State
- `lifecycle_identity`: an ID or equivalent boundary that identifies the instance

### 3.2 State Definition

A State definition MUST contain at least the following:

```json
{
  "state_id": "FE_IDLE",
  "role": "frontend",
  "description": "A stable state that can accept input",
  "on_enter": null,
  "on_exit": null,
  "next_transitions": []
}
```

`on_enter` and `on_exit` are optional. When used, their execution order and failure behavior MUST follow Section 4.3.

### 3.3 Transition Definition

A Transition contains at least the following:

```json
{
  "event": "SUBMIT",
  "guard": "can_submit",
  "action": "start_submission",
  "target_state": "FE_WAITING_BE",
  "outcome_policy": "REJECT_IF_GUARD_FALSE"
}
```

- `event`: the target Canonical Event
- `guard`: a pure function that evaluates whether the Transition may be applied; when omitted, it is treated as always valid
- `action`: an optional function that changes Context or performs a side effect
- `target_state`: the destination State
- `outcome_policy`: the Event outcome policy when the Transition is not applicable; optional, but SHOULD be explicit

### 3.4 Context

Context is not merely an unstructured data store. Values used for control decisions SHOULD define their name, meaning, owning Role, lifecycle, and restoration policy.

Invariants for entity identity, version, sequence, checkpoint, and similar data MAY be specified as contracts separate from the State machine definition.

### 3.5 Scope of the Definition Set as the SSOT

In SFA, the trusted specification source is not the State JSON alone. It is an internally consistent definition set containing at least:

- State catalog
- Event catalog
- Transition definitions
- Guard registry and contracts
- Action registry and contracts
- Projection policy
- Role and ownership policy
- Lifecycle policy
- Invariant definitions
- Contract tests or equivalent verification assets

Even when some of these elements are implemented directly in code, they MUST be mutually traceable.

---

## 4. Engine Execution Semantics

### 4.1 Event Acceptance

The SFA engine applies a Canonical Event to the current MachineInstance.

An Event SHOULD contain at least the following information:

```text
Event = {
  type,
  payload,
  source_role,
  correlation_id?,
  metadata?
}
```

A `correlation_id` or timestamp MAY be used for domain requirements or observability. However, the basic decision about whether an Event is acceptable in the current State MUST NOT depend only on hidden chronological comparisons.

### 4.2 Ordered Transition Evaluation

When multiple Transitions exist for the same Event, they are evaluated in declaration order as priority order.

1. Obtain, in declaration order, all Transitions in `current_state` whose Event matches `event.type`.
2. Evaluate each Guard in order.
3. Select the first candidate whose Guard succeeds.
4. Do not evaluate later candidates after one has been selected.
5. If no candidate succeeds, return `DISCARDED`, `REJECTED`, or `UNHANDLED` according to the outcome policy.

Because Guard evaluation order is semantically significant, reordering definitions MUST be treated as a behavioral change.

### 4.3 Transition Execution Order

The standard execution order is:

1. finalize the selected Transition
2. execute the source State's `on_exit`
3. execute the Transition Action
4. update `current_state` to the target State
5. execute the target State's `on_enter`
6. record the `APPLIED` outcome
7. evaluate AUTO Transitions in the target State

If an implementation uses a different order, it MUST document that order and fix it with contract tests.

### 4.4 Action Failure and Atomicity

When an Action fails, a partially applied State or Context change MUST NOT be treated as successful.

The implementation MUST explicitly use one of the following approaches:

- **Atomic rollback**: restore Context and State to their pre-Transition values and return `FAILED`
- **Failure transition**: record the failure in Context and move to a declared failure State
- **Compensating action**: perform a compensating Action and then move to a failure State

When an external side effect cannot be fully rolled back, that fact and the retry policy MUST be explicit.

### 4.5 AUTO Transition

`AUTO` is a transient Event evaluated without waiting for external input.

AUTO evaluation MAY continue until a stable State is reached, but the implementation MUST provide:

- a maximum number of AUTO Transitions within one dispatch
- cycle detection based on State or a State+Context signature
- `FAILED` or a dedicated outcome when the limit or a cycle is detected
- a termination rule for AUTO Action failure

Unbounded recursion MUST NOT be part of the specification.

---

## 5. Projection

### 5.1 Definition

Projection is the boundary that converts raw input into a Canonical Event.

```text
Raw Input -> Parse -> Validate -> Project -> Canonical Event
```

Projection prevents source-specific input formats from leaking into the State machine.

### 5.2 Convergence of Multiple Input Sources

When user input, automated control, an API, and a Worker request the same semantic operation, they SHOULD project to the same Canonical Event.

This allows the same Transition, Guard, and Action to be reused without duplicating domain behavior for each input source.

### 5.3 Responsibilities of Projection

Projection MAY perform:

- parsing of raw input
- schema validation
- normalization of Event payloads
- assignment of the source Role
- conversion into an Event type that can be dispatched to the current State

Projection MUST NOT bypass Transitions and modify Context directly.

---

## 6. Event Outcome

### 6.1 Standard Outcomes

An SFA engine SHOULD distinguish at least the following outcomes:

- `APPLIED`: a Transition was selected and applied
- `DISCARDED`: the Event is intentionally ignored in the current State
- `REJECTED`: the Event was recognized but rejected by a Guard, ownership rule, precondition, or policy
- `UNHANDLED`: no Event definition or Transition exists
- `FAILED`: failure occurred after selection, including in an Action, `on_exit`, `on_enter`, or AUTO processing

### 6.2 Difference Between Discard and Reject

An additional click during rapid repeated input, or stale UI input after a process has already completed, MAY be `DISCARDED`.

Identity inconsistency, insufficient authorization, ownership conflict, and unsatisfied mandatory preconditions SHOULD be `REJECTED` because the caller or operator needs to know about them.

An Event that may indicate a missing definition SHOULD be `UNHANDLED` rather than silently discarded.

### 6.3 Observability

An Outcome SHOULD be associated with the current State, Event, selected Transition, Guard result, Action, target State, and reason.

---

## 7. Roles and Boundary States

### 7.1 Role Ownership

Every State MUST have an owning Role. Examples include `frontend`, `backend`, `worker`, `device`, and `shared`.

A Role MUST NOT directly modify a State owned by another Role. Progress across Role boundaries is expressed through Events and Transitions.

### 7.2 Boundary State

For asynchronous processing or communication between Roles, waiting, pending application, pending cancellation, and similar Boundary States SHOULD be explicit.

A Boundary State restricts acceptable Events and can suppress duplicate submissions and conflicting requests through State structure.

### 7.3 Delayed Events

If a delayed Event arrives when the current State has no corresponding Transition, the Event is discarded or rejected according to the outcome policy.

However, when different requests share the same State and Event type, State mismatch alone cannot distinguish an old Event. When necessary, explicit request identity, generation, correlation, or equivalent data is used as part of Context and the Guard contract.

SFA does not prohibit timestamps or sequences themselves. It avoids hiding them outside the state transition model or using them as the sole basis of control.

---

## 8. Runtime Ownership and Preemption

### 8.1 Applicability

A process completed by a single Transition does not require an additional ownership model.

When a process spans multiple Events or multiple turns, or when multiple Policies control the same MachineInstance, runtime ownership MUST be explicit.

### 8.2 Ownership Contract

An ownership contract defines at least:

- owner ID or owner type
- ownership start condition
- Events the owner may emit
- whether another Policy may preempt the owner
- completion condition
- invalidation condition
- failure condition
- the next State or reevaluation policy after release

### 8.3 Preemption

Preemption MUST NOT occur implicitly. When preemption is permitted, the preempting Event, Guard, cleanup Action, and next State are declared as a Transition.

---

## 9. Lifecycle

### 9.1 Scope

Lifecycle covers the creation, start, retention, pause, restoration, termination, and disposal of a State machine instance and its Context.

### 9.2 Lifecycle Boundaries

An implementation SHOULD make the following explicit:

- the unit of MachineInstance creation
- when Context is initialized
- which State is retained across boundaries such as a map, screen, session, or job
- when transient plans or failure memory are discarded
- what information is retained across pause and resume
- which State, Context, and ownership information are included in a checkpoint
- schema validation and invariant validation during restore

Specific keys and persistence mechanisms are domain specifications. SFA requires restored State and Context to belong to the same logical checkpoint and prohibits accepting contradictory combinations.

### 9.3 Reset and Resume

Reset, Resume, and Retry SHOULD be defined as dedicated Events and Transitions or as explicit lifecycle operations, rather than as ambiguous direct assignment.

---

## 10. Invariants and Identity

The correctness of a State machine cannot be guaranteed by the Transition graph alone.

An implementation defines the following invariants when necessary:

- uniqueness of entity identity
- monotonicity of sequences
- consistency of owning Roles
- existence of references inside Context
- consistency of State and Context combinations
- preconditions and postconditions of a transaction
- resource limits

When an invariant is violated, the implementation SHOULD expose the violation as `REJECTED` or `FAILED` rather than continuing with an ambiguous Action.

---

## 11. Verifiability

### 11.1 What Static Analysis Can Verify

When definitions are structured, static analysis can detect:

- undefined target States
- unreachable States
- unintended terminal States
- inconsistencies between the Event catalog and Transitions
- broken Guard or Action registry references
- Role boundary violations
- some AUTO cycles
- Transitions made unreachable by priority order

### 11.2 What Static Analysis Alone Cannot Guarantee

The following require runtime tests, contract tests, property tests, simulation, or equivalent methods:

- the implementation of Guards and Actions
- success of external side effects
- identity and transaction consistency
- performance, memory, and timeouts
- physical calculations, numerical calculations, and AI search results
- lifecycle defects that depend on real data

SFA does not claim that every bug can be guaranteed away through static analysis alone.

### 11.3 Recommended Contract Tests

- Outcome for each State × Event combination
- A/B tests for Guard success and failure
- Transition execution order
- State and Context after Action failure
- AUTO limits and cycle detection
- Role boundaries
- ownership start, completion, invalidation, and preemption
- checkpoint and restore consistency
- replay of the same fixture in systems that support deterministic replay

---

## 12. Trace and Explainability

An SFA implementation SHOULD be able to output a Transition trace.

```json
{
  "from": "FE_READY",
  "event": "SUBMIT",
  "guard": "can_submit",
  "action": "start_submission",
  "to": "FE_WAITING_BE",
  "outcome": "APPLIED"
}
```

On failure, it is desirable to trace the State, Event, Guard, Action, Role, ownership, and reason.

A Trace is not a substitute for the specification. It is an observability asset used to verify that definitions and execution results agree.

---

## 13. Reference Metamodel

The following is a reference schema. It does not require a specific implementation language or storage format.

```json
{
  "state_id": "FE_READY",
  "role": "frontend",
  "description": "Can accept submit",
  "on_enter": null,
  "on_exit": null,
  "next_transitions": [
    {
      "event": "SUBMIT",
      "guard": "can_submit",
      "action": "start_submission",
      "target_state": "FE_WAITING_BE",
      "outcome_policy": "REJECT_IF_GUARD_FALSE"
    }
  ]
}
```

Example of Event projection:

```json
{
  "raw_source": "button.click",
  "projector": "project_submit_click",
  "canonical_event": "SUBMIT"
}
```

Example of an ownership policy:

```json
{
  "owner": "UPLOAD_SESSION",
  "starts_on": "UPLOAD_ACCEPTED",
  "allowed_events": ["CHUNK_READY", "CANCEL", "UPLOAD_FAILED"],
  "preemptible_by": ["CANCEL"],
  "completes_on": "UPLOAD_COMPLETED",
  "invalidates_on": ["SESSION_EXPIRED", "SOURCE_CHANGED"]
}
```

---

## 14. Conformance Levels

### 14.1 Core SFA

An implementation conforms to Core SFA when it provides:

- Explicit State
- Canonical Event
- Declared Transition
- Pure Guard / Controlled Action
- a defined Transition evaluation order
- observable Event outcomes

### 14.2 Distributed SFA

In addition to Core SFA, a Distributed SFA implementation provides:

- Role ownership
- Boundary States
- inter-Role Events
- request identity or an equivalent asynchronous consistency contract

### 14.3 Long-Running SFA

In addition to Core SFA, a Long-Running SFA implementation provides:

- runtime ownership
- completion, invalidation, and preemption rules
- a lifecycle policy
- a consistency contract when checkpoints or resume are used

Conformance levels do not indicate superiority. They describe the scope appropriate to the complexity of the system.

---

## 15. Non-Normative Application Examples

Games, Web UIs, distributed processing systems, and device-control systems may all be application examples of SFA.

For example, a design in which manual input and an automated Controller project to the same Canonical Event and share an existing Transition demonstrates the benefit of Projection. However, concrete Event names, keys, screens, persistence methods, and domain rules are not part of the common SFA specification.

Application examples are used to explain and verify the common specification. Application-specific rules MUST NOT be introduced into the normative specification without first being abstracted into a general principle.
