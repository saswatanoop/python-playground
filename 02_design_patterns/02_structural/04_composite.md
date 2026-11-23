# Composite

Source: not on Hello Interview's Design Patterns page. Taught by Claude on 22 Nov 2026 in the page's style; teach-back pending. Needed for the File System LLD problem.

## What this note covers

- Trees where a group must be treated like a single item
- Leaf and composite behind one interface
- Where `add()` lives, and the traps of recursion

## Review points

- **What it is:** single items (leaves) and groups of items (composites) share one interface, so callers treat a whole tree like one item.
- **The crux:** a composite holds children *of the interface type*, which can be leaves or other composites, and implements each operation by asking its children.
- **Signal:** "contains", "nested", "folders inside folders", "a team of teams", "a total over the hierarchy". File systems, org charts, nested comments, menus, bundles of products.
- **Payoff:** no `isinstance` checks in callers; new node types plug in.
- **Traps:** cycles (a folder inside itself), very deep recursion, and `add()` on a leaf.

---

## The discussion

### "A folder holds files and other folders. How do I get its total size?"

Without a shared interface, every caller checks types:

```python
def total_size_naive(item) -> int:
    if isinstance(item, dict):                          # a "folder": {name: child}
        return sum(total_size_naive(child) for child in item.values())
    return item                                         # a "file": its size
```

Every operation (size, search, count, print) repeats that type check, and a new node type, say a symlink, means finding all of them.

### "So what does Composite do?"

Files and folders implement the same interface. A folder's children are typed as that interface, so a folder can't tell, and doesn't care, whether a child is a file or another folder:

```python
from abc import ABC, abstractmethod

class Node(ABC):
    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    def size(self) -> int: ...

class File(Node):                                       # leaf
    def __init__(self, name: str, size: int):
        super().__init__(name)
        self._size = size

    def size(self) -> int:
        return self._size

class Folder(Node):                                     # composite
    def __init__(self, name: str):
        super().__init__(name)
        self._children: list[Node] = []

    def add(self, child: Node) -> None:
        if child is self:
            raise ValueError("a folder can't contain itself")
        self._children.append(child)

    def size(self) -> int:
        total = 0
        for child in self._children:
            total += child.size()                       # file or folder: same call
        return total

root = Folder("root")
docs = Folder("docs")
docs.add(File("cv.pdf", 300))
docs.add(File("notes.md", 20))
root.add(docs)
root.add(File("photo.jpg", 900))

root.size()         # 1220
docs.size()         # 320: any subtree works the same way
```

The recursion lives in the structure: `root.size()` asks `docs`, which asks its files. The caller just calls `size()`.

### "Where does `add()` go: on `Node` or only on `Folder`?"

Two schools:

- **On `Node`** (every node looks the same): `File.add` must raise. Callers never need to check types, but a bug shows up only at runtime.
- **Only on `Folder`** (safer): you can't call `add` on a file at all, but code that builds trees must know it holds a `Folder`.

For an interview, put `add` on `Folder` only. It's simpler to explain, and the operations that matter (size, search, print) are still on `Node`.

### "What can go wrong with a tree like this?"

- **Cycles.** Add `root` into `docs` and `size()` recurses forever. The check above only stops the direct case; a real file system checks that the new child isn't an ancestor (walk up parent pointers).
- **Deep trees.** Python's recursion limit is about 1000 frames. Very deep trees need an explicit stack instead of recursion.
- **Parent pointers.** Needed for `path()` and "move this folder". Keep them updated on `add` and remove, or they go stale.

### Gotchas

- **`isinstance` checks in callers.** If a caller asks "is this a file or a folder?", an operation is missing from the interface.
- **Cycles.** Check before `add`, or `size()` never returns.
- **Building it for flat data.** No nesting, no Composite: a list is enough.

### Connections

- Polymorphism: one call, a different implementation per node type ([06_polymorphism_inheritance.md](../../01_lld_in_a_hurry/06_polymorphism_inheritance.md)).
- Decorator also holds an object of its own interface, but exactly one, to add behaviour; a composite holds many, to treat a group as one ([02_decorator.md](02_decorator.md)).
- LLD problems: File System; also org charts and nested comments.

---

## Review questions

Answer first, then open the answer to check.

1. What makes a folder and a file interchangeable for the caller, and how does `Folder.size()` work?

   <details><summary>Answer</summary>

   - Both implement `Node`, and a folder's children are typed as `Node`.
   - `Folder.size()` sums `child.size()` over its children; each child is a file or another folder, and the recursion follows the tree.

   </details>

2. A caller writes `if isinstance(node, Folder)`. What does that tell you?

   <details><summary>Answer</summary>

   - An operation the caller needs is missing from `Node`. Add it to the interface and let each type implement it.

   </details>

3. `add()` on `Node` or only on `Folder`? Trade-off?

   <details><summary>Answer</summary>

   - On `Node`: uniform, but `File.add` must raise, so mistakes show up at runtime.
   - On `Folder`: a file can't be given children at all, but tree-building code must know it holds a `Folder`. The safer interview default.

   </details>

4. Someone adds `root` into `docs`. What happens, and how do you prevent it?

   <details><summary>Answer</summary>

   - A cycle: `size()` recurses until `RecursionError`.
   - Before adding, check the child isn't the folder itself or any of its ancestors, using parent pointers.

   </details>
