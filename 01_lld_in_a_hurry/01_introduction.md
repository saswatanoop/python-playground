# LLD interviews: what they test

Source: Hello Interview LLD → In a Hurry → Introduction. Read 16 Nov 2026. Next: [02_delivery_framework.md](02_delivery_framework.md).

## What this note covers

- What an LLD (OOD) interview asks for, and how it differs from system design
- How the format changes by region and company size
- The five areas interviewers score

## Review points

- **LLD = OOD.** Same interview, different name. You design the classes, interfaces, relationships and state transitions of one self-contained problem (parking lot, elevator, Connect Four).
- **LLD vs HLD:** HLD is architecture at scale (services, storage, caching, sharding; no code). LLD is the code structure of *one* part: classes, methods, state. *"System design is the map; LLD is the blueprint for one building on it."*
- **Hello Interview's stance:** pragmatic, not academic. Composition over inheritance, simple state over deep hierarchies, patterns only when they fit.
- **Regional variants matter for me (India target):** expect structured **pseudocode**, **vaguer requirements** you must pin down, and **patterns asked by name** at small and mid-size companies. US Big Tech wants partial real code and cares about reasoning more than pattern vocabulary. When unsure, ask the recruiter.
- **Rubric, 5 areas:** problem analysis → class design → code quality → extensibility → communication.

---

### Same prompt, two interviews: ride sharing

| | HLD | LLD |
|---|---|---|
| Output | Services on a whiteboard: matching, pricing, location, datastore, queue | Classes: `Trip`, `TripState`, `PricingCalculator`, `Location` |
| Talk about | Load spikes, consistency, scaling, reliability | Methods, ownership, data model, state transitions |
| Code | None | Pseudocode or partial real code |

What the LLD answer looks like: an entity that owns its state machine and rejects illegal transitions.

```python
class Trip:
    def assign_driver(self, driver: Driver) -> None:
        if self.state != TripState.REQUESTED:
            raise ValueError(f"cannot assign driver in state {self.state}")
        self.driver = driver
        self.state = TripState.DRIVER_ASSIGNED

    def complete(self, calculator: PricingCalculator) -> None:   # pricing injected, not hard-coded
        if self.state != TripState.IN_PROGRESS:
            raise ValueError(f"cannot complete trip in state {self.state}")
        self.fare = calculator.calculate_fare(self)
        self.state = TripState.COMPLETED
```

- The 2 ideas the example carries: **state transitions guarded inside the entity** (encapsulation), and a **varying rule behind an interface** (`PricingCalculator`, which is DIP plus strategy).

### What interviewers score

| Area | Strong signal | Red flag |
|---|---|---|
| Problem analysis | Extract entities and responsibilities, lock scope with a few questions | Jumping into code |
| Class design | Right responsibilities, clear ownership, clean method signatures | Weak structure that makes everything downstream harder |
| Code quality | Encapsulation, managed state, composition used sensibly, SoC, naming, **dependency direction** | Ad hoc decisions |
| Extensibility | Absorbs the follow-up requirement without a rewrite | Designing for every possible future (rewarded: *adapts easily*, not *anticipates everything*) |
| Communication | Clear narrative, adjusts when probed; think out loud | Silent coding |

---

## Review questions

Answer first, then open the answer to check.

1. The prompt is "design a ride-sharing app". How do the HLD and LLD answers differ in output and in what you discuss?

   <details><summary>Answer</summary>

   - HLD: services and data flow (matching, pricing, location, store, queue), talking about scale, load spikes, consistency. No code.
   - LLD: classes for one part (`Trip`, `TripState`, `PricingCalculator`), talking about methods, ownership and state transitions, in pseudocode or partial code.

   </details>

2. You're interviewing at an Indian mid-size product company versus US Big Tech. What changes in how you run the LLD round?

   <details><summary>Answer</summary>

   - India / mid-size: structured pseudocode is fine, requirements are vaguer so you drive the scoping, and you should name a pattern when it's the clean fit.
   - US Big Tech: partial real code, slightly clearer requirements, reasoning over vocabulary; bring a pattern up only when it applies naturally.
   - Either way: ask the recruiter what format they expect.

   </details>

3. Your design anticipates five future features the interviewer never asked for. Which rubric area does that hurt, and why?

   <details><summary>Answer</summary>

   - Extensibility (and code quality via YAGNI/KISS). Interviewers reward a design that *adapts easily* to the follow-up, not one that anticipates every future. Extra machinery costs time and reads as over-engineering.

   </details>
