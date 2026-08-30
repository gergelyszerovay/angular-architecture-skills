# Angular block architecture

A layered building-block architecture for modern Angular (v17+, signals, standalone,
zoneless-compatible), packaged as a skill for coding agents.

Every file belongs to exactly one **block type**, identified by its filename suffix, and the
suffix decides what the file may import. The suffix is not decoration — it is the boundary,
so getting the name right is what makes the architecture hold.

The architecture draws **two** boundaries, and keeping them distinct is what makes it
predictable.

**The view-model seam**, at `.facade.ts`. Above it, code is screen-shaped: components and
pages that render one view model. Below it, code is resource-shaped: resources, mutations,
stores, services. The facade is the translator — domain types in, display-ready values out.
Business logic never lives above this line.

**The framework boundary**, at `domain/` and `util/`. Below *this* line is plain TypeScript
that imports nothing — no `@angular/*`, no RxJS, no DI — and survives every UI and vendor
decision.

The tier between them (`.resource.ts`, `.mutation.ts`, `.store.ts`, `.api.ts`) is below the
seam but still Angular-coupled: it calls `inject()`, returns `Resource<T>`, and cannot be
tested without Angular. It is disposable in the same way components are. What makes it
*swappable* is not that it is framework-free but that each vendor is confined to a named set
of suffixes — so replacing one is a rewrite of files matching one glob.

Two Angular-specific commitments sharpen the view-model seam:

- **Signals above, RxJS below.** Components and facades speak signals only. RxJS is confined
  to `services/` and interop edges — `toSignal` happens at the boundary, never in a component.
- **DI is the wiring, not the architecture.** `inject()` is how blocks find each other, but
  what a block *may* inject is decided by its suffix.

## Contents

- `skills/SKILL.md` — the architecture: both boundaries, a decision procedure for placing any
  new file, the suffix tables, the six load-bearing rules, one declaration per file
- `skills/references/blocks.md` — per-block contracts: what each takes, exposes, may inject,
  and must never import
- `skills/references/layouts.md` — the three lib layouts, their tradeoffs and migrations
- `skills/references/vendors.md` — the resource API, RxJS containment, NgRx SignalStore,
  TanStack Query as an alternative, Supabase
- `skills/references/lint.md` — where every boundary in the skill is mechanically enforced

## The six rules

Everything else is convention. These are enforced:

1. **`domain/` and `util/` import nothing.** No `@angular/*`, no RxJS, no vendor packages, no
   DI. This is the framework boundary, and the rule that makes the tier survive rewrites.
2. **Wire shapes stay in `services/`.** DTO types and generated database types are importable
   only by `.api.ts`, `.mapper.ts`, and `.schema.ts`.
3. **A component injects at most one facade**, and no resource, mutation, store, or api.
4. **A facade injects no other facade**, no `.api.ts`, and no `HttpClient`.
5. **RxJS stays below the view-model seam.** The `async` pipe in a template is a violation,
   not a technique.
6. **Cross-lib imports hit the public surface only**, and the graph stays acyclic.

## Lib layouts

A lib is one unit of the app — a declared public surface, a private interior, its own path
alias. It is a unit of encapsulation, not of reuse; most libs are imported by nobody but the
router. Three ways to express the surface, chosen **per app**, never per lib:

| Layout | Surface is | Private placement | Needs |
|---|---|---|---|
| Path-depth (default) | files at the lib root | chosen by the author | nothing beyond the skill |
| Barrel (Nx-style) | one root `index.ts` | chosen by the author | resolved-path analysis |
| Fractal-tree | a configured list of declaration names | derived from the call graph | a generator + validator |

Path-depth is the default: no extra tooling, and the boundary cannot rot silently — a file is
either at the root or it is not. Fractal-tree is the outlier worth knowing about: it is the
only one that answers "where does this file go" without a human, which matters when agents
write most of the code, at the cost of a tree that churns whenever the call graph changes.

`skills/references/layouts.md` has the trees, the tradeoff table, and the conversions.
