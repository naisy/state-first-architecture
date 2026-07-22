# State-First Architecture (SFA)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **Represent system flow control as explicit State, Event, Context, and Transition definitions, and do not hide control logic outside the state transition model.**

👉 [日本語版READMEはこちら (Japanese README)](./README.ja.md)  
🎮 [**Play now in your browser: SFA Game Demo Portal**](https://naisy.github.io/state-first-architecture/demo/)

---

## What Is State-First Architecture (SFA)?

State-First Architecture is a design approach that explicitly models system control decisions using the following elements:

- **State**: the current control position of the system
- **Event**: a normalized input used to evaluate state transitions
- **Context**: explicit data read or updated by Guards and Actions
- **Transition**: a rule connecting a State and Event to the next State
- **Guard**: a pure condition that determines whether a Transition may be selected
- **Action**: a change or side effect associated with a Transition
- **Role**: the actor that owns a State or process, such as a Frontend, Backend, or Worker

SFA does not attempt to eliminate conditional logic itself. It attempts to eliminate implicit flags, ordering dependencies, hidden ownership, and ad hoc state changes used for flow control outside the state transition model.

```text
Raw Input / Timer / Network / AI Decision
                  |
                  v
             Projection
                  |
                  v
          Canonical Event
                  |
                  v
State + Context --Transition/Guard--> Action --> Next State
```

---

## Core Principles of SFA

### 1. Explicit State

State that affects control decisions is made explicit as observable and verifiable State or Context. Information required for flow control is not hidden outside the state transition model.

### 2. Canonical Event

Different input sources, such as keyboard input, API responses, timers, and AI decisions, are normalized into Canonical Events through Projection. Subsequent Transitions do not depend on the original input source.

### 3. Declared Transition

State changes occur through declared Transitions. A Frontend, Backend, Worker, AI, or other actor does not arbitrarily mutate Context or State to bypass the defined flow.

### 4. Pure Guard / Controlled Action

A Guard performs evaluation only and does not modify Context. An Action performs only the changes and side effects authorized by the selected Transition.

### 5. Explicit Role and Ownership

The Role that owns each State, and the runtime ownership of processes spanning multiple steps, are made explicit. The design defines whether another decision may preempt a long-running process and when that ownership completes, becomes invalid, or is released.

### 6. Observable Outcome

The result of Event handling is observable. At minimum, the implementation should distinguish successful application, intentional discard, contract-based rejection, an unhandled Event, and Action failure.

---

## What SFA Defines

SFA does not prescribe a specific UI, game, persistence format, communication protocol, or database. It treats the following as common architectural specifications:

- the meaning of State / Event / Context / Transition / Guard / Action
- the evaluation order of Transition candidates
- the execution order of a Transition
- termination conditions for AUTO transitions
- normalization of input through Projection
- Role boundaries and runtime ownership
- Event outcomes and observable failures
- lifecycle boundaries for creating, retaining, discarding, and restoring State machine instances
- the scope of static validation and runtime contracts

The following belong to the specification of each application that uses SFA:

- where a game returns the player after death
- key bindings such as F8 or F9
- a save-data schema
- domain rules for payments, orders, equipment, bombs, and similar concepts
- concrete timeout values, retry counts, or search budgets

General principles may be extracted from application-specific examples, but the examples themselves do not become normative SFA requirements.

---

## Main Benefits of SFA

### Manual and automated control can converge on the same Transition

When a user action and an automated controller emit the same Canonical Event, the downstream behavior does not need to be duplicated.

```text
Manual Input ----\
                  > Projection --> RETURN_TO_TOWN --> shared transition
Auto Controller -/
```

### Unauthorized operations can be prevented by State structure

If a processing State has no `SUBMIT` Transition, an additional submission is not accepted. Instead of relying on an implicit flag such as `isSubmitting`, the current State expresses which Events are acceptable.

### The relationship between specification and implementation is easier to trace

By recording State, Event, Guard, Action, Role, and Outcome, the implementation can expose a Transition trace that explains why a particular operation was selected.

### The impact of changes is easier to constrain

Existing Transitions and Actions can remain unchanged while the design changes which Event is projected, which Guard becomes valid, or which Policy is selected.

---

## What SFA Does Not Guarantee Automatically

SFA is not a mechanism that prevents every kind of bug through state transitions alone. The following still require invariants, contract tests, and runtime validation:

- uniqueness of entity identity
- correctness of numeric or physical calculations
- transactional consistency inside Actions
- corruption of persisted data
- failures of external APIs or devices
- performance, memory, and search-time limits
- implementation defects inside Guards or Actions

SFA provides a control structure that makes such failures observable instead of hiding them, including the State, Event, and Action in which they occurred.

---

## Proofs of Concept (PoC)

This repository contains several game demos that apply SFA. Their game-specific rules are not part of the SFA specification, but they serve as examples for examining State, Event, Guard, Action, Projection, and Role boundaries.

### SFA Tetris

- expresses input that must not be accepted during animation through State definitions
- evaluates wall-kick and floor-kick candidates as ordered Transitions
- separates rendering from game flow control

### SFA Freeway

- makes the relationship between input and driving State explicit
- models escaping to the title screen as an Event and Transition

### SFA Rogue

- projects manual input and auto-patrol decisions into Canonical Events
- makes runtime ownership of multi-step tactics explicit
- validates Transitions through debug snapshots and deterministic replay

These are application examples of SFA. Their game-specific specifications are not included in the normative sections of the technical specification.

---

## Running the Demos

### Play online

[naisy.github.io/state-first-architecture/demo/](https://naisy.github.io/state-first-architecture/demo/)

### Run locally

1. Clone or download this repository.
2. Open `demo/index.html` in a browser.

---

## Documentation

- [Technical Specification (English)](./specification.md)
- [技術仕様書（日本語版）](./specification.ja.md)

The technical specification separates normative requirements from reference implementations and defines the execution semantics of the SFA engine, Projection, Event outcomes, Roles, Ownership, Lifecycle, and the boundaries of verifiability.

---

## License

This project is released under the MIT License. See [LICENSE](./LICENSE) for details.
