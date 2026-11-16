# Python and low-level design

Notes and code for **Track 1** of my job-switch plan ([../job_switch/syllabus.md](../job_switch/syllabus.md)). Main resource: Hello Interview LLD. How I learn: [../LEARNING.md](../LEARNING.md).

## Note format

- **Theory** (`.md` in `lld_in_a_hurry/`): starts with a `Review points` block, ends with review questions. Written from my teach-back, in my words.
- **Code** (`.py`): a docstring at the top with `Review points`, then the smallest example that shows the idea.
- **LLD problems** (`lld_problems/<nn>_<name>/`): `README.md` with requirements → class diagram → patterns used → extensions → review points, plus the code and a short test.

## Review order

Same as the learning order. ⬜ = not written yet.

### 1. In a Hurry (Hello Interview)

| # | Note | Code |
|---|---|---|
| 1 | ⬜ `lld_in_a_hurry/01_introduction.md` | — |
| 2 | ⬜ `lld_in_a_hurry/02_delivery_framework.md` | — |
| 3 | ✔ [lld_in_a_hurry/03_design_principles.md](lld_in_a_hurry/03_design_principles.md) | [SOLID/solid_in_python.py](SOLID/solid_in_python.py) |
| 4 | ✔ [lld_in_a_hurry/04_oop_concepts.md](lld_in_a_hurry/04_oop_concepts.md) | [oop_concepts.py](oop_concepts.py), [class_and_access_modifiers.py](class_and_access_modifiers.py), [types_enums_dataclass.py](types_enums_dataclass.py) |
| 5 | ⬜ `lld_in_a_hurry/05_design_patterns.md` | [design_patterns/](design_patterns/) |

### 2. Design patterns (code)

| Group | Done | To add |
|---|---|---|
| Creational | [singleton](design_patterns/creational/singleton.py), [factory](design_patterns/creational/factory.py), [builder](design_patterns/creational/builder.py) | — |
| Structural | [adapter](design_patterns/structural/adapter.py), [decorator](design_patterns/structural/decorator.py), [facade](design_patterns/structural/facade.py), [composite](design_patterns/structural/composite.py) | — |
| Behavioural | [strategy](design_patterns/behavioural/strategy.py), [observer](design_patterns/behavioural/observer.py), [state](design_patterns/behavioural/state.py), [command](design_patterns/behavioural/command.py) | chain of responsibility |

Recognize-only, no code: proxy, template method, iterator, abstract factory.

### 3. Concurrency (Hello Interview)

| # | Topic | Code |
|---|---|---|
| 1 | Introduction | ⬜ |
| 2 | Correctness | [Concurrency/1_correctness.py](Concurrency/1_correctness.py) |
| 3 | Coordination | ⬜ |
| 4 | Scarcity | ⬜ |

### 4. LLD problems

Folder `lld_problems/`, created with the first problem. Order: Connect Four, Amazon Locker, Elevator, Parking Lot, File System, Movie Ticket Booking, Logging Service, Rate Limiter, then LRU cache, Splitwise, vending machine, pub-sub queue.
