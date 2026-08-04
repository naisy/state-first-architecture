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

#### 3.4.1 Provenance

For every Context item used in a control decision, the implementation MUST declare **how the value was produced (its provenance)**.

A provenance declaration contains at least:

- **origin**: the kind of source (observation, a decision made by another machine, external input, derived computation, or configuration)
- **producer**: the place that actually produces the value (a function, module, machine, or external boundary)
- **freshness**: the point in time the value represents (determined within the same Transition, the most recent observation, refreshed periodically, or fixed at startup)

Context items that carry the same meaning MUST NOT be computed independently in two or more places. When several consumers need the value, the value produced by the single declared producer is passed to them.

A Guard MUST be pure (Section 2.4), but **purity does not imply correctness**. If the value handed to a Guard is produced incorrectly, the decision is wrong even when the predicate is right. Provenance therefore MUST be treated with the same rigor as the Guard predicate itself.

Guard, Action, and Projection implementations MUST NOT produce a Context item through any path other than its declared producer. When an implementation turns out to need a newly derived value for a decision, the Context item and its provenance are added to the definition first.

> Incorrect decisions appear more often in how the value handed to a predicate was produced than in the predicate itself. Declaring provenance makes that production path verifiable, and structurally forbids computing the same value twice — the situation in which only one of the two copies gets corrected.

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

### 3.6 Guard Contract and Validity Range

The Guard registry contract (Section 3.5) is not satisfied by the predicate name and the meaning of its truth value alone. When a Guard contains an **approximation**, that approximation MUST be declared.

An approximation is any means of reaching a decision with finite resources by examining only part of the subject, or by representing the subject with a simplified model. Examples include a bounded lookahead depth, sampling, reuse of a cached value, a search with an upper bound, and aggregation into a single representative value.

An approximation declaration contains at least:

- **approximated**: what was approximated (the subject that should be examined, and the part actually examined)
- **valid_when**: the condition under which the approximation is sound
- **breaks_when**: the condition under which the approximation fails; this MUST be written **as a property of the subject**, not as a likelihood
- **on_break**: what happens when it fails (admits incorrectly, refuses incorrectly, or cannot decide)

A Guard whose approximation fails toward **admitting incorrectly** MUST NOT be used for a safety-related decision. For safety-related decisions, an approximation MUST be designed to fail toward **refusing incorrectly**.

The condition declared in `breaks_when` MUST actually be exercised as a negative control during verification (Section 11.3).

> The judgment "this range is sufficient for the purpose of the decision" is itself capable of being wrong. If the approximation is not declared, that judgment is never reviewed by anyone. Once the breaking condition is written as a property of the subject, whether that condition actually holds can be checked against the subject's real data.

### 3.7 Responsibility Boundaries (Judgements Not Owned)

The responsibility of a State machine is not determined by enumerating **what it owns** alone. When it is part of the design that a given judgement is **not made by that machine**, that fact reaches the next implementer only if the definition states it.

In a system where judgements are divided across several machines, the implementation SHOULD declare **the judgements that machine MUST NOT make** and **where each of those judgements is made**.

The declaration contains at least:

- **judgement**: the judgement that must not be made here (what is not decided)
- **owned_by**: the machine that makes it, and the identifier of the invariant that grounds it
- **rationale**: why it must not be made here

When the same judgement is made in two or more places, **the definition no longer determines which one is authoritative**. Role ownership (Section 7) governs the ownership of State and does not express "the judgement this machine must not make". The two are separate declarations.

A declaration of judgements not owned SHOULD be written in a form that can be checked statically: that the implementation of that machine does not reference the predicates or inputs corresponding to the declared judgement.

> A judgement will be placed wherever there is a place to put it. The incentive — "it would be quicker to count it here", "it would be quicker to decide it here" — arises on every implementation. Declaring what is not owned gives the definition an answer to that incentive. Without the declaration, the same rule is copied into several places and diverges over time.

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

#### 4.2.1 Co-Satisfiable Candidates

By step 4, candidates that were not selected are **neither evaluated nor observed**. When candidates are mutually exclusive, this is only a matter of efficiency. But when **two or more candidates for the same Event can hold at once**, only the selected candidate is recorded, and **the fact that another trigger also held leaves no trace**.

When multiple candidates for the same Event can hold at once, those candidates MUST declare that they are co-satisfiable.

The declaration contains at least:

- **co_satisfiable_with**: the identifiers of the other candidates that can hold at the same time
- **why_one_is_chosen**: why selecting a single candidate is nevertheless correct (identical effect, a precedence fixed by the domain, and so on)
- **must_record**: those unselected candidates whose having held MUST be recorded

For declared candidates, the engine or the Action MUST be able to retain **the set of candidates that held** in the outcome (Sections 6.1 and 12). Only the record is added; evaluation order and the meaning of selection are unchanged.

Multiple triggers with the same effect MUST NOT be recorded under the name of the first matching candidate alone. When the record collapses to one, later analysis cannot distinguish "the other trigger did not hold" from "the other trigger held but was not recorded".

> This gap is not a coding habit; it is a consequence of ordered evaluation. The specification elsewhere requires a **forensic minimum** (Section 10.2). Declaration and recording are required so that the execution semantics do not create a state that the specification demands be traceable. Where candidates cannot hold at once, no declaration is needed and this subsection imposes nothing.

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

When co-satisfiable candidates are declared (Section 4.2.1), the outcome MUST be able to carry **the set of candidates that held in that cycle**. The standard outcomes describe the result of the single selected Transition; they **do not express how many triggers held**. The two MUST NOT be collapsed into the same field.

### 6.4 Aggregation Keys and Their Meaning

When outcomes or events are aggregated, the implementation SHOULD declare **what each aggregation key counts**.

- A key's name may denote only a **subset** of what is actually being counted
- Several events that increase for **different reasons** may be merged into a single key

In either case, a reader of the key's value cannot determine what they are looking at. When a key is renamed or merged, the implementation SHOULD declare **its correspondence to the previous key**. Without that correspondence, a judgement that compares values across the change fails silently.

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

### 10.1 Invariant Scope and Participants

Invariants divide into those contained within a single instance and those that **span multiple instances**. The latter — mutual exclusion, aggregate limits, ordering relations — cannot be expressed in a per-instance definition.

An invariant that spans multiple instances MUST declare:

- **scope**: `instance` or `cross_instance`
- **participants**: how the set of participating instances is determined (the predicate or index used for evaluation)
- **evaluator**: who performs the check (not an instance, but an actor that can observe the set)
- **cadence**: when the check runs (per Transition, periodically, or at checkpoints)

When a `cross_instance` invariant violation is recorded, the record MUST include **the Context of every participant**. The state of the single instance that happened to detect the violation is not sufficient to determine afterwards whether the violation came from a resource being held twice or from an inconsistency in observation.

### 10.2 Invariant Diagnosability (Forensic Minimum)

For every invariant, the implementation MUST declare the **minimum set of observations required to trace the cause once that invariant is violated**. This is referred to below as the **forensic minimum**.

The declaration contains at least:

- which Events, outcomes, or Context items must remain in order to trace the cause
- the extent (time, instances, resources) over which they must be retained

An implementation that adopts a policy for reducing the volume of observation (Section 12) MUST NOT drop observations listed in the forensic minimum. The policy and the minimum MUST be declared side by side, and **a conflict between them MUST be treated as an error in the definition**.

> A policy for reducing observation volume is a discipline against recording too much; it is not a discipline against recording too little. Without the counterpart, a decision that carries responsibility for an invariant can enter operation without carrying any record of that decision. Declaring the minimum in advance makes this conflict detectable before operation.

The forensic minimum is meaningful **only when the invariant is actually violated**, and is therefore decided independently of the volume of observation during normal operation. Even where aggregate counters are sufficient in normal operation, anything listed in the minimum MUST be recordable individually **at the moment a violation is detected**.

### 10.3 Preconditions for Invariant Validity

An invariant depends on the meaning of the Context items it references. When that meaning varies with configuration — settings, feature switches, operating modes, degraded states — the implementation MUST declare **the precondition under which the invariant holds**.

The declaration contains at least:

- **holds_when**: the configuration under which the invariant holds
- **out_of_scope_when**: the configurations for which validity is not claimed, and what is guaranteed instead in those configurations

A violation observed in a configuration that does not satisfy the precondition MUST NOT be counted as an invariant violation. It MUST be distinguishable as out of scope. Without that distinction, genuine violations are buried in the out-of-scope count.

The same dependency on configuration also appears in a Guard's enforcement mode (enforced versus observed only). Invariants and Guards SHOULD express such preconditions in the same form.

> If the definition does not allow one to decide whether an invariant is broken or simply not claimed for the current configuration, an observed violation has no determinate meaning. A change that alters the meaning of a Context item also requires reviewing the invariants that reference it; recording the precondition makes those invariants traceable from the definition.

### 10.4 Progress Invariants

Every invariant listed in Section 10 states **something that must not happen**. A system can satisfy all of them and still **fail to move forward**. A state that holds only defined Transitions, and remains there because none of them fires, violates no invariant.

Progress is not determined by the Transition graph of a single machine, because **the Event that leaves a state may be one that the machine cannot produce itself**. Progress therefore MUST be treated as a separate subject of declaration and verification.

#### 10.4.1 Declaring Situations That Can Stall

A **situation that can stall** is a state or condition which, once entered, cannot be left unless something external occurs.

Such a situation MUST declare:

- **exits**: the means of leaving it; at least one
- **progress_measure**: the quantity that expresses whether progress is occurring (Section 10.4.3)
- **if_no_exit**: what happens when none of the means takes effect

The place of declaration is **not restricted to States**. In a design where a machine remains under the same condition without changing state, what creates the stall is **the Guard that keeps the Transition from succeeding**. In that case the declaration MUST be placed on the Guard. If States are fixed as the only place of declaration, this form of stall cannot be expressed.

#### 10.4.2 Providers of Exits, and the Condition for Self-Help

Each entry in `exits` MUST declare **the party that makes that means succeed**.

- **provider**: `self` (it can be made to succeed by the machine itself), another Role, or **another instance of the same machine**
- **waits_for**: when the provider is not `self`, the situation of the party being waited on
- **requires_change_in**: the input that must change for the means to succeed, and the owner of that input

Even when `provider` is `self`, if the owner named in `requires_change_in` is not the machine itself, the means is not self-help but **a dependency on another party**. **Repeating the same computation over the same inputs is not an exit.** When a retry is declared as an exit, the declaration MUST state **why the result will differ next time**.

When a self-help means requires acquiring a resource, the implementation SHOULD declare **who can hold that resource**. If the resource can be held by a party in the dependency relation, that means does not succeed in that situation.

#### 10.4.3 The Progress Measure

A `progress_measure` MUST declare:

- **quantity**: the quantity that expresses whether progress is occurring
- **advances_when**: the condition under which the quantity advances
- **resets_on**: the condition under which the quantity returns to its starting point

`resets_on` MUST NOT include **operations the system itself performed in order to make progress**. In a design where the measure resets on each remedy or retry, **the more action is taken, the further away any mechanism triggered by that measure becomes**.

A measure SHOULD be derived from **progress itself**. Deriving it from an internal classification or working state allows the measure to move while the externally observable situation does not change.

#### 10.4.4 Cycles in the Dependency Graph

The `waits_for` declarations form **a finite graph whose nodes are the situations that can stall**. This graph can be checked statically.

- when the graph contains a cycle, that cycle MUST contain at least one self-help means **that can succeed within that cycle**
- if every node of a cycle only waits on another party, the system **does not progress** once the cycle is entered
- when a self-help means requires a resource that can be held by a node of the same cycle, that means MUST be counted as **not succeeding for that cycle**

Cycles themselves are not prohibited. What is required is **the ability to distinguish a cycle that can be broken from one that cannot**.

#### 10.4.5 Reachability of Exits

The presence of an `exit` in a definition **does not mean the means is usable**. When another decision closes the entry to that means, a declared means never succeeds even once.

Each entry in `exits` SHOULD carry **the declaration that makes it reachable** (an invariant or an equivalent identifier). When the referenced declaration does not exist, verification MUST treat that means as **one that does not succeed**.

#### 10.4.6 Exits for Machines That Only Detect

A machine that only **detects** an anomaly or a stall MUST declare **to whom the detected fact is handed**.

A state in which detection occurs but no party owns the resolution is the archetypal form of a system that satisfies every invariant and still does not progress.

#### 10.4.7 Attribution of Resolution

When a situation that can stall is left, the implementation SHOULD record **by which means it was left**.

Recording only that it was left makes it impossible to distinguish, after the fact, whether the mechanism took effect or an external circumstance happened to change. The breakdown of attributions is the only material with which the value of holding that mechanism can be measured.

#### 10.4.8 Assumptions Behind Progress

A claim of progress holds only under assumptions about the environment. Assumptions such as a periodic process continuing to run, or input continuing to arrive, SHOULD be declared as **the preconditions of progress**.

When a precondition is written in absolute time, **its meaning changes in an environment whose execution rate differs from the time base of the subject system**. A precondition that involves time MUST state what that time is measured against.

A mechanism triggered by the passage of time — a periodic sweep, the detection of quiescence, the expiry of a holding period — carries the assumption that **the clock measuring that passage advances**. This assumption SHOULD be declared as:

- **which clock measures it** (the clock of the system, a clock provided by the execution environment, an external cadence)
- **the conditions under which that clock advances**

The execution environment can change how this clock advances. In an execution unit placed in the background, one whose resources are constrained, or one running under replay or acceleration, **the same definition does not necessarily advance at the same rate**.

Therefore, when a mechanism triggered by the passage of time is observed in order to judge it, it SHOULD be confirmed **before the judgement** that the clock is advancing at the assumed rate.

An observation that omits this confirmation cannot distinguish **that no progress occurred** from **that the condition for progress did not hold in the observing environment**. The error that appears in this case points in the direction of the mechanism not working, and therefore **creates a motive to change a correct implementation**. Unless the assumption is confirmed first, changing the implementation leaves the symptom unchanged, and whether the change was warranted cannot be judged either.

> Safety (what must not happen) is a property of the state space and can be checked on the Transition graph. Progress (that a state can eventually be left) depends on assumptions about the environment and is therefore not determined within a single machine. This section does not require a **proof** of progress. What it requires is that **no stall for which nobody holds the duty of exit exists in the definition**. That is a check over a finite graph and can be performed before implementation.

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
- Transitions that use a Context item in a decision without a declared provenance (Section 3.4.1)
- places where a Context item is produced by something other than its declared producer (Section 3.4.1)
- Context items with the same meaning being computed in two or more places (Section 3.4.1)
- observations listed in a forensic minimum being dropped by a policy for reducing observation volume (Section 10.2)
- a `cross_instance` invariant that lacks a participants, evaluator, or cadence declaration (Section 10.1)
- a Guard that contains an approximation but lacks a `breaks_when` declaration (Section 3.6)
- an invariant that references configuration-dependent Context items but lacks a validity precondition (Section 10.3)
- multiple candidates for the same Event that can hold at once without a co-satisfiability declaration (Section 4.2.1)
- a situation that can stall but lacks an `exits`, `progress_measure`, or `if_no_exit` declaration (Section 10.4.1)
- an `exits` set whose only autonomous means is the passage of time (Section 10.4.2)
- a `progress_measure` whose `resets_on` includes an operation of the system itself (Section 10.4.3)
- a cycle in the dependency graph with no self-help means that can succeed within that cycle (Section 10.4.4)
- a machine that only detects, without a declaration of where its findings are handed (Section 10.4.6)
- a mechanism triggered by the passage of time, without a declaration of the clock that measures it and the conditions under which that clock advances (Section 10.4.8)
- an implementation that references a judgement its machine declared it does not own (Section 3.7)
- a control on the intent-breaking side that lacks a declaration of the diagnostic it expects (Section 11.1.1)

### 11.1.1 Conditions the Checks Themselves Must Meet

Static analysis MUST NOT **silently drop its subject**.

- it MUST report **the number of elements it examined** and **the forms it could not examine**
- it MUST NOT return success when the subject set is empty; empty means "nothing has been checked yet", not "there is no problem"
- it MUST NOT collapse **the absence of a subject** and **the check's failure to reach its subject** into the same result. When the subject could not be reached, the check MUST report **unreachable**, which is neither success nor failure

When the existence of the subject is determined outside the check — when the subject is a running system, an external surface, or a generated artifact — the check SHOULD declare **how it identified the subject** and **by what independent path it confirmed that the subject exists**.

When absence and unreachability are collapsed, a report of "no subject" is read as **the subject being absent rather than the check being defective**. Such a report is not treated as a failure, and **subjects that were never examined pass alongside those that were**.

When definitions exist in more than one storage format or notation, a check MUST either handle all of them or **state explicitly which forms it did not handle**. A check that looks at only one form passes while overlooking the other.

A check whose subject is the **structure** of an implementation — the position of a call, the number of branches, the order of statements — MUST declare in the check itself **what that structure protects**.

A check that pins structure can fail on a refactoring that preserves intent. Without the declaration, the only remaining option is to weaken the failing check, and **the property that structure protected is lost silently**.

A check that pins structure SHOULD carry controls in both directions:

- **it does not fail on an implementation that changes the structure but preserves the intent**
- **it does fail on an implementation that breaks the intent**

One direction alone does not establish what the check protects.

The control on the intent-breaking side is **not satisfied by the mere fact that it failed**. Such a control MUST declare **the diagnostic that is expected to appear for that particular way of breaking the intent** — the name of the failing item, the identifier of the diagnostic, the kind of failure, or an equivalent. A control that fails for a reason other than the one declared MUST NOT be treated as passing.

A control that does not examine the reason **is also satisfied by a defect in the check itself**. If the procedure that breaks the intent is itself wrong, or if it does not break the subject at all, the condition is still met as long as some failure occurs. The number of controls then grows while **the set of properties actually established does not**.

When several controls **all fail for the same reason**, they do not break the intent independently. The number of controls MUST NOT be used as evidence for the breadth of what was established.

### 11.2 What Static Analysis Alone Cannot Guarantee

The following require runtime tests, contract tests, property tests, simulation, or equivalent methods:

- the implementation of Guards and Actions
- success of external side effects
- identity and transaction consistency
- performance, memory, and timeouts
- physical calculations, numerical calculations, and AI search results
- lifecycle defects that depend on real data
- the soundness of an approximation contained in a Guard (whether its `breaks_when` condition actually holds is checked against the real data of the subject)

SFA does not claim that every bug can be guaranteed away through static analysis alone.

When no engine executes the definitions — that is, when the definitions are used only as a design and verification baseline — **the fact that verification has covered every declared Transition is not evidence that the implementation performs those Transitions**. In that case, verification is placed not on Transition coverage but on the correspondence between Guards, invariants, and the mechanism (where each declared decision is actually made).

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
- **that values are produced as declared by their provenance** (replacing the declared producer changes the decision)
- **that the forensic minimum is satisfied** (violate an invariant deliberately and confirm that the cause can be traced using only the declared observations; if it cannot, the declared minimum is incomplete)
- **that a `cross_instance` invariant violation record contains the Context of every participant**
- **that a Guard's `breaks_when` condition is actually exercised** (create the condition under which the approximation fails and confirm that it fails in the declared `on_break` direction; if the condition cannot be created, the `breaks_when` declaration is wrong)
- **that a violation in a configuration which does not satisfy an invariant's precondition is treated as out of scope rather than as a violation** (Section 10.3)

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

An implementation that adopts a policy for reducing the volume of observation — which Events are aggregated instead of recorded individually, and what is not retained — declares that policy in the definitions. The policy MUST cover **only what may be dropped**. Observations listed in an invariant's forensic minimum (Section 10.2) are outside the scope of the policy, and a conflict between the policy and the minimum MUST be treated as an error in the definition.

When Context used in a decision is included in a Trace, it is desirable that the **provenance** (Section 3.4.1) be traceable as well as the value. The value alone does not distinguish a wrong decision from a decision made on a wrongly produced value.

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

Example of a Context item and its provenance (Section 3.4.1):

```json
{
  "context_item": "pending_chunk_count",
  "meaning": "Number of chunks not yet confirmed",
  "origin": "observation",
  "producer": "upload_progress_reader.read()",
  "freshness": "same_transition",
  "used_by": ["can_finalize"]
}
```

Example of an invariant declaration (Sections 10.1, 10.2, and 10.3):

```json
{
  "invariant_id": "one_writer_per_resource",
  "statement": "At most one instance holds the write right for a given resource",
  "scope": "cross_instance",
  "participants": "All instances holding the same resource_id",
  "evaluator": "resource_registry",
  "cadence": "periodic",
  "holds_when": "Configurations in which the write right is acquired through a single registry",
  "out_of_scope_when": "Configurations in which each instance acquires independently (duplication is only detected there)",
  "forensics": {
    "required_records": [
      "The Event that granted the write right, and to whom and when",
      "The Context of every participant at the moment of detection"
    ],
    "retention": "Wide enough to identify the counterpart around the moment of detection"
  }
}
```

Example of a Guard contract (Section 3.6):

```json
{
  "guard": "can_finalize",
  "predicate": "pending_chunk_count == 0",
  "enforced": { "strict_mode": true, "lenient_mode": "observe_only" },
  "approximation": {
    "approximated": "The full set of chunks should be examined, but only the aggregate at the last observation is examined",
    "valid_when": "No chunk is added during the observation interval",
    "breaks_when": "Configurations in which a new chunk can be added after the observation",
    "on_break": "admits incorrectly"
  }
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

Example of judgements not owned (Section 3.7):

```json
{
  "does_not_own": [
    {
      "judgement": "deciding whether a resource may be released",
      "owned_by": "resource_registry.one_writer_per_resource",
      "rationale": "the release decision is kept in one place on the resource side; re-deciding here puts the rule in two places"
    },
    {
      "judgement": "measuring elapsed time",
      "owned_by": "progress_watch.dwell_is_measured_by_position",
      "rationale": "the definition of the measure is kept in one place; counting here creates a second measure"
    }
  ]
}
```

Example of co-satisfiable candidates (Section 4.2.1):

```json
{
  "event": "RETRY_WINDOW_ELAPSED",
  "guard": "retry_budget_left",
  "action": "discard_and_recompute",
  "target_state": "FE_RECOMPUTING",
  "co_satisfiable_with": ["escalation_selected"],
  "why_one_is_chosen": "both perform the same operation (discard the current result), so one is sufficient",
  "must_record": ["escalation_selected"]
}
```

Example of a situation that can stall (Section 10.4):

```json
{
  "situation": "FE_WAITING_BE",
  "can_stall": true,
  "progress_measure": {
    "quantity": "number of confirmed chunks",
    "advances_when": "a new chunk was confirmed",
    "resets_on": ["session_restarted"]
  },
  "exits": [
    {
      "event": "CHUNK_CONFIRMED",
      "provider": "other_role",
      "of_role": "backend",
      "waits_for": "BE_QUEUED.capacity_available"
    },
    {
      "event": "DISCARD_AND_RECOMPUTE",
      "provider": "self",
      "requires_change_in": { "what": "the set of inputs", "owned_by": "self" },
      "resource": { "what": "the scratch area used for recomputation", "held_by": [] },
      "reachable_because": ["upload_policy.recompute_is_always_permitted"]
    }
  ],
  "if_no_exit": "the session stays valid, makes no progress, and keeps holding its resource"
}
```

Example of a check whose subject is structure (Section 11.1.1):

```json
{
  "check": "resource_release_is_dominated_by_the_registry",
  "target": "every place that calls release",
  "protects": "that the decision to release is made in one place",
  "pinned_structure": "a query to the registry appears before the release call",
  "controls": {
    "refactor_keeps_intent": "does not fail when the query is moved to the function entry",
    "intent_broken": "fails when the query is removed",
    "intent_broken_diagnostic": "release_without_registry_lookup"
  },
  "subject_reach": {
    "identified_by": "enumeration of the places that call release",
    "existence_confirmed_by": "the list of release entry points published by the registry"
  }
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
- a declared provenance for every Context item used in a decision (Section 3.4.1)
- a declared validity range and breaking condition for every Guard that contains an approximation (Section 3.6)
- a declared forensic minimum for every invariant it defines (Section 10.2)
- a declared precondition wherever the validity of an invariant depends on configuration (Section 10.3)
- a co-satisfiability declaration, and an outcome able to retain the candidates that held, wherever multiple candidates for the same Event can hold at once (Sections 4.2.1 and 6.3)
- a declared `exits`, `progress_measure`, and `if_no_exit` for every situation that can stall (Section 10.4.1)
- a declared provider for every entry in `exits`, and a declared `requires_change_in` wherever self-help is claimed (Section 10.4.2)

### 14.2 Distributed SFA

In addition to Core SFA, a Distributed SFA implementation provides:

- Role ownership
- Boundary States
- inter-Role Events
- request identity or an equivalent asynchronous consistency contract
- a declared scope, participants, evaluator, and cadence for every `cross_instance` invariant (Section 10.1)
- a declaration of the judgements not owned and where they are owned, wherever judgements are divided across machines (Section 3.7)
- a declared `waits_for` for every stall whose exit provider is not the machine itself (Section 10.4.2)
- a check that every cycle in the dependency graph can be broken (Section 10.4.4)
- a declaration of where findings are handed, for every machine that only detects (Section 10.4.6)

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
