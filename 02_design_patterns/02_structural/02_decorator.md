# Decorator

Source: Hello Interview LLD → Design Patterns, and my teach-back on 21 Nov 2026 (clean on the third pass).

## What this note covers

- The subclass explosion that Decorator fixes
- The crux: same interface, wrap and delegate
- Stacking, and which wrapper runs first
- The base decorator that saves pass-through boilerplate
- Decorator versus a plain subclass, and versus Python's `@decorator`

## Review points

- **What it is:** a wrapper that implements the same interface as the object it holds, does its extra work, and passes the call through.
- **The problem:** optional features as subclasses multiply: 3 features give 7 combinations per base class. Decorators give 3 small classes you combine freely.
- **The crux: the decorator *is a* `DataSource`.** That's why callers can't tell it's wrapped and why decorators can wrap decorators.
- **The original object never changes.** You get a new wrapper; only code holding the wrapper gets the extra behaviour.
- **Order:** the last wrapper added is the outermost and runs first on a write.
- **Base decorator:** one class that passes every method through, so each real decorator overrides only what it changes.
- **Subclass instead** when the difference is fixed at design time and isn't combined with others.

---

## The discussion

### "I need encryption and compression, but only sometimes. Why not subclass?"

Try it. One feature is one subclass: `EncryptedFileDataSource`. Fine.

Now three optional features (encryption, compression, logging), any combination: 3 + 3 + 1 = 7 subclasses. Two kinds of data source? 14. Add a fourth feature and it's 15 per source, 30 in all. Each one is a near-copy of another, and the names read like a cry for help: `LoggedCompressedEncryptedFileDataSource`.

With decorators it's 3 feature classes plus 2 sources, combined at runtime as each case needs.

### "So what does a decorator look like?"

Two things make it work: it **implements** the interface, and it **holds** an object of that interface.

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    @abstractmethod
    def write_data(self, data: str) -> None: ...

    @abstractmethod
    def read_data(self) -> str: ...

class MemoryDataSource(DataSource):         # stands in for a file or a database
    def __init__(self) -> None:
        self.stored = ""

    def write_data(self, data: str) -> None:
        self.stored = data

    def read_data(self) -> str:
        return self.stored

class EncryptionDecorator(DataSource):      # is a DataSource...
    def __init__(self, wrapped: DataSource):
        self._wrapped = wrapped             # ...and has a DataSource

    def write_data(self, data: str) -> None:
        encrypted = f"enc({data})"           # fake encryption, so the order shows
        self._wrapped.write_data(encrypted)

    def read_data(self) -> str:
        data = self._wrapped.read_data()
        return data.removeprefix("enc(").removesuffix(")")
```

Because `EncryptionDecorator` is a `DataSource`, it goes anywhere a `DataSource` is expected. The code using it doesn't know, and doesn't need to.

### "So the decorator adds the feature to my object?"

That's how I put it first, and it's not quite right. The object is untouched. You get a *new* object that holds the old one:

```python
memory = MemoryDataSource()
source = EncryptionDecorator(memory)

source.write_data("card 4242")
memory.stored           # "enc(card 4242)": went through the wrapper
memory.write_data("plain")
memory.stored           # "plain": the original has no encryption
```

Anyone still holding `memory` bypasses the encryption. Hand out the wrapper, not the original.

### "The interface has ten methods. Do I write ten pass-throughs in every decorator?"

Without help, yes, and a method added to the interface means editing every decorator. The fix is a **base decorator** that passes everything through once:

```python
class DataSourceDecorator(DataSource):
    def __init__(self, wrapped: DataSource):
        self._wrapped = wrapped

    def write_data(self, data: str) -> None:
        self._wrapped.write_data(data)

    def read_data(self) -> str:
        return self._wrapped.read_data()

class CompressionDecorator(DataSourceDecorator):
    def write_data(self, data: str) -> None:
        compressed = f"zip({data})"
        self._wrapped.write_data(compressed)

    def read_data(self) -> str:
        data = self._wrapped.read_data()
        return data.removeprefix("zip(").removesuffix(")")

class LoggingDecorator(DataSourceDecorator):
    def __init__(self, wrapped: DataSource):
        super().__init__(wrapped)
        self.log: list[str] = []

    def write_data(self, data: str) -> None:   # read_data comes from the base
        self.log.append(f"writing {len(data)} chars")
        self._wrapped.write_data(data)
```

`LoggingDecorator` overrides one method and inherits the rest.

### "I stack two of them. Which runs first?"

I got this backwards in the teach-back. **The last wrapper you add is the outermost, and the outermost runs first** on the way in:

```python
memory = MemoryDataSource()
source = EncryptionDecorator(memory)        # added first: inner
source = CompressionDecorator(source)       # added last: outer, runs first
source.write_data("hi")
memory.stored           # "enc(zip(hi))": compressed, then encrypted
source.read_data()      # "hi": unwrapped in reverse
```

And the order isn't cosmetic. Compress-then-encrypt is the right one: encrypted bytes look random, and random data barely compresses. Wrap the other way round and the compression does nothing.

### "Isn't this just Python's `@decorator`?"

Same idea, different target:

- **`@log` on a function** wraps the *function*, once, when it's defined. Every call gets logged. It only has to match the call signature.
- **The Decorator pattern** wraps an *object*, at runtime, one instance at a time. One data source is encrypted, another isn't.

Both are wrappers that add work and delegate. Hello Interview flags the naming clash too.

### Interviewer: "When would you just subclass?"

When the variation is **fixed at design time** and isn't combined with others. A `CsvDataSource` versus a `JsonDataSource` is a stable kind of thing; that's a subclass.

Hello Interview's rule: if the behaviour depends on runtime conditions (logging only in debug, caching only for some requests), Decorator. If it's a predefined type difference, subclass. And Decorator comes up less often than Strategy or Observer; listen for "optional features", "stack behaviours", "combine enhancements".

### Gotchas

- **Handing out the unwrapped object.** It silently skips every feature.
- **Wrong stacking order.** The last added runs first. Encrypt-then-compress compresses nothing.
- **Skipping the base decorator.** Every decorator repeats every pass-through, and every interface change touches all of them.
- **`isinstance(source, MemoryDataSource)` is `False`** once wrapped. Code that checks concrete types breaks; code that uses the interface doesn't.
- **Deep stacks are hard to debug.** Five wrappers means five frames before you reach the real work.

### Connections

- Composition over inheritance, made concrete ([06_polymorphism_inheritance.md](../../01_lld_in_a_hurry/06_polymorphism_inheritance.md)).
- Open/closed: a new feature is a new wrapper; nothing existing changes ([04_solid.md](../../01_lld_in_a_hurry/04_solid.md)).
- Adapter also wraps, but *changes* the interface; Decorator keeps it.

---

## Review questions

Answer first, then open the answer to check.

1. Three optional features, two data sources. Why not subclasses, and how many classes does each approach need?

   <details><summary>Answer</summary>

   - Subclasses: 7 combinations per source (3C1 + 3C2 + 3C3), so 14, all near-copies. A fourth feature makes it 15 per source, 30 in all.
   - Decorators: 3 feature classes plus 2 sources, combined at runtime.

   </details>

2. What must a decorator share with the object it wraps, and what breaks without it?

   <details><summary>Answer</summary>

   - The interface: it *is a* `DataSource` as well as *having* one.
   - Without it, the wrapper can't be passed where a `DataSource` is expected, and another decorator can't wrap it, so stacking breaks.

   </details>

3. You want compress-then-encrypt on write. Which decorator do you add first, and why does the order matter?

   <details><summary>Answer</summary>

   - Add `EncryptionDecorator` first and `CompressionDecorator` last: the last added is outermost and runs first.
   - Encrypted data looks random and barely compresses, so the other order wastes the compression.

   </details>

4. The interface has ten methods and your new decorator changes one. How do you avoid writing ten methods?

   <details><summary>Answer</summary>

   - A base decorator that implements the interface and passes every method to the wrapped object.
   - Real decorators inherit from it and override only what they change.

   </details>

5. `CsvDataSource` vs `JsonDataSource`, and "log writes only in debug mode". Decorator or subclass for each?

   <details><summary>Answer</summary>

   - CSV vs JSON: subclass. A fixed, design-time difference in kind.
   - Debug logging: decorator. Chosen at runtime and combinable with other features.

   </details>
