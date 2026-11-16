# LLD delivery framework

Source: Hello Interview LLD → In a Hurry → Delivery Framework. Read 16 Nov 2026. Previous: [01_introduction.md](01_introduction.md). Next: [03_general_principles.md](03_general_principles.md).

## What this note covers

- The five steps of an LLD interview and how long to spend on each
- How to turn a one-line prompt into requirements, entities and class outlines
- Where each rule belongs: orchestrator or the entity that owns the data
- What to implement, how to verify it, and how to answer extension questions

## Review points

- **~35 minutes, 5 steps:**

  | # | Step | Time | Output on the board |
  |---|---|---|---|
  | 1 | Requirements | ~5 min | Numbered spec + explicit **Out of scope** list |
  | 2 | Entities and relationships | ~3 min | Entity list + a few arrows |
  | 3 | Class design | 10–15 min | Per class: state + methods, derived from requirements |
  | 4 | Implementation | ~10 min | Key methods: happy path, then edge cases; then trace a scenario |
  | 5 | Extensibility | ~5 min | Point to where the change lands; no rewrite |

- **Two failure modes it prevents:** diving into code and drowning in edge cases before the structure is visible; or perfecting setup and running out of time before any real design.
- **Everything traces back to requirements:** entities come from the nouns, state from "what must this class remember?", methods from "what does the outside world need?". That's how you avoid both guessing and bloat.
- **Rule placement:** workflow and lifecycle rules ("can this run now?") go in the **orchestrator**; data rules ("is this cell taken?") go in the **entity that owns the data**. Tell, don't ask.
- **No UML.** Simple class notation (fields, `+ methods`, arrows). If an interviewer asks for UML, ask whether simplified notation is OK.
- If the interviewer pulls you off the framework, follow them, then gently steer back to the steps you haven't covered.

---

### 1. Requirements (~5 min)

Turn a one-line prompt into a spec. Prime your questions with 4 themes:

| Theme | Ask |
|---|---|
| Primary capabilities | What operations must it support? |
| Rules and completion | What means success or failure? When does it stop or change state? |
| Error handling | What happens on invalid input or an illegal action? |
| Scope boundaries | In: core logic, business rules. Out: UI, storage, networking, concurrency, extensibility? |

Confirm the spec, and **write the out-of-scope list too**: it stops scope creep. Tic Tac Toe:

```
Requirements
1. Two players alternate X and O on a 3x3 grid.
2. Win = a full row, column or diagonal.
3. Draw = all 9 cells filled, no winner.
4. Reject invalid moves (occupied cell, move after game over).
5. Query current state; reset the game.
Out of scope: UI, AI opponent, networked play, NxN boards, undo/redo
```

### 2. Entities and relationships (~3 min)

- **Entities = the meaningful nouns.** Filter: *changing state or enforces rules* → its own entity; *just information attached to something* → a field.
- **Relationships, 4 questions:** who orchestrates the main workflow? Who owns durable state? Has-a, uses, or contains? Where does each rule live?
- **This is the whole "class diagram":** a list and arrows, no notation.

```
Entities: Game, Board, Player
Game -> Board
Game -> Player (x2)
```

### 3. Class design (10–15 min)

Go **top-down: orchestrator first** (`Game`), then supporting entities. For each class answer two questions, both from the requirements:

- **State:** what must it remember to enforce the requirements? Map requirement → what to track.
- **Behaviour:** what operations or queries does the outside world need? A small API, one method per real action or question.

```
class Game
  - board: Board
  - player_x, player_o: Player
  - current_player: Player
  - state: GameState (IN_PROGRESS, WON, DRAW)
  - winner: Player | None
  + make_move(player, row, col) -> bool
  + get_current_player() -> Player
  + get_state() -> GameState
  + get_winner() -> Player | None
  + get_board() -> Board
```

- `Game` owns turn order and game state (workflow rules). `Board` owns `can_place()`, `check_win()`, `is_full()` (data rules on cells it owns).
- **UML:** skip it. It's outdated in production (Microsoft dropped UML tooling from Visual Studio in 2016), and the formality slows you down while you iterate live. Engineers design in code: stubbed classes and interfaces.

### 4. Implementation (~10 min)

1. **Ask first** what they want: pseudocode, near-complete code, or a verbal walkthrough. Default: pseudocode for the key methods. Use your daily language (Python), not an unfamiliar one.
2. **Happy path first:** inputs → steps → calls to other classes → return value or state change.
3. **Then edge cases:** invalid input, illegal operation, out of range, wrong state.
4. **Only the methods that show the design:** how classes cooperate, how state transitions, where rules live. The interviewer often names them.

```
make_move(player, row, col)
    if state != IN_PROGRESS: return false          # lifecycle rule → Game
    if player != current_player: return false
    if not board.can_place(row, col): return false # data rule → Board
    board.place(row, col, player.mark)
    if board.check_win(row, col, player.mark): state = WON; winner = player
    elif board.is_full(): state = DRAW
    else: current_player = other(player)
    return true
```

- **Patterns:** candidates over-engineer far more often than they miss a needed pattern. Justify each one before adding it.
- **Verify (1–2 min):** trace one non-trivial scenario tick by tick: initial state, each operation, the state after it, the transition to done. It catches a missed turn switch, a win that never triggers, or transitions in the wrong order. Many rubrics score this explicitly; fixing your own bug live is a *positive* signal.

```
start: board empty, current = X
make_move(X,0,0) → [0][0]=X, current = O
make_move(O,1,1) → [1][1]=O, current = X
... X completes the top row → state = WON, winner = X; next make_move returns false
```

### 5. Extensibility (~5 min, interviewer-led)

- **Depth depends on level:** junior gets little or none, mid-level gets 1–2 follow-ups, **senior gets several "what if…" in a row**. That's me.
- **Answer by pointing, not rewriting:** name the place in the design where the change lands and why nothing else moves.
- Example, *"add undo"*: every state change goes through `make_move`, so record the previous state on a history stack there. `undo()` pops it, and nothing else changes. This works *because* state mutations were isolated in one method.

---

## Review questions

Answer first, then open the answer to check.

1. Give the 5 steps with times. Which two failure modes does the pacing prevent?

   <details><summary>Answer</summary>

   - Requirements ~5, entities and relationships ~3, class design 10–15, implementation ~10 (including verification), extensibility ~5. About 35 minutes.
   - Diving into code and drowning in edge cases before the structure exists; or over-perfecting setup and running out of time before any real design.

   </details>

2. Prompt: "Design a parking lot." Which 4 themes drive your questions, and what goes in the out-of-scope list?

   <details><summary>Answer</summary>

   - Capabilities (park, unpark, fees?), rules and completion (when is a spot free, when is the lot full), error handling (no spot, unknown ticket), scope boundaries.
   - Out of scope, typically: UI, payments integration, persistence, networking, concurrency, multiple lots, unless the interviewer puts them in.

   </details>

3. "License plate", "spot", "ticket", "colour": which become entities and which become fields? What's the test?

   <details><summary>Answer</summary>

   - Test: does it hold changing state or enforce rules? Then it's an entity; otherwise it's a field.
   - Spot (occupied or free, size rules) and ticket (entry time, fee) are entities. Plate and colour are fields on `Vehicle` (or just on the ticket).

   </details>

4. In Tic Tac Toe, where does "is the game still running?" go, and where does "is this cell occupied?" go? Why does it matter?

   <details><summary>Answer</summary>

   - Game still running: a lifecycle rule → the orchestrator, `Game`.
   - Cell occupied: a data rule → the owner of the cells, `Board`.
   - Rules sit next to the state they protect, so APIs stay small and you know which class to check when something breaks (tell, don't ask).

   </details>

5. The interviewer asks for a UML class diagram. What do you say, and what do you draw?

   <details><summary>Answer</summary>

   - Ask whether simplified class notation is acceptable; it usually is.
   - Draw entities with fields and `+ methods`, plus arrows for has-a and uses, with multiplicity in words (`Game -> Player (x2)`).

   </details>

6. You finished `make_move`. What do you do with the next 2 minutes, and what bugs does it catch?

   <details><summary>Answer</summary>

   - Trace one concrete scenario tick by tick: initial state, each call, the state after it, the transition to the end.
   - Catches: a missing turn switch, a win that never triggers, transitions in the wrong order, edge-case handling that breaks the flow. Fixing it live is a positive signal.

   </details>

7. "Add undo." How do you answer without rewriting, and what earlier design decision makes it easy?

   <details><summary>Answer</summary>

   - Point to `make_move` as the single place state changes. Push the previous state (or a command) onto a history stack there; `undo()` pops and restores. Nothing else changes.
   - It's easy because state mutations were isolated in one method from the start.

   </details>
