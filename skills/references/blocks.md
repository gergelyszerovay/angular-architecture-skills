# Block reference

Contents:
- [Things that render](#things-that-render)
- [Things that answer "what does this screen show?"](#things-that-answer-what-does-this-screen-show)
- [Things that talk to the server](#things-that-talk-to-the-server)
- [Things that hold state over time](#things-that-hold-state-over-time)
- [Things the router calls](#things-the-router-calls)
- [Things that are just TypeScript](#things-that-are-just-typescript)
- [Where things commonly go wrong](#where-things-commonly-go-wrong)

Groups match the suffix tables in `SKILL.md`. "Is it a service?" tells you nothing useful —
five different tiers show up as injectables. Classify by what a file exposes and what it may
inject.

A **lib** here is one unit of the app (`libs/cart/`) — a directory with a public surface and a
private interior. It is a unit of encapsulation, not of reuse: most libs are imported by
nobody but the router. So "reusable across libs: never" in the tables below is not a
contradiction — it means the block belongs to one lib's interior, and code genuinely meant for
many libs lives in `ui/`, `util/`, or `domain/` instead.

## Things that render

Have a template or touch the DOM. Never import `.api.ts`, stores, or resources.

| | `.ui.ts` | `.container.ts` | `.page.ts` / `.dialog.ts` | `.adapter.ts` |
|---|---|---|---|---|
| Takes | `input()` only | inputs (VM slice) | router inputs only | inputs |
| Internal state | UI-only (open, hover) | UI-only | UI-only | whatever the widget needs |
| Side effects | none | none of its own | none of its own | permitted |
| May import/inject | `ui/` siblings, tokens, i18n | `ui/`, one facade, ambient | one facade, layout pieces | its vendor package |
| Must never import | anything outside `ui/` | `.api.ts`, stores, resources | `.api.ts`, stores, resources | facades, stores |
| Lives in | `ui/` | `libs/<name>/components/` | `libs/<name>/` (page) / `components/` (dialog) | `ui/` |

Two more render blocks carry no injectable surface of their own:

| Block | Suffix | Contract |
|---|---|---|
| Layout | `.layout.ts` | Structural shell — chrome plus `<router-outlet>` or content slots; may open dialogs, holds no lib state |
| Generic pipe | `.pipe.ts` | `@Pipe({ standalone: true })`, pure, product-agnostic; lives in `ui/`, imports only `util/` |

All of them: standalone, OnPush, signal inputs/outputs, empty constructor, native control
flow. A `.ui.ts` component is a pure function of its inputs in the same sense a React
presentational component is — OnPush plus signals makes that mechanical, not aspirational.

`.dialog.ts` components are opened via the CDK `Dialog` (or `MatDialog`) with data passed
through `DIALOG_DATA` and a typed close result. Unlike a page, a dialog is *not* public by
default: it is opened by code inside its own lib, so it belongs in `components/` and only
moves to the lib root if another lib genuinely opens it. Only pages, layouts, and facades open
dialogs; a `.ui.ts` component asking to open a dialog does it by emitting an `output()`.

`.adapter.ts` exists for components that cannot be declarative — Stripe Elements, maps,
charts, rich text editors. They need `ElementRef`, `afterNextRender`, and imperative calls.
Naming them explicitly makes them an allowlist entry rather than a silent exception, and
`**/*.adapter.ts` greps every imperative component in the codebase in one command. An adapter
must still expose a signal-inputs-only surface to its consumers, and must register teardown
with `DestroyRef`. Adapters live directly in `ui/` like any other shared render block — the
suffix is what groups them, so `ui/adapters/` would be a subdirectory buying nothing.

`.pipe.ts` and `.format.ts` both format. The split is the product test: a pipe is
product-agnostic and reusable in any app; a formatter knows domain types and belongs to the
facade tier. A pipe that imports `domain/` is a formatter in the wrong file.

A `.ui.ts` starts in the lib that needs it (`libs/<name>/components/`) and moves to
global `ui/` only when a second, unrelated lib needs it. The `Lives in` row above is the
destination, not the starting point — see *Shared code starts local* in `SKILL.md`. A
component in `ui/` with one consumer reads as a contract the whole app may depend on, which
is more expensive than the duplicate it was meant to avoid.

For lib-local containers, derive inputs from the view model rather than declaring
parallel types:

```ts
lines = input.required<CheckoutVm['lines']>();
```

One source of truth, and a view-model rename surfaces as a type error everywhere.

## Things that answer "what does this screen show?"

The seam itself. Domain types in, display-ready values out. At most one facade per screen.

| | Ambient | Facade (`.facade.ts`) |
|---|---|---|
| Takes | nothing | route inputs |
| Exposes | environment signals | `Signal<Vm>` + commands |
| Named after | a capability | a screen or task |
| Side effects | none | delegates only |
| Formats for display | i18n only | yes |
| Screen-aware | no | yes |
| May inject/import | config tokens, i18n, session, sinks | resources + mutations + ambient + stores, domain logic, policies, formatters, `Router` (navigation only) |
| Must never import | libs, domain, stores | other facades, components, `.api.ts`, `HttpClient` |
| Provided in | root | the route that mounts the page |
| Per component | any number | at most 1 |
| Reusable across libs | always | never |
| Lives in | `app/`, `util/` | `libs/<name>/facades/` |
| Tested by | `TestBed` + fake providers | `TestBed.inject` + fakes, no template |

| Block | Suffix | Contract |
|---|---|---|
| View model | `.vm.ts` | Type of what one screen renders; display-ready fields only, no domain types |
| Formatter | `.format.ts` | Pure domain-type → display-string functions; locale passed in as an argument |

Two questions settle almost every case:

1. Does it expose `Money` or `"€42.00"`? Domain types mean below the seam; formatted strings
   mean it is a facade.
2. Could you use it on a screen that does not exist yet? Yes means reusable; no means use-case.

An injectable exposing formatted strings *and* usable anywhere is the failure case —
formatting escaped downward and will be duplicated the moment a second screen needs it
differently.

`Router` in the facade column is a deliberate compromise, not an oversight: facades need
`Router.navigate` for post-submit redirects, and injecting navigation as an abstract port
costs more ceremony than it returns. It is the one vendor allowance above the seam — see the
containment table in `references/vendors.md`. The ban stays strict below it.

Formatters are the only pure code above the seam: they know domain types *and* produce
display strings. That is exactly why they need a suffix — the "components never format" rule
depends on being able to state where formatting lives. Pass locale in as an argument (the
facade reads `LOCALE_ID` once) rather than reading it ambiently, or they stop being pure.
Angular's `formatDate`/`formatNumber` helpers may be *called from* `.format.ts` — that is a
pragmatic exemption, marked in the lint config, because reimplementing CLDR is worse.

### Facade granularity

One facade per **use case**, which usually means per page — not per component. Adjacent
components need overlapping slices of the world; giving each its own facade produces three
injectables reading the same cart, three loading states, and no single answer to "is this
screen ready?".

A child component earns its own facade only if at least two of these hold:
- It appears on multiple unrelated pages
- It owns a data lifecycle nobody else cares about
- It is droppable — deleting one template line removes it with no other edits

A comment thread widget passes all three. A cart line list on the checkout page passes none.

## Things that talk to the server

The whole read/write path, wire format to domain type.

| | Resource (`.resource.ts`) | Mutation (`.mutation.ts`) |
|---|---|---|
| Takes | domain ids (as signals) | nothing |
| Exposes | `Resource<DomainType>` | `{ run, status }` |
| Named after | a resource | an action verb |
| Side effects | fetch on demand | writes, on call |
| Formats for display | no | no |
| May inject/import | services, stores, mappers | services, resources (to reload), domain |
| Must never import | facades, components, formatters | facades, components |
| Provided in | root or lib route | root or lib route |
| Per component | 0 — via facade | 0 — via facade |
| Tested by | fake service | fake service + spy |

| Block | Suffix | Contract |
|---|---|---|
| Service | `.api.ts` | `@Injectable` class over `HttpClient`; parses DTO and maps to domain before returning |
| Wire schema | `.schema.ts` | Zod schema plus inferred DTO type; service-private |
| Mapper | `.mapper.ts` | `(dto) => DomainType`; the only translator of field names |
| Interceptor | `.interceptor.ts` | `HttpInterceptorFn` — auth headers, retries, error normalization |

`.api.ts` methods return `Promise<DomainType>` (or a cold observable consumed only by
`resource` loaders). RxJS operators are welcome inside; observables do not escape upward —
the resource layer is where async becomes signals.

The mapper is mandatory even when the wire types are accurate. Snake_case rows, `_id`
suffixes, and ISO date strings must never reach a facade — if a facade can read
`total_cents`, the mapper is decoration.

A form schema is never the wire schema. They validate different things — the wire schema
describes what the server sends, the form schema describes what a user may type. Both use
`.schema.ts`, disambiguated by location: `services/` versus the lib's `internal/`.
Reactive-forms `Validators` live with the form; cross-field business constraints they call
live in `.rules.ts`.

### Resource factories

`.resource.ts` exports **factory functions that require an injection context**, prefixed
`inject*` so call sites read honestly:

```ts
// libs/cart/cart.resource.ts — public
export function injectCartResource(userId: Signal<string>) {
  const api = inject(CartApi);
  return resource({
    params: () => ({ userId: userId() }),
    loader: ({ params }) => api.fetch(params.userId),  // service parses DTO, maps to domain
  });
}
```

The factory form (rather than a class per resource) keeps parameters reactive and lets a
facade own the resource's lifetime. A resource shared by many screens graduates to a
root-provided store that wraps it — sharing via module-level state is forbidden because it
never disposes and breaks test isolation.

## Things that hold state over time

Grouped by lifetime, which is the only question that matters here.

| Block | Suffix | Lifetime | Contract |
|---|---|---|---|
| Store | `.store.ts` | root, or the owning lib's route | Shared client state; signals service or NgRx SignalStore |
| Flow | `.flow.ts` | route subtree, dies on exit | Task state spanning several routes |
| Connector | `.connector.ts` | global or scoped | Cross-store or store-to-service reactions; disposes via `DestroyRef` |
| Realtime | `.realtime.ts` | scoped | Subscription that reloads resources; disposes via `DestroyRef` |
| Integration events | `.events.ts` | n/a — types only | Typed payloads; imports from no lib |
| Injection token | `.token.ts` | root | Config, environment, notification sinks — `InjectionToken<T>` with a factory |

Store versus flow is a **lifetime** choice, not a size one. Both hold client state. A store
outlives any single task: ambient state (session, flags, notifications) is provided in root
and lives for the whole session, while a store owned by one lib may be provided at that lib's
route and live as long as the lib is mounted. A flow is provided in a route *subtree* and is
destroyed on exit — that is the distinction, not the provider's depth. Promoting flow state to a root store makes it outlive the task and leak
between sessions — DI scoping is the disposal mechanism.

Neither is a component-level store. A flow deliberately outlives every component in the task:
navigating from cart to address destroys the cart page, and the flow survives because it is
provided on the parent route. Per-component state is local signals in the component (open,
hover) or belongs to the facade, which is already screen-scoped.

**Import-visible is not the same as injectable.** A flow lives at the lib root under the
path-depth layout because its `.routes.ts` provides it, which makes the class importable by
any lib. That is a layout artifact, not permission: DI decides who can actually inject it,
and only pages mounted inside the providing subtree can. Lint cannot express this, so it is
the one boundary here enforced at runtime — an outside injection fails with `NullInjectorError`
rather than a lint error. See *Path-depth layout* in `references/layouts.md`.

A **flow** belongs to the lib that owns its *outcome*, not the libs it touches.
Checkout spans cart, address, and payment, but it is `libs/checkout/checkout.flow.ts`,
provided in checkout's route subtree, injecting the other libs' public surfaces. The flow
is public (the routes file references it); a private `internal/` accessor wraps
`inject(CheckoutFlow)` if only some screens should read it. If another lib needs to know
a flow is in progress, that is an event.

### When several libs need the same store or flow

Sharing does not move the block. Walk the ladder, stop at the first rung that fits:

1. **One other lib, dependency acyclic** → import through the owner's public surface.
   `checkout` injecting cart's store is the documented normal case.
2. **Several libs, one owns the outcome** → still the owner's; everyone else goes through its
   public surface. Ownership is who requests changes to it, not who reads it.
3. **Many libs, no owner** (session, notifications, flags) → it was never lib state; it is
   ambient. Move it to `app/`.
4. **Sharing would create a cycle** → `.events.ts`; one direction becomes fire-and-forget.

For a **flow** the ladder sits under a hard constraint: it is provided in a route subtree, so
only pages mounted inside that subtree can inject it — DI enforces this at runtime, not just
lint. The two cases that feel like "shared flow":

- Pages from other libs mounted *inside* the owner's subtree → fine; inject via the owner's
  public surface. That is the checkout shape above.
- A lib *outside* the subtree wants the flow → it cannot, and should not. It wants either an
  event ("checkout in progress") or the data is not task-scoped — in which case it is a
  store, and calling it a flow was the mistake.

So sharing pressure never promotes a flow: the consumer joins the subtree, gets an event, or
the block is reclassified. And a store needed by many libs moves to `app/` only when
ownership genuinely dissolves — a shared `state/` dumping ground flattens the dependency
graph into a star, which is why no `state` lib type exists (see *Lib types* in `SKILL.md`).

A flow may be written either way, and both get identical route-`providers` scoping:

```ts
// plain @Injectable class — no extra dependency
@Injectable()
export class CheckoutFlow {
  private readonly _step = signal(0);
  readonly step = this._step.asReadonly();
  next(): void { this._step.update(nextStep); }
}

// signalStore — worth it once derived state and methods accumulate
export const CheckoutFlow = signalStore(
  withState({ step: 0 as CheckoutStep }),
  withComputed(({ step }) => ({ isLastStep: computed(() => step() === LAST_CHECKOUT_STEP) })),
  withMethods((store) => ({ next: () => patchState(store, { step: nextStep(store.step()) }) })),
);
```

Start with the class; move to `signalStore` when the conventions pay for themselves. Either
way the domain transition (`nextStep`) lives in `.rules.ts` — `withMethods` is wiring, not
logic.

Connectors come in two forms. **Global** ones take no arguments and start once via
`provideAppInitializer` (or an `ENVIRONMENT_INITIALIZER`) in `app.config.ts`. **Scoped** ones
take arguments (a user id, a document id) and are instantiated by a route-provided flow or
facade, cleaning up through `DestroyRef`. A per-user subscription in the global set leaks on
logout.

Cross-store reactions use `effect()` inside the connector, never inside a component — a
component with an `effect()` that writes state is a connector hiding above the seam.

**Public surface vs. integration event** — they solve different problems:

| | Public surface | Integration event |
|---|---|---|
| Direction | Known, one-way | Unknown, broadcast |
| Coupling | Consumer names the producer | Neither names the other |
| Timing | Synchronous read | Fire and forget |
| Use when | Dependency is acyclic | Would be cyclic, or N unknown listeners |

`checkout` calling `injectCartResource()` is fine — the dependency is real and stable. The
moment `cart` also needs to react to something in `checkout`, use an event, because a cycle
between libs is the failure mode this structure exists to prevent. The event bus itself
is one root service in `app/`; `.events.ts` files declare payload types only.

## Things the router calls

Angular invokes these, you do not. All functions, never classes.

| Block | Suffix | Exports | Rule |
|---|---|---|---|
| Routes fragment | `.routes.ts` | `Routes` | One per lib; the shell `loadChildren`s it |
| Resolver | `.resolver.ts` | a `ResolveFn` | Warms resources / prefetches; may import the resource or query it warms, never a component or facade |
| Guard | `.guard.ts` | a `CanActivateFn` | Reads session synchronously, calls a policy, redirects |
| Path builders | `.paths.ts` | typed functions | Leaf module — importable by anyone |

Pages and dialogs are components; their contracts are under *Things that render*. The router
mounts them via `withComponentInputBinding()`, and they take router inputs only.

Guards and resolvers are **functions**, not classes — `inject()` works inside them, they
compose, and they tree-shake. A class implementing `CanActivate` is legacy style and fails
review.

`.paths.ts` is the decoupling trick. A lib never imports another lib's routes or page
just to build a link — it imports `checkout.paths.ts` and calls `checkoutPaths.confirm(id)`.
That file has no dependencies, so it sits at the bottom of the graph next to `domain/`, and
every route string in the app is typed and defined once. Templates use
`[routerLink]="paths.confirm(id())"` with the builder exposed through the VM or a static
import — never a hand-assembled string.

The lib's `.routes.ts` also carries the lib's DI scope: route-level `providers` is
where facades and flows are registered, which is what gives them screen-scoped lifetime.

Render-level permission is not a guard. `<app-can [policy]="..." [resource]="...">` is
`can.ui.ts` — a UI component that calls a policy. Reserving `.guard.ts` for route-level
checks keeps the suffix meaning one thing.

## Things that are just TypeScript

Across the framework boundary — the stronger of the architecture's two lines. Everything here
imports nothing: no `@angular/*`, no RxJS, no DI. This is the code that survives every
framework and vendor decision, so protect the rule fiercely. (The tier between this and the
view-model seam — resources, mutations, stores, services — is Angular-coupled and makes no
such claim; see the framing at the top of `SKILL.md`.)

| Block | Suffix | Where | Contains |
|---|---|---|---|
| Model | `.model.ts` | `domain/` | Domain entity types and constructors |
| Logic | `.rules.ts` | `domain/` | Pure functions and state machines over domain types |
| Policy | `.policy.ts` | `domain/` | `(actor, resource) => boolean` predicates |
| Technical types | `.types.ts` | `util/` | Product-agnostic generics (`Result<T>`, `AsyncState<T>`) |
| Test fixtures | `.fixture.ts` / `.fake.ts` | beside what they fake; `testing/` once shared across libs | Domain factories and fake services (`provide: CartApi, useClass: FakeCartApi`) |
| Generic pure function | *none* | `util/` | One declaration per file; the filename is the function name |

```ts
// domain/invoice/invoice.policy.ts
export const canEditInvoice = (actor: Actor, invoice: Invoice) =>
  invoice.status === 'draft' && actor.permissions.includes('invoice:write');
```

Policies are called from three places that cannot share code otherwise: guards (router
context), facades (VM booleans), and a `<app-can>` UI component (render path). Keeping them
here is what stops the disabled-button state and the route protection from drifting apart.

Note what is *not* here: no `@Injectable()` decorator anywhere in `domain/`. The moment
domain logic wants a dependency, it takes it as an argument. This is the discipline that
keeps the tier framework-free — Angular DI is convenient exactly where it must not be used.

The `util/` test: could you publish this to npm without leaking anything about your product?
If yes it is `util/`, which sits below `domain/` and imports nothing from the app at all. A
`.util.ts` suffix there would be noise no lint rule would ever reference — a suffix earns its
place only when a rule, codemod, or glob uses it. Generic pure functions take no suffix:
`util/clamp.ts` exports `clamp`.

Wire shapes are the exception that is *not* here — they are inferred from `.schema.ts` and
stay in `services/`, under *Things that talk to the server*.

Fixtures promoted to `testing/` keep the same suffixes but flip the dependency rule: the
`testing` lib type may import every other type (fixtures need the real interfaces), and only
`*.spec.ts` files may import it back — see the lib-type matrix in `SKILL.md` and the merged `src/**` block in
`references/lint.md`. Production code importing a fixture is a smuggling route between types.

## Where things commonly go wrong

**Business logic in the facade.** A facade coordinates; it should be wiring plus shaping. If
its `computed()` contains a non-trivial calculation, extract it to `.rules.ts` and call it.
The test: could this computation be unit-tested without `TestBed`?

**A second facade injected into a component.** Symptoms are
`checkout.vm().canSubmit && !promo.vm().blocking` in a template. Merge the facades so the
combination is computed once by a domain function.

**`effect()` in a component.** Almost always one of: a connector hiding above the seam
(move to `.connector.ts`), derived state (should be `computed`), or a DOM concern (belongs in
an `.adapter.ts`). Legitimate component effects are rare enough to justify a comment each.

**`.subscribe(` above the seam.** A component or facade subscribing manages async by hand —
lifecycle bugs follow. Convert at the boundary: `resource` for reads, mutation status
signals for writes, `toSignal` for ambient streams.

**Constructor-driven fetching.** A service that fetches in its constructor fires on injection
order, not on need. Resources fetch on demand; stores expose explicit `load()` triggered by a
resolver or facade.

**Module-level singletons instead of DI.** `export const cartState = signal(...)` at module
scope never disposes, breaks SSR (state shared across requests), and cannot be faked in
tests. Everything stateful is provided.

**A resource that formats.** The moment `injectCartResource` returns `totalLabel`, every
screen is stuck with that format. Return `Money`; let each facade format it.

**Promoting flow state to a root store.** It then outlives the task and leaks between
sessions. Provide the flow in the route subtree instead — DI scoping is the disposal
mechanism.
