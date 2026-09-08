# Building blocks

Responsibility, library role, provider lifetime, and visibility are separate axes.

Composition is a library responsibility, not an additional block suffix. Pages, dialogs,
and other UI hosts can compose capability containers without owning a route or a flow.

## Rendering and presentation

| Block | Contract |
| --- | --- |
| .page.ts | Route entry and composition; hosts capability containers and consumes at most one facade of its own. |
| .container.ts | Connects an interaction area to at most one facade; passes inputs and forwards outputs. |
| .ui.ts | Presentational inputs/outputs and local visual state. May be product-specific. No capability-state injection. |
| .layout.ts | Arrangement and slots, not application workflow. |
| .dialog.ts | Typed opening data and closing result; at most one facade for its interaction. |
| .adapter.ts | Imperative rendering integration with input/output surface and widget cleanup. |
| .pipe.ts | Pure presentation transform, with dependencies appropriate to its library role. |
| .facade.ts | VM and commands for one cohesive interaction area; projection, formatting, and delegation. |
| .vm.ts | Display contract, including immutable interaction identifiers; no wire objects or writable state. |
| .format.ts | Extracted presentation transformation; locale is an explicit input where relevant. |

Rendering blocks may compose permitted components and use presentation types. They do not
consume APIs, resources, command blocks, stores, or flows. UI/adapters do not consume
facades. Smart rendering blocks may use presentation services, but those must not hide
capability state access.

Facades may consume resources, commands, stores, flows, domain functions, formatters, and
narrow environment/navigation services. They do not consume other facades, components,
transport APIs, HttpClient, or observable subscriptions. Commands are facade methods,
not callbacks stored in the VM.

Keep business decisions below templates. An empty list or expanded panel is an ordinary
visual condition. Simple label projection may stay in a facade. Immutable domain IDs or
enums may appear in a VM when suitable; avoid mechanical parallel type definitions.

## State and operations

| Block | Owns/exposes | Permitted lower-level collaborators |
| --- | --- | --- |
| .store.ts | Authoritative client state, read-only views, meaningful commands | Domain logic and explicit read contracts |
| .flow.ts | Use-case process, transitions, combined outcome; may own workflow state | Commands, resources, stores, domain logic |
| .resource.ts | Query identity, loading/error state, caching and reload behavior | APIs, domain logic, explicit query inputs |
| .command.ts | Explicit operation, outcome, concurrency/failure policy | APIs, domain logic, approved capability commands/state contracts |
| .connector.ts | External events translated into capability commands | Event sources, commands/read contracts, interop |
| .realtime.ts | Optional specialized subscription connector | Owned data-access event contracts and invalidation/commands |
| .events.ts | Facts and payload types; not requests or a mandatory global bus | Approved domain/contract types |
| .token.ts | Narrow injectable configuration/integration contract | Types appropriate to its owning tier |

A store owns state; a flow owns a process. Neither implies root or route scope. Stores
must not launch hidden network workflows. A flow needs a separate store only when that
store has independent ownership. Commands cannot depend back on the coordinating flow;
return candidate results to it. Legacy .mutation.ts uses command restrictions.

Read [state-and-operations.md](state-and-operations.md) for lifetime and consistency.

## Data access and pure logic

| Block | Contract |
| --- | --- |
| .api.ts | Endpoint invocation, external validation, owned results, private transport. |
| .schema.ts | Validation at a named boundary: wire schemas in data access, form/URL schemas with the input owner. |
| .mapper.ts | Substantive representation translation; a separate file is optional for trivial mappings. |
| .interceptor.ts | Transport concerns such as headers and transport errors. |
| .model.ts | Framework-independent domain concepts and values. |
| .rules.ts | Pure business calculations and transitions. |
| .policy.ts | Pure eligibility/permission decisions; server authorization remains authoritative. |
| .fixture.ts / .fake.ts | Test data/substitutes; local until shared use warrants extraction. |

Domain dependencies are passed as arguments, not injected. No framework, transport, or
presentation imports; explicit acyclic domain and approved pure dependencies are allowed.
Product-independent pure helpers need no mandatory suffix.

Wire isolation concerns ownership, not spelling. CamelCase does not prove isolation, and
snake_case does not prove a leak. Re-exporting generated DTOs under new names is not an
owned public contract.

## Routing

| Block | Contract |
| --- | --- |
| .routes.ts | Route composition/provider wiring; only route-owning libraries need one. |
| .paths.ts | Typed links without route/component implementation imports; pure ID types allowed. |
| .guard.ts | Functional access checks using identity/read contracts and domain policies. |
| .resolver.ts | Functional preparation using the same query/cache owner consumed after navigation. |

Resolvers must not create disposable duplicate resources and claim to warm a panel's cache.
Define whether preparation blocks navigation. Guards explicitly handle pending identity;
synchronous bootstrap hydration is one option.

Use one primary declaration per file with the explicit exceptions in
[declarations.md](declarations.md). Supporting declarations require the stated consumer
or coupled-pair relationship; subjective cohesion alone is not an exception.
