# Angular block architecture

A skill for Angular capability boundaries, state ownership, building blocks, and
dependency enforcement.

Start with [skills/SKILL.md](skills/SKILL.md), which routes to:

- [Architecture](skills/references/architecture.md): library roles and public contracts.
- [Blocks](skills/references/blocks.md): rendering, state, commands, APIs, and pure logic.
- [Declarations](skills/references/declarations.md): primary declarations, explicit companions, and enforcement across layouts.
- [State and operations](skills/references/state-and-operations.md): lifetimes and consistency.
- [Layouts](skills/references/layouts.md): separate path-depth, barrel/Nx-like, and fractal-tree guides.
- [Vendors](skills/references/vendors.md): containment and integration.
- [Enforcement](skills/references/lint.md): coverage, pitfalls, and verification.

Libraries encapsulate capabilities even with one consumer. A screen may compose multiple
facades while each component consumes at most one. Stores own state, flows own processes,
and provider scope determines lifetime. Domain dependencies are explicit, acyclic, and pure.

Architectural invariants are distinguished from implementation conventions. Load detailed
references only when relevant to the task.
