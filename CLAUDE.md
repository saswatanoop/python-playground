# Python and LLD learning folder

Part of the job-switch plan. Follow ../LEARNING.md (loaded from the parent CLAUDE.md) and the Track 1 plan in ../job_switch/syllabus.md. Keep the review-order tables below up to date when a note or code file is added.

Rules for this folder:

- Goal of `01_lld_in_a_hurry/`: after it, I have every OOP concept and Python language tool I need to write LLD code. If a tool is missing, add it there.
- Folders and files are numbered in review order. A new note takes the next number in its folder. The Hello Interview Design Patterns page goes in `02_design_patterns/00_overview.md`, not in `01_lld_in_a_hurry/`.
- In `02_design_patterns/`, each pattern is one note, `<nn>_<name>.md`, with its code as short snippets inside. Don't keep a separate `.py`; when an old pattern `.py` is studied, replace it with the `.md`. Run the note's snippets before finishing.
- Every note starts with a `What this note covers` list (topics only, no answers), then `Review points`.
- Keep each note short (about 150–300 lines). Split a long source page into parts, each with its own review points and review questions.
- Review questions test judgment, not recall, and each answer sits in a collapsed `<details><summary>Answer</summary>` block under its question.
- Notes must be concise and accurate. A correct, meaningful example beats copying the source's example.
- In `02_design_patterns/` only, write the body of a note as a discussion that teaches: each section heading is the question a curious learner or an interviewer would ask next ("Hold on, the if/elif is still there. What did I gain?"), answered in a conversational voice, building from the pain to the pattern to the pushbacks. Use my own wrong instincts from the teach-back as the questions. Model: `02_design_patterns/01_creational/02_factory.md`. The `Review points` block on top stays terse. Notes in `01_lld_in_a_hurry/` keep the reference style; don't convert them.
- Give each design pattern note a `Gotchas` section (traps, wrong instincts, interviewer pushbacks) and keep the tone lively. Leave out variants nobody uses any more (for example the Gang of Four Factory Method), even as recognize-only.
- Every snippet must be defined, used and consistent with the text. Run it before claiming an output.
- Write snippets as plain, readable Python: one statement per line. No `Service(dep).method()` chains (build the object on one line, call it on the next), no `;`, no `if x: return y` or `def f(): return y` on one line, no `self.a, self.b = a, b`. `def f(): ...` stubs are fine.
- When a saved source page exists in `html_pages/`, check the note against it for accuracy and missing points.

@README.md
