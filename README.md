# Python and low-level design

Notes and code for **Track 1** of my job-switch plan ([../job_switch/syllabus.md](../job_switch/syllabus.md)). Main resource: Hello Interview LLD. How I learn: [../LEARNING.md](../LEARNING.md).

## How to review

Go through the folders in number order, and the files inside each folder in number order. Every note is one short sitting: test yourself on its review questions first, then read its review points.

| # | Folder | What's in it |
|---|---|---|
| 1 | [01_lld_in_a_hurry/](01_lld_in_a_hurry/) | Theory notes: design principles, OOP concepts, Python mechanics |
| 2 | [02_design_patterns/](02_design_patterns/) | One code file per pattern: creational → structural → behavioural |
| 3 | [03_concurrency/](03_concurrency/) | Concurrency topics, one file each |
| 4 | `04_lld_problems/` | Solved LLD problems (created with the first problem) |

`html_pages/` holds saved Hello Interview pages for checking notes against the source. It is not part of the review path and is git-ignored.

## Note format

- **Theory** (`.md` in `01_lld_in_a_hurry/`): starts with a `Review points` block and ends with review questions whose answers are collapsed. Written from my teach-back, in my words.
- **Keep each note short.** A long Hello Interview page is split into parts of about 150–300 lines, each with its own review points and questions.
- **Code** (`.py`): a docstring at the top with `Review points`, then the smallest example that shows the idea.
- **LLD problems** (`04_lld_problems/<nn>_<name>/`): `README.md` with requirements → class diagram → patterns used → extensions → review points, plus the code and a short test.

## Review order

Same as the learning order. ⬜ = not written yet.

### 1. In a Hurry (Hello Interview)

| # | Note | Hello Interview page | Code |
|---|---|---|---|
| 1 | ⬜ `01_lld_in_a_hurry/01_introduction.md` | Introduction | — |
| 2 | ⬜ `01_lld_in_a_hurry/02_delivery_framework.md` | Delivery Framework | — |
| 3 | ✔ [03_general_principles.md](01_lld_in_a_hurry/03_general_principles.md) | Design Principles, part 1 | — |
| 4 | ✔ [04_solid.md](01_lld_in_a_hurry/04_solid.md) | Design Principles, part 2 | [04_solid_example.py](01_lld_in_a_hurry/04_solid_example.py) |
| 5 | ✔ [05_encapsulation_abstraction.md](01_lld_in_a_hurry/05_encapsulation_abstraction.md) | OOP Concepts, part 1 | — |
| 6 | ✔ [06_polymorphism_inheritance.md](01_lld_in_a_hurry/06_polymorphism_inheritance.md) | OOP Concepts, part 2 | — |
| 7 | ✔ [07_python_oop_toolkit.md](01_lld_in_a_hurry/07_python_oop_toolkit.md) | — (my older Python notes, merged) | — |
| 8 | ⬜ `01_lld_in_a_hurry/08_design_patterns.md` | Design Patterns | [02_design_patterns/](02_design_patterns/) |

### 2. Design patterns (code)

Written before the Hello Interview pass. Each file gets tightened when I study that pattern and teach it back.

| # | Group | Done | To add |
|---|---|---|---|
| 1 | Creational | [singleton](02_design_patterns/01_creational/01_singleton.py), [factory](02_design_patterns/01_creational/02_factory.py), [builder](02_design_patterns/01_creational/03_builder.py) | — |
| 2 | Structural | [adapter](02_design_patterns/02_structural/01_adapter.py), [decorator](02_design_patterns/02_structural/02_decorator.py), [facade](02_design_patterns/02_structural/03_facade.py), [composite](02_design_patterns/02_structural/04_composite.py) | — |
| 3 | Behavioural | [strategy](02_design_patterns/03_behavioural/01_strategy.py), [observer](02_design_patterns/03_behavioural/02_observer.py), [state](02_design_patterns/03_behavioural/03_state.py), [command](02_design_patterns/03_behavioural/04_command.py) | chain of responsibility |

Recognize-only, no code: proxy, template method, iterator, abstract factory.

### 3. Concurrency (Hello Interview)

| # | Topic | Code |
|---|---|---|
| 1 | Introduction | ⬜ |
| 2 | Correctness | [03_concurrency/02_correctness.py](03_concurrency/02_correctness.py) |
| 3 | Coordination | ⬜ |
| 4 | Scarcity | ⬜ |

### 4. LLD problems

Folder `04_lld_problems/`, created with the first problem. Order: Connect Four, Amazon Locker, Elevator, Parking Lot, File System, Movie Ticket Booking, Logging Service, Rate Limiter, then LRU cache, Splitwise, vending machine, pub-sub queue.
