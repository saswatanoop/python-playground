# Command

Source: not on Hello Interview's Design Patterns page. Taught by Claude on 22 Nov 2026 in the page's style; teach-back pending.

## What this note covers

- Turning an action into an object
- Undo and redo with two stacks
- What an undo has to remember
- When a plain function is enough

## Review points

- **What it is:** wrap a request as an object with `execute()` (and often `undo()`), so it can be stored, queued, logged, undone or replayed.
- **The crux:** the code that *triggers* an action (a button, a queue worker) holds only `Command` objects and calls `execute()`; it doesn't know what they do.
- **Signal:** "undo/redo", "queue or schedule jobs", "replay or log actions", "macros".
- **Undo needs state captured at execute time** (the text that was deleted, the old value).
- **Undo/redo:** two stacks. A new command clears the redo stack.
- **No undo needed?** In Python, a callable often is the command.

---

## The discussion

### "Build a text editor with undo. Why can't the editor just call `document.insert(...)`?"

It can, until someone asks for undo. A direct call leaves nothing behind: once `insert` has run, there's no record of what happened, so there's nothing to reverse. And a "delete" can't be undone unless someone remembered *what* was deleted.

### "So what does Command do?"

Each action becomes an object that knows how to do itself and how to undo itself:

```python
from abc import ABC, abstractmethod

class Document:
    def __init__(self) -> None:
        self.text = ""

class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...

    @abstractmethod
    def undo(self) -> None: ...

class Insert(Command):
    def __init__(self, document: Document, text: str):
        self._document = document
        self._text = text

    def execute(self) -> None:
        self._document.text += self._text

    def undo(self) -> None:
        self._document.text = self._document.text[: -len(self._text)]

class DeleteLast(Command):
    def __init__(self, document: Document, count: int):
        self._document = document
        self._count = count
        self._deleted = ""                              # filled in by execute

    def execute(self) -> None:
        self._deleted = self._document.text[-self._count :]
        self._document.text = self._document.text[: -self._count]

    def undo(self) -> None:
        self._document.text += self._deleted
```

`DeleteLast` shows the key point: **undo needs state captured when the command ran.** It saves the deleted text in `execute`, because by `undo` time it's gone from the document.

### "And who keeps track of what to undo?"

The invoker: the editor. It runs commands and keeps two stacks:

```python
class Editor:
    def __init__(self) -> None:
        self._undo_stack: list[Command] = []
        self._redo_stack: list[Command] = []

    def run(self, command: Command) -> None:
        command.execute()
        self._undo_stack.append(command)
        self._redo_stack.clear()                        # a new action ends the redo history

    def undo(self) -> None:
        if not self._undo_stack:
            return
        command = self._undo_stack.pop()
        command.undo()
        self._redo_stack.append(command)

    def redo(self) -> None:
        if not self._redo_stack:
            return
        command = self._redo_stack.pop()
        command.execute()
        self._undo_stack.append(command)

document = Document()
editor = Editor()
editor.run(Insert(document, "hello"))
editor.run(Insert(document, " world"))
editor.run(DeleteLast(document, 5))
document.text       # "hello "
editor.undo()
document.text       # "hello world"
editor.undo()
document.text       # "hello"
editor.redo()
document.text       # "hello world"
```

`Editor` has no idea what inserting or deleting means. A new action (bold, replace) is a new command class; the editor doesn't change.

### "Why does a new command clear the redo stack?"

Type "hello world", undo " world", then type "!". Redo would now re-insert " world" after the "!", into a history that no longer exists. Every editor you've used drops the redo history when you do something new.

### "Do I need a class if there's no undo?"

No. If all you need is "store an action and run it later", like a job queue, a Python function is already a command object:

```python
from typing import Callable

jobs: list[Callable[[], None]] = []
```

Append functions, and a worker calls each one. Use classes when commands need `undo`, carry data you want to inspect or log, or must be serialised to run elsewhere.

### Gotchas

- **Undo without saved state.** Capture what you'll need during `execute`, not during `undo`.
- **Forgetting to clear redo** on a new command: redo then corrupts the document.
- **Commands holding the wrong reference.** A command stores its receiver (`document`) at creation; if the target can change, decide which one it should act on.
- **Commands with logic in them.** A command calls the receiver; editing rules belong in `Document`.

### Connections

- Strategy also wraps behaviour in an object, but a strategy is *how* an object does its job; a command is *one request*, stored to run, undo or replay ([01_strategy.md](01_strategy.md)).
- Job queues and producer–consumer: Concurrency (Week 3).
- LLD problems: text editor, any "undo" requirement, task schedulers.

---

## Review questions

Answer first, then open the answer to check.

1. A text editor needs undo. Why does turning each action into an object make that possible?

   <details><summary>Answer</summary>

   - A direct call leaves no record. A command object is stored on a stack after running, and knows how to reverse itself.
   - The editor only calls `execute`/`undo`; it doesn't know what each action does.

   </details>

2. How does `DeleteLast.undo()` know what text to restore?

   <details><summary>Answer</summary>

   - `execute` saved the deleted text on the command before removing it. Undo needs state captured at execute time.

   </details>

3. Walk through undo and redo with two stacks. What happens to the redo stack when the user types something new, and why?

   <details><summary>Answer</summary>

   - Run: execute, push on undo. Undo: pop undo, call `undo`, push on redo. Redo: pop redo, `execute`, push on undo.
   - A new command clears redo, because the redo history assumed a past that no longer exists.

   </details>

4. A background job queue with no undo. Command classes or functions?

   <details><summary>Answer</summary>

   - Functions (`Callable[[], None]`) are enough.
   - Classes when you need undo, inspectable or loggable data, or serialisation.

   </details>
