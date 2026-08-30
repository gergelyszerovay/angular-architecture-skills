# Vendor integration

Contents:
- [Containment table](#containment-table)
- [RxJS is a vendor here](#rxjs-is-a-vendor-here)
- [The resource API](#the-resource-api)
- [Mutations](#mutations)
- [NgRx SignalStore](#ngrx-signalstore)
- [TanStack Query as an alternative](#tanstack-query-as-an-alternative)
- [Supabase](#supabase)
- [What is actually swappable](#what-is-actually-swappable)

The architecture is a taxonomy of responsibilities; these libraries are implementations of
some of them. None of them displaces a block — each fills one.

"Flexible" has exactly one operational definition here: **each vendor package is importable
only from a named set of suffixes.** Count the files matching those globs and you have the
true cost of replacing it.

## Containment table

| Package | Importable from | Never from |
|---|---|---|
| `@angular/common/http` | `*.api.ts`, `*.interceptor.ts`, `app.config.ts` | everything above `services/` |
| `rxjs` | `services/`, `*.connector.ts`, `*.realtime.ts`, interop edges in `app/` | components, facades, `*.vm.ts`, `domain/`, `util/` |
| `@ngrx/signals` | `*.store.ts`, `*.flow.ts` | everything else |
| `@supabase/supabase-js` | `supabase.client.ts`, `*.api.ts`, `*.realtime.ts`, `session.store.ts` | everything above `services/` |
| `@angular/router` | `*.routes.ts`, `*.guard.ts`, `*.resolver.ts`, `*.page.ts`, `*.layout.ts`, `*.facade.ts` | `*.ui.ts`, `domain/`, `util/` |
| `zod` | `*.schema.ts` | everything else |
| `@angular/forms` | `*.container.ts`, `*.page.ts`, `internal/*.schema.ts` | `*.ui.ts` leaves receive `FormControl` via input, do not build forms |

The `@angular/router` allowance for facades is a deliberate compromise: facades need
`Router.navigate` for post-submit redirects. Injecting navigation as an abstract port would
keep the seam perfectly clean but costs more ceremony than it returns. Keep the ban strict
below the seam.

## RxJS is a vendor here

The biggest cultural shift from classic Angular: RxJS is treated as a data-tier
implementation detail, not the application's lingua franca.

- `.api.ts` may use operators freely; what it *returns* is a `Promise` or a cold observable
  consumed only by a `resource` loader.
- Ambient push streams (router events, auth state) convert once, at the edge:
  `toSignal(stream, { initialValue })` inside the ambient service — never in a component.
- No `async` pipe anywhere. If a template needs an observable's value, the boundary was
  drawn wrong; fix the boundary, not the template.
- No `Subject` as a poor man's event bus above the seam — cross-lib signaling is
  `.events.ts` plus the one bus service.

This makes the RxJS glob small enough to audit by hand, and makes "learn RxJS" optional for
anyone working above the seam.

## The resource API

`resource` / `httpResource` fill `.resource.ts`. Two patterns matter.

**Reactive params, service-owned fetching.** The loader delegates to `.api.ts` so parsing
and mapping stay at the service boundary:

```ts
// libs/cart/cart.resource.ts — public
export function injectCartResource(userId: Signal<string>) {
  const api = inject(CartApi);
  return resource({
    params: () => ({ userId: userId() }),
    loader: ({ params, abortSignal }) => api.fetch(params.userId, abortSignal),
  });
}
```

`httpResource` is the shortcut when there is genuinely no mapping — which rule 2 makes rare.
If you find yourself passing `httpResource` a `map` that renames fields, you have written a
mapper in the wrong file.

**Status collapses in the facade, not the component.** A resource exposes `value`, `status`,
`error`. The facade folds them into VM fields (`isLoading`, `errorMessage`, or a
discriminated `state` field) — components never touch `ResourceStatus`. Where a route
subtree wants suspense-like behavior, `@defer` blocks in the page plus a resolver that warms
the resource replace boundary components; the placement of `@defer` and error branches in
pages and layouts is the entire loading design.

**Resolvers warm, facades own.** A `ResolveFn` that triggers the same store/resource the
facade reads gives navigation-time prefetch without duplicating fetch logic. The resolver
returns nothing of interest; its job is timing.

## Mutations

There is no vendor filling `.mutation.ts` — it is a small standing pattern:

```ts
// libs/cart/cart.mutation.ts — public
export function injectCartMutations(reload: () => void) {
  const api = inject(CartApi);
  const status = signal<'idle' | 'pending' | 'error'>('idle');

  const wrap = <A extends unknown[]>(fn: (...a: A) => Promise<unknown>) =>
    async (...a: A) => {
      status.set('pending');
      try { await fn(...a); reload(); status.set('idle'); }
      catch (e) { status.set('error'); throw e; }
    };

  return {
    status: status.asReadonly(),
    add: wrap(api.add.bind(api)),
    clear: wrap(api.clear.bind(api)),
  };
}
```

Bundle the actions — identities are stable and callers usually want more than one. The
`reload` argument is the resource's `reload()`, passed by the facade that owns both; the
mutation file never imports the resource file, which keeps write and read acyclic.

## NgRx SignalStore

Plain signals-in-a-service covers most `.store.ts` needs. Reach for `@ngrx/signals` when a
store has enough derived state and methods that the conventions pay for themselves:

```ts
// libs/checkout/checkout.flow.ts — public, provided in the checkout route subtree
export const CheckoutFlow = signalStore(
  withState({ step: 0 as CheckoutStep, address: null as Address | null }),
  withComputed(({ step }) => ({
    isLastStep: computed(() => step() === LAST_CHECKOUT_STEP),
  })),
  withMethods((store) => ({
    next: () => patchState(store, { step: nextStep(store.step()) }),
  })),
);
```

This is the `signalStore` form of a flow; the plain `@Injectable()` class form is equivalent.
Start with the class and move to `signalStore` once derived state and methods accumulate — see
*Things that hold state over time* in `blocks.md` for both side by side.

Route-`providers` scoping applies unchanged — `signalStore()` without `providedIn: 'root'`
is exactly the scoped-and-disposed semantics a flow needs. Domain transitions (`nextStep`)
still live in `.rules.ts`; `withMethods` is wiring, not logic.

Do not use `withEntities`/store features to mirror server data that a resource already
caches — a store duplicating a resource is two sources of truth, and the store always loses.
Stores hold *client* state: selections, drafts, wizard position, optimistic overlays.

## TanStack Query as an alternative

`@tanstack/angular-query-experimental` can fill `.resource.ts` / `.mutation.ts` instead of
the built-ins, and brings cache invalidation, retries, and devtools. If chosen:

- `injectQuery`/`injectMutation` are confined to the same two suffixes plus `*.resolver.ts`
  (a resolver's whole job is to warm the cache, so it reaches the query factory directly —
  see the resolver row in `references/blocks.md`).
- Query options factories are declarations, so resolvers can `prefetchQuery` the same keys.
- Choose per app, not per lib — two caching regimes in one codebase means every reviewer
  must know which rules apply where.

The block contract is identical either way; that is the point of the suffix.

## Supabase

Supabase pushes on three blocks rather than slotting cleanly into one.

### Realtime revives connectors

Every subscribed table is a channel reloading a resource or store. These files are the only
ones permitted to import the Supabase client outside `services/`:

```ts
// libs/cart/cart.realtime.ts — scoped connector, instantiated by a facade or flow
export function connectCartRealtime(userId: string, reload: () => void) {
  const supabase = inject(SUPABASE_CLIENT);
  const destroyRef = inject(DestroyRef);

  const channel = supabase
    .channel(`cart:${userId}`)
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'cart_items', filter: `user_id=eq.${userId}` },
      () => reload(),
    )
    .subscribe();

  destroyRef.onDestroy(() => void supabase.removeChannel(channel));
}
```

Note the argument — this is a **scoped** connector; `DestroyRef` ties it to the providing
route or facade, so leaving the screen tears down the channel. Putting a per-user channel in
the global initializer set leaks it on logout, which is the standard Supabase bug.

Prefer `reload()` over patching state from a realtime payload. The payload is a raw row, so
writing it straight into a signal bypasses the mapper and puts wire shapes above `services/`.
Only patch directly after measuring that the extra round trip hurts, and then map the
payload explicitly.

### Session is a store, hydrated before the router

`onAuthStateChange` is push-based, and guards need a **synchronous** read — a `CanActivateFn`
cannot await `getSession()` on every navigation.

```
app/session/session.store.ts      signals; synchronous actor()
app/session/auth.connector.ts     onAuthStateChange → store (global connector)
app.config.ts                     provideAppInitializer(() => sessionStore.hydrate())
```

`provideAppInitializer` blocks bootstrap until the initial `getSession()` resolves. The
ordering is not optional: let the router start before the session resolves and every guard
sees `null` on the first pass, bouncing authenticated users to the login screen.

### RLS duplicates your policies

`canEditInvoice` now exists twice — once in `.policy.ts`, once as a Postgres policy. The rule
that keeps this sane: **the client policy is UX-only and the database is authoritative.**
Client policies disable buttons and hide routes; they never protect anything.

Write an integration test matrix asserting the two agree for each (actor, resource-state)
pair. They will drift, and a failing test is better than a support ticket.

### Generated types vs. Zod

Supabase generates a `Database` type from the schema, which overlaps `.schema.ts`. The split
is not where you would guess:

| Source | Runtime parse | Why |
|---|---|---|
| Table/view select | skip | Generated types are accurate; the DB enforces the shape |
| `jsonb` column | required | Typed as `Json`, no structural guarantee |
| RPC / Postgres function | required | Return shape is not checked by the generator |
| Edge function | required | Arbitrary TypeScript, no contract |
| Storage metadata | required | External shape |

So `.schema.ts` becomes conditional. `.mapper.ts` stays mandatory regardless — well-typed
snake_case is still snake_case.

Generated types live in `services/supabase/database.types.ts`, are never hand-edited, and
fall under the same import restriction as any other wire shape.

## What is actually swappable

This buys swap-ability at the **edges**, not semantic portability.

Replacing plain signal stores with NgRx SignalStore, or the resource API with TanStack
Query, is a mechanical rewrite of files matching one glob — the boundaries do not move and
no component changes. Even RxJS removal (as Angular's signal APIs grow) is glob-scoped,
because rule 5 already fenced it into `services/` and connectors.

Moving off Supabase is not a lint-scoped change. RLS pushed authorization into the database
and realtime shaped the reload strategy; those are architectural commitments living in
Postgres policies and connector design, not in an import statement. That is a reasonable
trade for what Supabase provides — it is just a different kind of reversible, and worth
knowing going in rather than discovering later.
