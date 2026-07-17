# State-First Architecture (SFA)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **"Resolve all control, asynchronous complexity, and user interactions solely through 'State Transition Rules'."**  
> A software design philosophy that completely eliminates implicit state-control flags and if/else spaghetti from your application logic.

👉 [日本語のREADMEはこちら (Japanese README)](./README.ja.md)

---

## 💡 What is State-First Architecture (SFA)?

SFA is a software design philosophy that deterministically controls all system behaviors, asynchronous processes, and synchronization between components/roles using **only the logical model of "States" and "Transitions."**

It completely eliminates "hidden variables" that reside outside the state transition diagram—such as timestamp-based event ordering or manual flag management (`isSubmitting = true`)—which are notorious for introducing bugs in traditional development.

### The 3 Core Principles of SFA
1. **Strict State-Driven**: The system's behavior is determined solely by the intersection of the *Current State* and the *Triggered Event*. No implicit control flags or ad-hoc if-statements for state-control exist within the application layer.
2. **Explicit Roles**: Every state explicitly declares its owner/manager (e.g., `frontend`, `backend`, `worker`) in its schema.
3. **Forward-Only Autonomy**: Each state is oblivious to *how* it reached its current state. It only knows *where* it can transition next based on incoming events and guard conditions.

---

## 🎮 Proof of Concept (PoC): SFA Tetris

To prove the robustness of SFA, this repository includes a **proof-of-concept (PoC) Tetris game** written in pure, vanilla JavaScript.

Game development is one of the most challenging fields for state and async management, involving real-time key inputs, collision detection, and animation interruptions. In this demo, **all conditional branching related to gameplay progression is elegantly resolved solely through state transition definitions (JSON)**.

### 3 Key SFA Concept Proofs in This Demo

1. **Input Discarding During Animations (Structural Elimination of Spam Bugs)**
   During the line-clearing animation (`FE_CLEARING` state), keys like move (`MOVE_LEFT`) or rotate (`ROTATE`) are simply not registered in the transition definition. As a result, even if a user spams keys at lightning speed, the engine automatically and silently discards these events. There is not a single line of defensive state-control branching (such as `if (isAnimating) return;`) in the application logic.

2. **Declarative "Wall-Kick / Floor-Kick" Priority Control**
   The complex "kick" behavior (pushing a tetromino away from walls or the floor when rotating) is defined as a prioritized list of fallback transitions. The rule *"If normal rotation fails, try kick-left; if that fails, try kick-right..."* is represented as a declarative JSON array sequence rather than a chaotic, nested if/else jungle for state control.

3. **Total Separation of View and Logic**
   When fixing a rendering bug where font widths (full-width vs. half-width) mismatched, we replaced the rendering function (`render`) without touching a single line of state definitions, engine logic, or collision guards. This proves SFA's loose coupling and safe maintainability.

---

## 🚀 How to Run the Demo

1. Clone or download this repository.
2. Open `demo/index.html` directly in any web browser (no build steps, Node.js, or external libraries required!).

### Controls
* `Enter`: Start / Restart Game
* `←` / `→`: Move Tetromino Left / Right
* `↑`: Rotate Clockwise
* `↓`: Rotate Counter-Clockwise
* `Space`: Soft Drop (Increase fall speed)

---

## 📄 Documentation

For a deep dive into the SFA theory, common schema specification, and handling of asynchronous networks, please refer to the technical specifications:

* [Technical Specification (English version)](./specification.md)
* [技術仕様書 (Japanese version)](./specification.ja.md)

---

## ⚖️ License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.
