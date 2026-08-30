# Lib layouts

Two ways to express one rule: **a lib has a public surface and a private interior, and
crossing that boundary is visible in the import statement.**

- **Path-depth layout** (default) — public files sit at the lib root, private ones in
  subdirectories. Depth *is* the boundary; no annotation, no extra file.
- **Barrel layout** (Nx-style) — a single `index.ts` at the lib root declares the public
  surface explicitly. Everything else, including root-level files, is private.
- **Fractal-tree layout** — the surface is a configured list of declaration names, and every
  private file's *position* is derived from the call graph rather than chosen. Requires a
  generator; the boundary is mechanical, not conventional.

Both keep the same suffixes, the same block contracts, and the same Feature Slicing rule
(shared code starts local — see `SKILL.md`). They differ only in how the boundary is
declared and, consequently, in what tool can enforce it.

**Choose one per app, not per lib.** Two conventions in one repo means every reviewer and
every agent must first work out which applies before judging an import. The same reasoning
the skill applies to TanStack Query applies here.

## Which one am I in?

Look at the lib root: an `index.ts` means the barrel layout, no `index.ts` means
path-depth. A `fractalTs` key in `package.json` means the fractal-tree layout regardless of
what the root looks like — the config, not the tree, is the source of truth there. If some
libs have an `index.ts` and some do not, that is drift to fix, not a design.

## Path-depth layout

```
libs/cart/
  cart.resource.ts     public
  cart.mutation.ts     public
  cart.paths.ts        public
  cart.routes.ts       public
  cart.events.ts       public
  cart.flow.ts         root — the routes file provides it; see below
  cart.page.ts         root — routed; see below
  components/          private
  facades/             private
  internal/            private
```

Rules:

- A file at the lib root is public. Do not put anything there you do not want imported.
- Everything else is private, regardless of subdirectory name.
- **Two blocks sit at the root without being a public API: `.flow.ts` and `.page.ts`.**
  The routes file must reference them, and the routes file is at the root — but a flow is
  reachable only from inside its route subtree (DI scoping, not lint), and a page is mounted
  by `loadComponent`, never imported. Root placement makes them *importable*; nothing makes
  importing them correct. If that bothers you, the barrel layout states it exactly: neither
  appears in `index.ts`.
- Cross-lib imports are absolute and reach the root only:
  `@app/libs/cart/cart.resource`.
- Same-lib imports are relative: `./internal/cart.vm`.
- No `index.ts` anywhere.
- **`export *` is banned** — see *No wildcard re-exports*; it applies here too.

Enforced by block 6 in `references/lint.md`: a specifier with a second segment after the
lib name is reaching into a subdirectory.

## Barrel layout

```
libs/cart/
  index.ts             the public surface — the only importable module
  cart.resource.ts     private
  cart.routes.ts       private (referenced by index.ts and the shell's loadChildren)
  components/          private
  facades/             private
  internal/            private
```

```ts
// libs/cart/index.ts
export { injectCartResource } from './cart.resource';
export { injectCartMutations } from './cart.mutation';
export { cartPaths } from './cart.paths';
export type { CartEvents } from './cart.events';
```

Rules:

- `index.ts` exists only at the lib root. No barrels in subdirectories — those are the
  ones that hide layering violations with nothing gained.
- **Named re-exports only. `export *` is banned.**
- Cross-lib imports name the lib only: `@app/libs/cart`.
- `.routes.ts` stays reachable by the shell's `loadChildren` path even when not re-exported —
  a lazy import is a path, not a specifier the barrel can serve.

### The barrel costs you the eslint suffix rules

This is the tradeoff, and it is not optional to understand before choosing.

`no-restricted-imports` matches the **specifier string**, not the resolved path. Given
`import { X } from '@app/libs/cart'`, the rule engine sees one opaque string. It cannot
tell whether `X` is a `.ui.ts`, a `.store.ts`, or an `.api.ts` — so every suffix-glob import
rule in `references/lint.md` silently stops applying to anything crossing a barrel. The rules
do not error; they match nothing, which is worse.

**Therefore the barrel layout requires Sheriff or dependency-cruiser.** Resolved-path
analysis is the only thing that can see through `index.ts`. Adopting the barrel layout
without one of those tools is not a stylistic choice — it is turning the enforcement off.

The barrel pattern in the merged `src/**` block of `references/lint.md` is scoped
accordingly: under the barrel layout it becomes **no barrels below the lib root**
(`'**/*/*/index'`), and the root `index.ts` is the single allowed exception. Subdirectory barrels stay banned under both layouts.

## Fractal-tree layout

A third option, and the one that differs in kind: the other two let you *choose* where a
private file sits, and only constrain what may cross the boundary. The fractal-tree layout
derives every private file's position from the call graph, and a generator moves files until
the tree matches. Placement stops being a judgement call.

The rule is one line: **a file's callees live in a directory named after it.**

```
libs/cart/
  cart.resource.ts                   public — listed in fractalTs.features
  cart.mutation.ts                   public — listed
  cart.paths.ts                      public — listed
  cart.routes.ts                     public — listed
  _shared/
    cart-line.model.ts               2+ consumers inside the lib
  checkout.facade.ts                 public — listed
  checkout.facade/                   private — callees of checkout.facade.ts
    to-line-vm.format.ts
    checkout.vm.ts
    submit-checkout.mutation.ts
    submit-checkout.mutation/
      map-checkout-dto.mapper.ts
```

Rules:

- **The public surface is a configured list of declaration names**, not a location:
  `package.json#fractalTs.features` maps a lib name to the declarations other libs may
  import. Plus everything in a `_shared/` directory.
- **A file imports only from its direct children (one level down), from `_shared/` at or
  above its level, and from other libs' public surface.** Never a sub-subtree, never
  upward, never a sibling's interior.
- **A file with two or more consumers moves to `_shared/` at their lowest common ancestor.**
  This is the same promotion ladder as *Shared code starts local*, applied continuously and
  by a tool rather than at review time.
- One declaration per file, named in kebab-case after that declaration — which the block
  suffixes already satisfy.
- No `index.ts`, no re-exports except named ones in the app's single entry point.
- Test and story files sit beside their subject and are invisible to every rule.

### What it buys, and what it costs

The generator is the whole point. Nothing in the other two layouts tells you whether
`to-line-vm.format.ts` belongs in `internal/` or beside its caller — the file gets a home
when it is created and keeps it long after the calls moved. Here, the second consumer
appears and the tool relocates the file to `_shared/`; the last consumer disappears and it
nests deeper. The tree stays a true picture of the call graph without anyone maintaining it.

The cost is that the tree churns. A refactor that changes who calls what produces a diff
full of moves, review is noisier, and the generator has to run and stabilise (`0 moved` on a
second pass) before a commit is trustworthy. Editors' recent-files and any stale path in a
comment go stale together.

It also **subsumes the "shared code starts local" ladder for lib-internal files** while
leaving the lib-level ladder alone: `_shared/` promotion is automatic and mechanical inside a
lib; promoting a block down to global `ui/` or `domain/` is still a human decision under the
second-consumer rule.

### Interaction with the block suffixes

The two axes are orthogonal and compose without conflict — suffix says *what a file is*,
fractal position says *who calls it*:

- The **suffix import rules keep working unchanged.** They match specifier strings, and
  fractal-tree specifiers are ordinary relative and absolute paths with no barrel in the way.
- The **lib-type matrix keeps working**, but Sheriff's directory globs must be written
  against the top-level tiers (`domain/`, `ui/`, `services/`), not against per-lib
  subdirectories — `components/` and `facades/` do not exist here, since position comes from
  the call graph.
- **`domain/`, `util/`, and `services/` are unaffected.** They are already leaf-ward tiers
  whose files import almost nothing; a fractal tree over them is mostly flat.

One genuine friction: fractal placement will happily nest a `.ui.ts` under the
`.container.ts` that renders it, which reads oddly next to a global `ui/` tier holding
product-agnostic components. That is consistent — a one-consumer `.ui.ts` *is* lib-local and
the block rules already say so — but expect to explain it.

### When to choose it

Choose fractal-tree when **agents write most of the code**. "Where does this file go" is the
question an agent answers worst and most often; making it derivable removes a whole class of
drift, and the generator repairs the answer when the agent gets it wrong. It also pays off in
a codebase whose call graph is genuinely deep — long chains of single-consumer helpers are
exactly what nesting makes legible.

Avoid it when the team reviews by file tree and expects stable paths, when the codebase is
mostly flat (a lib of ten sibling files gains nothing from a rule about depth), or when you
cannot commit to running the generator on every change. Half-applied, it is worse than
either alternative: the tree implies a call graph that is no longer true.

It needs the generator and validator to exist for your stack. This repo's implementation
(`@fractal-ts/cli` for validation, `apps/destiny` for restructuring) targets plain
TypeScript and is not Angular-aware — it has no notion of templates, DI, or the block
suffixes, so it will not stop a component from injecting `HttpClient`. It replaces the
*placement* question only; every rule in `references/lint.md` still has to be enforced
separately.

## No wildcard re-exports

```ts
export * from './cart.resource';                    // banned
export { injectCartResource } from './cart.resource';  // required
```

Applies under **both** layouts and to every file, not only `index.ts`. Under the barrel layout
it defeats the only thing the barrel is for; under path-depth a wildcard re-export in a public
root file republishes a private module through it, which is the same hole by another route.

`export *` re-exports whatever the target file happens to export, now and after every future
edit. Four consequences, in rough order of how much they hurt:

- **The surface changes without touching `index.ts`.** Adding an export to any re-exported
  file publishes it, silently, with no diff on the boundary file — so reviewing `index.ts`
  (the barrel layout's main defence, since its failure mode is surface creep) stops working.
- **The surface becomes unreadable.** `export *` cannot be answered without opening every
  target. A named list *is* the lib's public API, in one screen.
- **It re-exports transitively.** If a target itself re-exports, `export *` carries that
  through, and a private block reaches the surface via a file nobody thought was public.
- **Ambiguity is silent.** Two targets exporting the same name resolve to neither, and the
  export simply vanishes rather than erroring.

Enforced by block 8 in `references/lint.md` (`no-restricted-syntax` on
`ExportAllDeclaration`), which applies to **every** file, not only `index.ts` — a wildcard
re-export in an ordinary module opens the same hole, republishing a private module through a
public one. Note it is a *syntax* rule, not an import rule: it inspects the file rather than
the specifier, so unlike the suffix globs it keeps working under both layouts.

Type-only re-exports follow the same rule — `export type { CartVm }`, not `export type *`.

## Choosing

| | Path-depth | Barrel | Fractal-tree |
|---|---|---|---|
| Public surface | implicit (root files) | explicit (`index.ts`) | explicit (config declaration list) |
| Extra file per lib | none | one | none (a `package.json` key) |
| Enforced by | eslint globs alone | Sheriff / dependency-cruiser required | generator + validator |
| Suffix import rules across libs | work | need resolved-path analysis | work |
| Private file placement | chosen by author | chosen by author | derived from the call graph |
| Renaming a public file | changes every importer | invisible behind the barrel | config names declarations, not paths |
| Nx `enforce-module-boundaries` | not applicable | native fit | not applicable |
| Failure mode | a private file drifts to the root | the barrel re-exports too much | tree churn; stale if the generator is skipped |

Default to **path-depth**. It needs no extra tooling, and the boundary cannot rot silently —
a file is either at the root or it is not.

Choose **barrel** when the repo is already Nx (its `enforce-module-boundaries` rule is built
for this shape and works on resolved paths), when libs are destined to become published
libraries, or when you are already running Sheriff for cycle detection and the marginal cost
is zero.

Choose **fractal-tree** when agents write most of the code and you can run its generator on
every change — it is the only one of the three that answers "where does this file go" without
a human. Do not choose it for the boundary alone: its public-surface rule is roughly what the
barrel gives you, and the nesting is the part you are actually buying.

Both failure modes are real. Under path-depth, someone drops a file at the root that was
meant to be private, and it is public forever after because nothing flags it. Under the
barrel, someone appends an export to `index.ts` to unblock themselves and the surface grows
without review. Path-depth's failure is visible in a file tree; the barrel's is visible only
in a diff of `index.ts` — which is why `export *` is banned outright, so that diff is
guaranteed to exist, and why that one file is worth reviewing line by line.

## Converting between them

Path-depth → barrel: add `index.ts` re-exporting exactly the current root files, then rewrite
cross-lib specifiers to drop the trailing module. Root files stay where they are; they
become private by virtue of the barrel, not by moving.

Barrel → path-depth: for each name the barrel exports, ensure the file is at the lib root,
then rewrite importers to the full path and delete `index.ts`. Anything the barrel exported
from a subdirectory has to move up — that is the migration's real work, and it is a good
audit of what the surface actually was.

Either direction → **fractal-tree**: declare the current public names in
`package.json#fractalTs.features`, delete any `index.ts`, then run the generator and let it
place everything else. The first run produces a large diff and that is expected — review the
resulting *surface*, not the moves. Re-run until it reports nothing moved.

Fractal-tree → either: the tree is already legal under path-depth once you accept the nesting
(root files are public, everything else is not), so the conversion is mostly deleting the
config and stopping the generator. Flattening the callee directories afterwards is optional
and unrelated.

Do the whole repo in one change either way. A half-converted repo has both failure modes and
neither enforcement.
