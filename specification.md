# Technical Specification of State-First Architecture (SFA)

## 1. Overview and Design Philosophy

State-First Architecture (SFA) is a software design methodology that defines and resolves all system behaviors, controls, and synchronization between components/roles solely through the logical model of **"States"** and **"Transitions."**

In the face of uncertainties such as concurrent processes, asynchronous networks, and rapid user double-clicks, SFA rejects any reliance on "hidden variables" that exist outside the state transition diagram—such as timestamps for event ordering or implicit boolean flags (e.g., duplicate submission prevention flags). All exceptional behaviors and mutual exclusions must be modeled explicitly within states and transitions, and processed deterministically by the engine.

### The 3 Core Principles of SFA
1. **Strict State-Driven**: Every behavior of the system is determined solely by the intersection of the `Current State` and the `Triggered Event`. No implicit control flags or ad-hoc if-statements for state management/progression exist within the application layer.
2. **Explicit Roles**: Every state must explicitly declare its managing role (`frontend`, `backend`, `worker`, etc.) within its schema.
3. **Forward-Only Autonomy**: Each state is completely oblivious to how it reached its current state. It only holds "forward transition rules"—knowing where it stands now and which target state it can transition to next based on incoming events and guard conditions.

---

## 2. State Definition Meta-model (Common Schema)

All states are described in a format (JSON/YAML, etc.) that complies with the following meta-model. This definition serves as the **Single Source of Truth (SSOT)** in the development process.

```json
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "title": "SFA_StateDefinition",
  "type": "object",
  "properties": {
    "state_id": { 
      "type": "string", 
      "description": "Unique state identifier. It is recommended to prefix with the role (e.g., FE_IDLE, BE_PROCESSING)" 
    },
    "role": { 
      "type": "string",
      "description": "The lifecycle role that owns and executes this state (e.g., frontend, backend, worker)"
    },
    "description": { 
      "type": "string", 
      "description": "A brief overview of the business logic represented by this state" 
    },
    "on_enter": { 
      "type": "string", 
      "description": "The side-effect (function name) executed immediately AFTER entering this state"
    },
    "on_exit": { 
      "type": "string", 
      "description": "The side-effect (function name) executed immediately BEFORE exiting this state"
    },
    "next_transitions": {
      "type": "array",
      "description": "A list of valid transitions from this state. The order of the array denotes the evaluation priority.",
      "items": {
        "type": "object",
        "properties": {
          "event": { 
            "type": "string", 
            "description": "The event name that triggers the transition. If AUTO is specified, the transition happens automatically once conditions are met." 
          },
          "guard": { 
            "type": "string", 
            "description": "The name of a pure function used to evaluate whether the transition is allowed. Defaults to true if omitted." 
          },
          "action": { 
            "type": "string", 
            "description": "The name of a side-effect function that modifies context at the moment the transition occurs (optional)" 
          },
          "target_state": { 
            "type": "string", 
            "description": "The target state ID to transition to" 
          }
        },
        "required": ["event", "target_state"]
      }
    }
  },
  "required": ["state_id", "role", "next_transitions"]
}
```

---

## 3. Engine Execution Specification (Deterministic Routing Algorithm)

An "SFA Engine" that interprets and drives the defined logical model must adhere strictly to the following algorithms during state transitions.

### ① Prioritized Fallback Evaluation
If multiple transition definitions exist for the same event (e.g., with different guard conditions), the engine must scan them in the exact order declared in the `next_transitions` array (top-to-bottom).

1. Event `E` is triggered.
2. The engine filters the `next_transitions` of the current state where `event === E` in their declared order.
3. The engine evaluates the `guard` function of each candidate sequentially.
4. **The first candidate whose guard returns `true`** is confirmed as the next state, and further scanning is terminated immediately (Fallback established).
5. If all candidates' guards return `false`, the engine **silently discards** the event `E` without triggering any transition.

### ② Recursive Evaluation of AUTO (Transient) Transitions
Immediately after a transition is completed, if the new state contains `event: "AUTO"` in its definitions, the engine must immediately evaluate the guard condition without waiting for further external inputs. This AUTO evaluation runs **recursively** until the system reaches a stable state (a state with no AUTO events or where all AUTO guards return `false`).

---

## 4. Resolving Asynchronous Processing and Time Uncertainty

Bugs caused by "warps in time"—such as network delays or rapid multi-clicks on the frontend—are completely resolved **solely through the logical structure of state transitions**.

### ① Physical Synchronization via Boundary States
When different roles (e.g., Frontend and Backend) synchronize through communication, the sender system must always go through a **"Boundary State"** (waiting state).

*   **Automatic Locks (No Spam Bugs)**: A boundary state (e.g., `FE_WAITING_BE`) only registers transitions for events like "successful response from backend (`SUCCESS`)" or "failed response (`ERROR`)". During this state, even if the user spams buttons on the UI and triggers other events (e.g., `SUBMIT`), the SFA engine **silently discards them because those transitions do not exist in the boundary state.** Introducing flags like `isSubmitting` into the application code is strictly prohibited.

### ② "Natural Decay" of Delayed Events Due to State Mismatch
When an event triggered in an older state reaches the system late due to network latency, the system has already transitioned to the next state. If the new state's transition table does not define a transition for that old event, the engine simply discards it. Without comparing timestamps, **the logical shield of "state mismatch" automatically renders delayed packets harmless**.

---

## 5. Complete Separation of Concerns (Developer vs. System)

The key feature of SFA is the **elimination of state-control conditional branching (`if`/`switch` statements) and complex control flags** from the primary application logic.

While local infrastructure layers—such as geometric collision detection, user interface rendering, or parsing engines—may legitimately contain local conditional branches (`if` statements), all rules governing system progression are centralized within the structured data (JSON/YAML).

```
+-------------------------------------------------------------+
|               1. State Definitions (JSON / YAML)            |
|                 - System structure, rules, and priorities    |
+-------------------------------------------------------------+
                               |
       +-----------------------+-----------------------+
       v                                               v
+-------------------------------+             +-------------------------------+
|       2. Guards (Pure)        |             |      3. Actions (Impure)      |
| - Validate transition rules   |             | - Execute transition effects  |
| - Pure functions; no mutations|             | - UI, API calls, DB saves, etc|
+-------------------------------+             +-------------------------------+
```

1. **State Definitions (JSON/YAML)**: Declaratively describe the "skeleton (progression specifications)" of the system.
2. **Guards (Pure Functions)**: Write only the "evaluation logic" (returning a boolean) to verify if a transition is permitted.
3. **Actions (Impure Functions)**: Write only the "side-effects" (e.g., UI rendering outputs, API calls) triggered upon entering/exiting a state, or at the moment of transition.

---

## 6. Verifiability and Future Prospects

Since all behaviors are described as structured data (JSON/YAML), it is possible to verify and guarantee the absence of various bugs using **Static Analysis** even before running the program.

*   **Dead-end State Detection**: Identify isolated states with no incoming transitions or unintended dead-ends with no outgoing transitions using graph theory analysis before deployment.
*   **Completeness Verification**: Static validators can automatically alert developers about unhandled events (e.g., "Event X triggered during State Y has no defined transition").
*   **Automated Documentation & Diagrams**: Automatically generate state transition diagrams (e.g., Mermaid) that are guaranteed to be 100% aligned with the codebase during the build process.
