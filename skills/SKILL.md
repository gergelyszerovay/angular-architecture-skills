---
name: angular-block-architecture
description: Design, implement, or review Angular capability boundaries, building blocks, state ownership, and dependency enforcement. Use for applications adopting this architecture or when explicitly requested; supports path-depth, barrel/Nx-like, and fractal-tree layouts.
---

# Angular block architecture

Libraries encapsulate cohesive capabilities with public contracts and private interiors.
They need neither routes nor multiple consumers. These are app-local boundaries unless
the project explicitly uses independent packages.

Composition assembles capabilities into a UI host, such as a page, dialog, or embedded
widget. Workflows describe coordinated processes; workspaces are one interface pattern.
Neither is required for every application or capability.

## Two architectural seams

The **view-model seam** sits at each facade. It translates application state into
presentation-ready values and exposes user commands. Components render those values and
forward intent; resources, stores, flows, APIs, and cross-capability coordination stay
below the seam. Facades may perform simple projections, and templates may handle ordinary
visual conditions. A composed interface can contain several local facade seams without
duplicating the shared state they observe.

The **framework boundary** surrounds pure domain models, rules, and policies. They have
no Angular, DI, transport, reactive-library, or presentation dependencies. Explicit acyclic
dependencies on other domain modules and approved pure libraries are allowed. Blocks below
the view-model seam may still depend on Angular; being below that seam does not make them
framework-independent.

## Architectural invariants

- Cross-library dependencies are explicit, acyclic, and use declared public surfaces.
- Every authoritative state value has one owner. Distinguish drafts, caches, applied
  results, and presentation state; derive views instead of synchronizing writable copies.
- A component consumes at most one facade. Composition may host children with their own
  facades. Facades never consume other facades.
- Components do not consume APIs, resources, commands, stores, or flows directly.
- Domain code has no framework, transport, or presentation dependencies. Explicit acyclic
  domain dependencies and approved pure libraries are allowed.
- Wire contracts stay inside data access. Public results use owned capability/domain
  contracts, not exported DTOs or typed transport clients.
- Stateful and asynchronous blocks declare ownership, lifetime, cleanup, and failure
  behavior. Related results have an explicit consistency/publication contract.

## Working procedure

1. Inspect project instructions, Angular version, registered library roots, aliases, and
   enforcement. Preserve the owning package's selected layout and task scope.
2. For boundary changes, read [architecture.md](references/architecture.md).
3. Before implementing or changing blocks, read [blocks.md](references/blocks.md).
   For source file creation or declaration changes, also read
   [declarations.md](references/declarations.md), the shared file/declaration policy.
   For state, loading, or integration, also read
   [state-and-operations.md](references/state-and-operations.md).
4. For placement, use [layouts.md](references/layouts.md), then read only the selected
   layout guide for the owning package. Layout is selected per package, not per app or
   library; packages in the same application may use different layouts. Generated UI
   files do not establish the package's authored-code layout.
5. For vendor integration, read [vendors.md](references/vendors.md). For setup, boundary
   changes, or lint debugging, read [lint.md](references/lint.md).
6. Verify relevant behavior and boundaries. Report actual coverage and limitations.

## Implementation conventions

Use signals above the view-model seam and convert RxJS at lower-level interop boundaries.
Prefer standalone components, signal inputs/outputs, native control flow, and OnPush
behavior using the project's Angular version and conventions.

Suffixes express responsibility, not lifetime or visibility. Use one primary declaration
per implementation file with only the explicit supporting-declaration cases in
[declarations.md](references/declarations.md). Apply this policy across all three layouts;
fractal-tree additionally imposes dependency-derived placement rules.

Use .command.ts for new explicitly triggered operations, including scans and refreshes.
Existing .mutation.ts blocks follow command restrictions until a scoped migration renames
them. Do not rename unrelated application files just to adopt the revised skill.

Simple projection belongs in facades; ordinary visual conditions may stay in templates.
Extract business rules and substantial transformations. Product-specific presentational
components are valid; shared product-independent UI is a separate placement decision.

Do not scaffold future libraries, add dependencies, migrate applications, or install agent
hooks merely to follow these conventions. Architectural invariants and implementation
preferences are distinct.
