---
name: angular-block-architecture
description: Layered building-block architecture for modern Angular (v17+, signals, standalone) — file suffixes, import boundaries, and the view-model seam that keeps business logic out of components. Use this skill whenever writing, generating, reviewing, or refactoring code in an Angular codebase that uses these conventions, including when the user just says "add a page", "fetch this data", "add a form", "where should this go", "create a feature", "create a lib", or asks for a component or service without mentioning architecture at all. Also use it when deciding where a piece of code belongs, when naming a new file, or when reviewing a diff for layering violations. Covers signals, the resource API, NgRx SignalStore, HttpClient, Supabase, and Router integration.
---

# Angular block architecture

An architecture where every file belongs to exactly one **block type**, identified by its
filename suffix, and where the block type determines what the file may import. The suffix is
not decoration — lint rules are written against the suffix globs, so getting the name right
is what makes the boundary enforceable.

The architecture draws **two** boundaries, and keeping them distinct is what makes the rules
predictable. Most of this document is about the first; the second is the one that pays off
over years.

**The view-model seam**, at `.facade.ts`. Above it, code is screen-shaped: components and
pages that render one view model. Below it, code is resource-shaped: resources, mutations,
stores, services. The facade is the translator — domain types in, display-ready values out.
Business logic never lives above this line, and a component that reaches past its facade has
crossed it.

**The framework boundary**, at `domain/` and `util/`. Below *this* line is plain TypeScript
that imports nothing — no `@angular/*`, no RxJS, no DI — and survives every UI and vendor
decision. This is rule 1, and it is the most valuable rule here.

Both lines matter, and they are not the same line. Everything between them —
`.resource.ts`, `.mutation.ts`, `.store.ts`, `.api.ts` — is *below the view-model seam but
still Angular-coupled*: it calls `inject()`, returns `Resource<T>`, takes `Signal` params, and
cannot be tested without Angular. That tier is disposable in the same way components are.
What makes it *swappable* is not that it is framework-free but that each vendor is confined
to a named set of suffixes, so replacing one is a rewrite of files matching one glob.

So when this document says "below the seam" without qualification, it means the view-model
seam at the facade. Framework-free is a stronger claim, made only of `domain/` and `util/`.

Two Angular-specific commitments sharpen the view-model seam:

- **Signals above, RxJS below.** Components and facades speak signals only. RxJS is confined
  to `services/` and interop edges (`toSignal` happens at the boundary, never in a component).
- **DI is the wiring, not the architecture.** `inject()` is how blocks find each other, but
  what a block *may* inject is decided by its suffix, and linted.

## Before creating any file

Answer these in order. The first match wins.

1. **Does it import `@angular/*`?** No → it belongs in `domain/`, `util/`, or is a pure part of
   `services/`. Go to step 4.
2. **Does it have a template?** Yes → component. `.ui.ts` if it takes signal inputs only and
   could work in another app; `.container.ts` if it's lib-specific; `.page.ts` /
   `.dialog.ts` / `.layout.ts` if the router or the Dialog service mounts it.
3. **It's an injectable or a factory.** Which kind?
   - Provides environment values (session, flags, i18n, notify) → ambient, `.token.ts` or a
     root service in `app/`
   - Returns domain-typed read state for a resource → `.resource.ts`
   - Exposes `{ run, status }` for a write → `.mutation.ts`
   - Exposes a view model signal for one screen → `.facade.ts`
   - Holds shared client state → `.store.ts`; holds task state for a route subtree → `.flow.ts`
4. **Not an Angular block at all:** knows your business → `domain/`. Knows nothing about your
   product (could be published to npm) → `util/`. Talks to the network → `services/`. The
   first two are across the framework boundary and import nothing.

If a file seems to need two suffixes, it is two files. That is almost always the real answer
and splitting it is cheap.

## Suffixes

Grouped by what the file is made of, because that is what the lint rules test. The first
five groups sit above the framework boundary; the last sits below it. The view-model seam
runs *inside* that set — between the second group and the third.

### Things that render

Have a template or touch the DOM. Never import `.api.ts`, stores, or resources.

| Block | Suffix | Description |
|---|---|---|
| UI component | `.ui.ts` | Presentational component; `input()`/`output()` only, imports nothing outside `ui/` |
| Smart component | `.container.ts` | Lib-local component reading one facade and passing VM slices down |
| Page | `.page.ts` | Route entry component; router inputs only, at most one facade |
| Dialog | `.dialog.ts` | CDK/Material dialog component; typed `DIALOG_DATA` in, typed result out |
| Layout | `.layout.ts` | Structural shell component (chrome, slots) shared by pages |
| Adapter (imperative) | `.adapter.ts` | Wraps a non-declarative vendor widget — maps, charts, editors; signal-input surface, `DestroyRef` teardown |
| Generic pipe | `.pipe.ts` | Product-agnostic template transform; lives in `ui/` |

All of them: standalone, OnPush, signal inputs/outputs, empty constructor, native control
flow. `.adapter.ts` is the marked exception to the no-side-effects rule — it may hold
`ElementRef`, call `afterNextRender`, and import its vendor package, which is why it gets a
suffix instead of a comment.

Component files keep template and styles inline or as siblings (`cart-lines.ui.html`); the
`.ts` file carries the suffix the lint rules match.

### Things that answer "what does this screen show?"

The view-model seam itself. Domain types in, display-ready values out. At most one facade
per screen.

| Block | Suffix | Description |
|---|---|---|
| Use-case facade | `.facade.ts` | Screen-scoped seam exposing `Signal<Vm>` + commands; coordinates, never calculates |
| View model | `.vm.ts` | Type of what one screen renders — display-ready fields |
| Formatter | `.format.ts` | Pure domain-type → display-string functions; locale passed as an argument |

`.pipe.ts` and `.format.ts` both format, and the split is the product test: a pipe is
product-agnostic and lives in `ui/`; a formatter knows domain types and belongs to this tier.

### Things that talk to the server

The whole read/write path, wire format to domain type, in the order you read it.

| Block | Suffix | Description |
|---|---|---|
| Read resource | `.resource.ts` | `inject*` factory returning `Resource<DomainType>`; fetches on demand |
| Mutation | `.mutation.ts` | Injectable write action exposing `{ run, status }`, named after a verb |
| Service | `.api.ts` | `@Injectable` over `HttpClient`; parses DTOs and returns domain types |
| Wire schema | `.schema.ts` | Zod schema plus inferred DTO type; service-private (form schemas live in `internal/`) |
| DTO mapper | `.mapper.ts` | `(dto) => DomainType`; the only translator of wire field names |
| HTTP interceptor | `.interceptor.ts` | `HttpInterceptorFn` — auth headers, retries, error normalization |

Observables do not escape upward: `.api.ts` returns `Promise<DomainType>` or a cold
observable consumed only by a `resource` loader. The mapper is mandatory even when the wire
types are accurate — if a facade can read `total_cents`, the mapper is decoration.

### Things that hold state over time

Grouped by lifetime, which is the only question that matters here.

| Block | Suffix | Description |
|---|---|---|
| Store | `.store.ts` | Shared client state — signals service or NgRx SignalStore, provided in root |
| Flow | `.flow.ts` | Task state spanning several routes; provided in a route subtree, dies on exit |
| Connector | `.connector.ts` | Cross-store or store-to-service `effect()` reactions; disposes via `DestroyRef` |
| Realtime connector | `.realtime.ts` | Scoped subscription that reloads resources; disposes via `DestroyRef` |
| Integration events | `.events.ts` | Typed broadcast payloads only; imports from no lib |
| Injection token | `.token.ts` | `InjectionToken<T>` with a factory — config, environment, notification sinks |

Store versus flow is a lifetime choice, not a size one: promoting flow state to a root store
makes it outlive the task and leak between sessions. DI scoping is the disposal mechanism.

### Things the router calls

Angular invokes these, you do not. All functions, never classes.

| Block | Suffix | Description |
|---|---|---|
| Routes fragment | `.routes.ts` | Lib `Routes` array plus its route-level `providers` (DI scope) |
| Route resolver | `.resolver.ts` | `ResolveFn` that warms resources or prefetches; no component knowledge |
| Route guard | `.guard.ts` | `CanActivateFn` reading session synchronously, calling a policy, redirecting |
| Path builders | `.paths.ts` | Typed link builders; leaf module with no dependencies, importable by anyone |

Pages are listed under *Things that render*; the router mounts them, but they are components.
A class implementing `CanActivate` is legacy style and fails review. Render-level permission
is not a guard — that is `can.ui.ts` calling a policy.

### Things that are just TypeScript

Across the framework boundary. Import nothing — no `@angular/*`, no RxJS, no DI. Testable
without `TestBed`. This is the tier that survives a rewrite; the rest of the architecture
exists to keep it reachable and honest.

| Block | Suffix | Description |
|---|---|---|
| Domain model | `.model.ts` | Domain entity types and constructors |
| Domain logic | `.rules.ts` | Pure functions and state machines over domain types |
| Policy | `.policy.ts` | `(actor, resource) => boolean` predicates, shared by guards, facades, and `<app-can>` |
| Technical types | `.types.ts` | Product-agnostic generics (`Result<T>`, `AsyncState<T>`) in `util/` |
| Test fixture / fake | `.fixture.ts` / `.fake.ts` | Domain factories and fake services swapped in via providers |
| Generic pure function | *none* | One declaration per file in `util/`; the filename is the function name (`util/clamp.ts` exports `clamp`) |

No `@Injectable()` decorator appears anywhere in `domain/`. When domain logic wants a
dependency it takes it as an argument — Angular DI is convenient exactly where it must not be
used.

Read `references/blocks.md` for each block's contract: what it takes, returns, may inject,
and must never import. Consult it whenever creating an unfamiliar block type or when unsure
whether something belongs above or below the view-model seam.

## The six rules that carry the architecture

Everything else is convention. These are enforced and should never be worked around:

1. **`domain/` and `util/` import nothing.** No `@angular/*`, no RxJS, no vendor packages, no
   formatters, no schemas. No DI — plain functions and types only. This is what makes them
   survive rewrites — it is the most valuable rule here.
2. **Wire shapes stay in `services/`.** DTO types and generated database types are importable
   only by `.api.ts`, `.mapper.ts`, and `.schema.ts`. Parse and map at the service boundary so
   a contract violation surfaces once, with the endpoint in the stack trace.
3. **A component injects at most one facade, and no resource/mutation/store/api.** If a
   component needs one more field, add it to the view model — do not reach past the facade.
   Two facades in one component means the component reconciles them, which is business logic.
4. **A facade injects no other facade, no `.api.ts`, and no `HttpClient`.** It composes
   resources, mutations, stores, ambient services, and domain functions, then exposes a
   `Signal<Vm>` plus command methods.
5. **RxJS stays below the view-model seam.** Observables live in `services/` and connectors;
   the boundary converts (`toSignal`, `resource`) so components and facades never subscribe.
   `async` pipe in a template is a violation, not a technique.
6. **Cross-lib imports hit the public surface only, and the graph stays acyclic.** The
   surface is the lib root's files, or its `index.ts` under the barrel layout; everything
   deeper is private. If two libs need each other, one direction becomes an integration
   event.

## Components: the standing configuration

Every component in the codebase, regardless of kind:

- standalone (no NgModules), `changeDetection: ChangeDetectionStrategy.OnPush`
- signal inputs (`input()`, `input.required()`), `output()`, `model()` where two-way is real
- `inject()` at field initializers — constructors stay empty
- native control flow (`@if`, `@for` with `track`, `@switch`, `@defer`); no structural
  directive imports for flow control
- `host: {}` metadata, not `@HostBinding`/`@HostListener`
- zoneless-compatible: no code path relies on Zone.js scheduling

These are lintable with `angular-eslint` and are part of the architecture, not style
preference — OnPush + signals is what makes "component renders the VM and nothing else"
mechanically true.

## View models resolve every decision

The test for whether a facade is doing its job: the component's template contains no
conditional that depends on domain state. Resolve them into fields.

```ts
// checkout.vm.ts
export type CheckoutVm = {
  lines: { id: string; label: string; priceLabel: string }[];
  totalLabel: string;              // formatted, not a number
  canSubmit: boolean;              // not (user, cart) for the template to evaluate
  submitBlockedReason: string | null;
};
```

```ts
// facades/checkout.facade.ts — provided in the route, exposes one signal
@Injectable()
export class CheckoutFacade {
  private cart = injectCartResource();
  private session = inject(SessionStore);
  private locale = inject(LOCALE_ID);

  readonly vm: Signal<CheckoutVm> = computed(() => {
    const cart = this.cart.value();
    return {
      lines: cart.lines.map((l) => toLineVm(l, this.locale)),
      totalLabel: formatMoney(cartTotal(cart), this.locale),
      canSubmit: canCheckout(this.session.actor(), cart),
      submitBlockedReason: checkoutBlockedReason(this.session.actor(), cart),
    };
  });

  submit(): void { /* delegate to mutation */ }
}
```

`priceLabel` not `price`. `canSubmit` not `user.tier`. When a template writes
`@if (user().tier === 'pro' && cart().items.length)`, logic has leaked upward — move it into
a `.rules.ts` function and expose the result as a boolean.

Commands are **methods on the facade**, not fields on the VM. The VM stays a plain
serializable type; the component calls `facade.submit()`. This is the one deliberate departure
from callback-carrying view models — Angular components hold the facade reference anyway.

Formatting happens in `.format.ts` files at the facade tier, and takes locale as an explicit
argument so it stays pure and testable. Pipes are reserved for generic, product-agnostic
transforms (`.pipe.ts` in `ui/`); domain formatting never becomes a pipe, because pipes hide
formatting decisions in templates where no lint rule can see the domain coupling.

## Lib layout

A **lib** is one unit of the app — a directory with a declared public surface, a private
interior, and its own path alias. It is lib-*shaped*, not a built package: one app, one
build, no `project.json`. What it borrows from a real library is the boundary, not the
packaging — consumers import `@app/libs/cart`, never a path into it.

**A lib is a unit of encapsulation, not of reuse.** Most libs have exactly one consumer (the
router) and are never imported by another lib at all — that is the normal case, not a sign
something is wrong. "Lib" here says *this code has a surface and an interior*; it says nothing
about how many callers it has. Code meant to be reused by many libs does not stay in one: it
moves down to `ui/`, `util/`, or `domain/` (see *Shared code starts local*).

That alias is the point. `@app/libs/cart/cart.resource` names the lib and its public module;
nothing outside can reach `internal/` without saying so in the specifier, which is what makes
the crossing visible in review and matchable by a lint rule.

Deliberately *not* included: independent buildability, versioning, and per-lib `tsconfig`
references. Those turn promotion (see *Shared code starts local*) from a file move into a new
package, and people duplicate rather than pay that — the opposite of what the rule is for.

Two layouts express the surface; pick one **per app**, never per lib.

**Path-depth (default)** — public files at the lib root, private ones in subdirectories.
Depth is the boundary; no extra file, and eslint alone enforces it.

```
libs/cart/
  cart.resource.ts       public — injectCartResource()
  cart.mutation.ts       public — bundled commands, stable identity
  cart.paths.ts          public — typed link builders, zero dependencies
  cart.routes.ts         public — Routes, lazy-loaded by the shell
  cart.events.ts         public — typed integration events
  components/            private — containers and lib-local .ui.ts
  facades/               private
  internal/              private — .vm.ts, form schemas, lib-local .format.ts
```

**Barrel (Nx-style)** — one `index.ts` at the lib root declares the surface explicitly;
everything else, root files included, is private. It **requires Sheriff or
dependency-cruiser**: `no-restricted-imports` matches specifier strings, so a barrel makes
every suffix import rule match nothing rather than error. Choosing it without resolved-path
analysis turns the enforcement off silently.

Simple libs need only `internal/`. Add `components/` and `facades/` when there is more
than one thing to put in them — a subdirectory holding one file is filing, not structure.

Read `references/layouts.md` before setting up a repo or moving a file across the boundary:
it has both trees, the tradeoff table, the failure mode of each, and the migration path.

Cross-lib imports are **absolute**; same-lib imports are **relative**. That makes
every boundary crossing visible in the import statement itself, and in code review, without
opening the file tree.

```ts
import { CartLines } from './components/cart-lines.container';       // internal
import { injectCartResource } from '@app/libs/cart/cart.resource'; // crossing, public only
```

Each lib's `.routes.ts` is the lazy boundary: the shell references libs only through
`loadChildren` pointing at route files and through `.paths.ts` — so lib code stays out of
the initial bundle by construction.

### Shared code starts local

`libs/` is a flat list. There is no domain or area level above it, which means the global
`ui/` and `util/` folders are the *only* shared tiers — and that is exactly why nothing lands
there by default.

**A block is born inside the lib that needs it and moves down only when a second,
unrelated lib needs it too.** A `.ui.ts` used by one lib lives in that lib's
`components/`, not in global `ui/`. A `.format.ts` used by one screen lives in `internal/`.

| Consumers | Where it lives |
|---|---|
| One lib | that lib's `components/` or `internal/` |
| Two or more unrelated libs | global `ui/`, `util/`, or `domain/` |
| Two libs that are really one | merge the libs instead |

Promotion is a real edit — move the file, make it product-agnostic, and drop any lib
import it was relying on. That cost is the point: it is a decision, not a reflex, and it
happens when the second consumer appears rather than in anticipation of one.

Never promote in anticipation. A block in `ui/` with one consumer is worse than a duplicated
one: it reads as a contract the whole app may depend on, so the next change to it has to
consider callers that do not exist. Two similar `.ui.ts` files in two libs are cheap and
honest — they diverge independently, and if they stop diverging you merge them then.

The third row is the trap. If two libs constantly need the same non-trivial block, they
may be one lib that was split too early; check whether the same person requests changes
to both before promoting the block.

## Lib types

Block type is the file-level axis (suffix, eslint-enforced). **Lib type is the
directory-level axis** (Sheriff-enforced): a lib type says which block types a directory may
contain and which lib types it may import. The top-level folders are the types:

| Lib type | Directory | May contain |
|---|---|---|
| `feature` | `libs/*` | everything above the view-model seam plus its own `.resource` / `.mutation` / `.flow` / `.events` / lib-local `.ui` |
| `ui` | `ui/` | `.ui.ts` `.pipe.ts` `.adapter.ts` `.layout.ts` |
| `data-access` | `services/` | `.api.ts` `.schema.ts` `.mapper.ts` `.interceptor.ts` + generated types |
| `domain` | `domain/` | `.model.ts` `.rules.ts` `.policy.ts` |
| `util` | `util/` | `.types.ts` + no-suffix pure functions |
| `testing` | `testing/` | `.fixture.ts` `.fake.ts` shared across libs |
| *(shell)* | `app/` | ambient services, `.token.ts`, session store, global connectors, app config |

Dependency matrix — a type may import only what its row lists:

| From \ may import | feature | ui | data-access | domain | util | testing |
|---|---|---|---|---|---|---|
| `feature` | public surface only | yes | yes | yes | yes | no |
| `ui` | — | — | — | — | yes | no |
| `data-access` | — | — | — | yes | yes | no |
| `domain` | — | — | — | — | yes | no |
| `util` | — | — | — | — | — | no |
| `testing` | yes | yes | yes | yes | yes | — |

The matrix is triangular — that *is* the framework boundary, drawn at directory
granularity: every row can reach `util/`, nothing can reach back up. The two
enforcement layers divide cleanly: Sheriff checks this matrix on resolved paths (survives
barrels, catches cycles); the eslint suffix globs narrow each permitted edge to specific
block types (Sheriff allows `feature → data-access`; the suffix rule restricts it to
`.resource`/`.mutation` touching `.api`).

**`testing` is deliberately inverted**: it may import every other type, and nothing imports
it except `*.spec.ts` files — enforced by lint, or it becomes a legal smuggling route between
any two types. Fakes used by only one lib's specs stay beside what they fake; `testing/` is
the promotion target, under the same second-consumer rule as everything else.

**There is no `state` lib type.** State blocks encode lifetime by *location* — `.flow` in its
lib (route subtree), owned `.store` at its lib root, ambient state in `app/` — and a shared
`state/` directory would erase exactly that information. When several libs need the same
store or flow, see the sharing ladder in `references/blocks.md`.

## Routing, guards, policies

These are three blocks, not one, and the split matters:

- `.policy.ts` — pure predicate over `(actor, resource)`, lives in `domain/`, no Angular
- `.guard.ts` — functional `CanActivateFn`; reads session synchronously, calls a policy,
  returns `true` or a `RedirectCommand`/`UrlTree`
- `.resolver.ts` — functional `ResolveFn`; warms resources or prefetches so the page does not
  waterfall; no component knowledge

The same policy is called from a guard (router context), a facade (VM boolean), and a
render-level `<app-can>` UI component. If the check lives inside the guard, the
disabled-button state and the route protection drift apart. Client policies are UX-only — the
server is always authoritative.

Never import another lib's page or routes to build a link. Import its `.paths.ts`, which
has no dependencies and can sit at the bottom of the graph. Route-param access uses
`withComponentInputBinding()` — pages declare `input()` for params rather than injecting
`ActivatedRoute`, which keeps pages testable as plain components.

## Scoped state: flows via route providers

A flow is task state spanning several routes (checkout steps, an onboarding wizard). Angular's
DI makes the scoping free: provide the flow store in the route subtree, and it is created on
entry and destroyed on exit — no provider component, no manual disposal.

```ts
// checkout.routes.ts
export const checkoutRoutes: Routes = [{
  path: '',
  providers: [CheckoutFlow],           // one instance per visit to the subtree
  children: [ /* step routes, each with its own facade */ ],
}];
```

The flow class (`.flow.ts`) is signals-in-a-service; cleanup registers with `DestroyRef`.
Promoting flow state to a root-provided store is the classic leak — it outlives the task and
bleeds between sessions.

Facades are provided the same way: in the route that mounts the page, so facade lifetime
equals screen lifetime and two visits never share stale state.

## No barrel files

Barrels defeat suffix-based lint rules, because `no-restricted-imports` matches the specifier
string rather than the resolved path. Under the default path-depth layout, do not create
`index.ts` re-export files anywhere — the lib root's real files *are* the public surface.

Under the barrel layout the ban narrows rather than disappearing: **one `index.ts` per lib
root, none below it**, and Sheriff or dependency-cruiser must be in place to restore what the
barrel costs. Subdirectory barrels are banned under both layouts — they hide layering
violations and buy nothing. See `references/layouts.md`.

## One declaration per file

The suffix names what the file *is*, so a file with two exports of different kinds has no
honest name — this rule is what keeps the taxonomy one-to-one with the filesystem, and it is
why `cart.resource.ts` can be reasoned about without opening it.

One exported declaration per file, with four standing exceptions:

- A type describing a single declaration in the same file (kept unexported)
- A Zod schema plus its inferred type (`.schema.ts` — the pair is the block)
- Mutually recursive declarations
- **Blocks whose contract is a bundle:** `.mutation.ts` exports one factory returning several
  commands, `.paths.ts` exports one builder object, `.events.ts` exports one payload map.
  Each is a single exported declaration; the plurality is inside it. A `.mutation.ts`
  exporting two factories is still a violation.

Filename equals the exported declaration in kebab-case, minus the suffix:
`inject{Cart}Resource` → `cart.resource.ts`, `clamp` → `util/clamp.ts`.

This one is **review-only** — no lint rule expresses "the export matches the filename", and
`no-restricted-imports` cannot count exports. It is listed in the review checklist below for
that reason.

## Vendor integration

The built-in `resource`/`httpResource` APIs fill `.resource.ts`. NgRx SignalStore fills
`.store.ts` / `.flow.ts` when plain signal services are not enough. `HttpClient` is confined
to `.api.ts` and `.interceptor.ts`. Supabase fills `services/` plus `.realtime.ts`. Each is
confined to a named set of suffixes so the cost of replacing it equals the number of files
matching one glob.

Read `references/vendors.md` before writing any file that touches these — it covers
resource factories shared with resolvers, RxJS interop rules, SignalStore boundaries, session
hydration ordering with `provideAppInitializer` (a common source of spurious redirects), and
when Supabase's generated types remove the need for a runtime schema.

## Enforcement

Every enforcement tool is configured in one place: `references/lint.md`. The other reference
files describe what a rule protects; that one is where the rule is written.

`references/lint.md` has the complete flat config (angular-eslint + `no-restricted-imports`
per suffix glob), plus the Sheriff / dependency-cruiser configuration for the checks globs
cannot express (cycles, resolved-path privacy). Add or update the relevant rule whenever
introducing a new block type — a suffix with no rule behind it is documentation, not
architecture, and will erode.

When a violation is genuinely warranted, use an explicit `// arch-exempt: <reason>` comment
rather than restructuring code to slip past a rule. A handful of greppable exemptions is a
healthy system; zero usually means people are routing around the boundaries invisibly.

### Check on every iteration, not only in CI

CI is the wrong loop length when an agent is writing the code. A violation caught at CI has
already been built on for a dozen edits; caught after the iteration that introduced it, it is
one file and the agent still has the reasoning in context.

Run the fast checks on a **Stop hook** — after each agent turn, not on commit:

```jsonc
// .claude/settings.json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "npx tsc --noEmit && npx eslint src --quiet"
      }]
    }]
  }
}
```

Keep it under a few seconds or it stops being run. `tsc --noEmit` plus eslint covers the
suffix import rules and the derived-input renames; leave Sheriff, dependency-cruiser, and the
grep gates in CI where a slower pass is affordable.

Violation messages are the interface here. `no-restricted-imports` with an explicit `message`
per pattern (as in `references/lint.md`) tells an agent *which* boundary it crossed and what
to do instead — "Go through a resource or mutation" is actionable; "restricted import" is not.
Write the message for the reader who has to fix it without opening this skill.

This closes the loop earlier but does not add coverage. Anything with no rule behind it —
"guards are functions, not classes", suffix-matches-export — is still review-only no matter
how often the hook runs.

## Reviewing existing code

When asked to review a diff or refactor, check in this order — the first three catch most
real violations:

1. Any component injecting `HttpClient`, a store, or anything from a `.resource.ts` /
   `.mutation.ts` / `.api.ts`
2. Any template conditional that reads domain state rather than a view-model boolean; any
   `async` pipe or `.subscribe(` above the view-model seam
3. Any DTO field name (snake_case, `_id`, `_at`) appearing above `services/`
4. Cross-lib imports reaching into subdirectories
5. Files whose suffix does not match what they actually export; components missing OnPush or
   using decorator-based inputs
6. Files exporting more than one declaration outside the four exceptions, or whose filename
   does not match their export; guards or resolvers written as classes rather than functions
   — none of these are lintable, so review is the only gate
