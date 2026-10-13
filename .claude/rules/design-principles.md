---
paths: ["src/**"]
---
# Design principles for this repo

When two of these disagree, the earlier one wins: YAGNI → KISS → DRY → SOLID.

- YAGNI: build only what a current test or ticket demands. No config flag, no hook point, no abstract base class with one implementation. Delete a dead branch instead of guarding it.
- KISS: the simplest thing that passes the tests. A plain function before a class, a class before a framework. A change that needs a paragraph to explain is the wrong change.
- DRY: three occurrences justify an abstraction, two do not. Duplicated *knowledge* (a rule, a rate, a format) is the bug; duplicated *shape* is often fine.
- SRP: one reason to change per module. A file past ~200 lines or a function past ~30 lines usually holds two.
- OCP: extend by adding a case or handler, not by editing a switch that every feature touches.
- LSP: a subtype may not narrow its inputs or widen its errors. No `throw new UnsupportedOperationException()` in an override.
- ISP: a caller depends on the narrowest interface that serves it. No fat "manager" interface.
- DIP: policy depends on a protocol, not on a concrete adapter. Wiring happens at the composition root only.
