# Python and low-level design

Notes and code for **Track 1** of my job-switch plan ([../job_switch/syllabus.md](../job_switch/syllabus.md)). Main resource: Hello Interview LLD. How I learn: [../LEARNING.md](../LEARNING.md).

## How to review

Go through the folders in number order, and the files inside each folder in number order. Every note is one short sitting: test yourself on its review questions first, then read its review points.

| # | Folder | What's in it |
|---|---|---|
| 1 | [01_lld_in_a_hurry/](01_lld_in_a_hurry/) | Notes: design principles, OOP concepts, Python mechanics |
| 2 | [02_design_patterns/](02_design_patterns/) | Overview note, then one note per pattern: creational → structural → behavioural |
| 3 | [03_concurrency/](03_concurrency/) | Concurrency topics, one file each |
| 4 | `04_lld_problems/` | Solved LLD problems (created with the first problem) |

`html_pages/` holds saved Hello Interview pages for checking notes against the source. It is not part of the review path and is git-ignored.

## Note format

- **Notes** (`.md`): starts with a `What this note covers` list, then a `Review points` block, and ends with review questions whose answers are collapsed. Written from my teach-back, in my words.
- **Keep each note short.** A long Hello Interview page is split into parts of about 150–300 lines, each with its own review points and questions.
- **Design pattern notes are written as a discussion:** question-led sections, the questions a learner or interviewer would ask next, ending with a `Gotchas` list. The notes in `01_lld_in_a_hurry/` stay in the reference style (crux bullets under topic headings).
- **Design patterns:** one note per pattern, `<nn>_<name>.md`, with the code inside it as short snippets that run. No separate `.py`. Within each group, `<nn>` follows interview priority, highest first.
- **Other code** (`.py`): a docstring at the top with `Review points`, then the smallest example that shows the idea.
- **LLD problems** (`04_lld_problems/<nn>_<name>/`): `README.md` with requirements → class diagram → patterns used → extensions → review points, plus the code and a short test. Also record **v1 → what broke → v2** (my naive first design, the requirement that broke it, and the refactor) and **where I got stuck**, so the note can later be taught as a video (see "Make every note video-ready" in `../LEARNING.md`).

## Review order

Same as the learning order. ⬜ = not written yet.

### 1. In a Hurry (Hello Interview)

| # | Note | Hello Interview page | Code |
|---|---|---|---|
| 1 | ✔ [01_introduction.md](01_lld_in_a_hurry/01_introduction.md) | Introduction | — |
| 2 | ✔ [02_delivery_framework.md](01_lld_in_a_hurry/02_delivery_framework.md) | Delivery Framework | — |
| 3 | ✔ [03_general_principles.md](01_lld_in_a_hurry/03_general_principles.md) | Design Principles, part 1 | — |
| 4 | ✔ [04_solid.md](01_lld_in_a_hurry/04_solid.md) | Design Principles, part 2 | — |
| 5 | ✔ [05_encapsulation_abstraction.md](01_lld_in_a_hurry/05_encapsulation_abstraction.md) | OOP Concepts, part 1 | — |
| 6 | ✔ [06_polymorphism_inheritance.md](01_lld_in_a_hurry/06_polymorphism_inheritance.md) | OOP Concepts, part 2 | — |
| 7 | ✔ [07_python_oop_toolkit.md](01_lld_in_a_hurry/07_python_oop_toolkit.md) | — (my older Python notes, merged) | — |
| 8 | ✔ [08_python_objects_and_containers.md](01_lld_in_a_hurry/08_python_objects_and_containers.md) | — (drafted by Claude; teach back pending) | — |
| 9 | ✔ [09_parameters_and_dataclass_fields.md](01_lld_in_a_hurry/09_parameters_and_dataclass_fields.md) | — (drafted by Claude; teach back pending) | — |

Goal of this folder: after it, I have every OOP concept and Python language tool I need to write LLD code.

### 2. Design patterns

Start with ✔ [00_overview.md](02_design_patterns/00_overview.md), the note for the Hello Interview Design Patterns page (which patterns matter and when to use one). Then the patterns below. Inside each group, notes are numbered by interview priority: highest first, lowest last. The first patterns in each group are the ones on Hello Interview's page; the rest are a second tier. ✔ = note written.

| # | Group | Done | To add |
|---|---|---|---|
| 1 | Creational | ✔ [factory](02_design_patterns/01_creational/01_factory.md), ✔ [builder](02_design_patterns/01_creational/02_builder.md), ✔ [singleton](02_design_patterns/01_creational/03_singleton.md) | — |
| 2 | Structural | ✔ [decorator](02_design_patterns/02_structural/01_decorator.md), ✔ [facade](02_design_patterns/02_structural/02_facade.md), ✔ [composite](02_design_patterns/02_structural/03_composite.md), ✔ [adapter](02_design_patterns/02_structural/04_adapter.md) | — |
| 3 | Behavioural | ✔ [strategy](02_design_patterns/03_behavioural/01_strategy.md), ✔ [observer](02_design_patterns/03_behavioural/02_observer.md), ✔ [state](02_design_patterns/03_behavioural/03_state.md), ✔ [chain of responsibility](02_design_patterns/03_behavioural/04_chain_of_responsibility.md), ✔ [command](02_design_patterns/03_behavioural/05_command.md) | — |

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
