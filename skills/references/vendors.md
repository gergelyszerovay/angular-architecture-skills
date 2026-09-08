# Vendor integration

Preserve the installed stack unless the task includes changing it. Verify installed
versions and official APIs before implementation. Containment limits coupling; it does
not guarantee that replacing a vendor is mechanical.

| Integration | Owning blocks |
| --- | --- |
| HTTP, tRPC, generated clients | Data-access APIs and private transport/configuration |
| RxJS | APIs, resource/command interop, connectors, lower-level ambient integration |
| Signal-store implementations | Stores and flows |
| Router | Routes, guards, resolvers, composition, navigation facades/connectors |
| Schema validation | Boundary schemas and their validation owners |
| Imperative editors/charts | Rendering adapters |
| Forms | Interaction owners and presentational controls appropriate to their contract |

Components/facades use signals rather than observable subscriptions. Convert at the actual
owner. Keep raw vendor clients out of public capability contracts. Configuration wiring
may import provider APIs.

## Queries and commands

Resources delegate transport and wire interpretation to data access. A direct HTTP resource
handling raw responses belongs inside that boundary and exposes owned results upward.

Resolvers and facades share a cache/resource owner if they intend to share a load.
Calling the same resource factory twice does not itself share results.

Vendor mutation primitives implement command semantics. Use .command.ts for new blocks
even if the vendor calls its API a mutation. Choose overlap/retry behavior per operation;
do not wrap independent actions in a single pending flag.

Forward cancellation where supported and separately check result validity before publication.
See [state-and-operations.md](state-and-operations.md).

## State and subscriptions

Plain signal services and the existing store library are valid implementations. Scope
follows ownership, not vendor defaults. Avoid mirroring a resource cache into another
writable store; applied results and optimistic overlays need explicit distinct contracts.

Keep subscription clients and raw payload interpretation inside data access. Scoped
connectors consume owned events and invalidate or command the relevant capability.
Dispose on scope exit and identity changes.

Authentication requires explicit pending-identity behavior. Bootstrap hydration or async
guards can implement it; guards are not inherently synchronous.

## Validation

Generated TypeScript types do not validate runtime responses. Choose validation according
to actual external contract/trust guarantees. JSON and untyped integrations need explicit
interpretation. Mapping is required when semantics differ; a separate mapper file is
optional for trivial translations. Never re-export generated DTOs merely because their
fields appear suitable.

Client policies govern UX. Backend/database authorization remains authoritative; verify
agreement where behavior matters.
