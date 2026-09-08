# Capability architecture

## Library roles and permitted dependencies

| Role | Responsibility | Permitted dependencies |
| --- | --- | --- |
| Shell | Bootstrap and global wiring | Composition, provider/configuration entry points, support libraries |
| Composition | Assemble capabilities into a page, route subtree, dialog, or embedded UI host | Capabilities, UI, domain, utilities |
| Capability | Own a cohesive interaction or application responsibility | Explicitly allowed capabilities, data access, domain, UI, utilities |
| Data access | Isolate transport and interpret external contracts | Domain, utilities, explicitly allowed lower-level transport modules |
| Domain | Model concepts and calculate outcomes | Explicit acyclic domain dependencies and approved pure libraries |
| UI | Product-independent presentation | Approved UI and utilities |
| Utility | Product-independent pure computation | Approved pure dependencies |
| Testing | Fixtures and fakes | Subjects and their dependencies; production cannot import testing |

Register actual library roots and roles, including grouped and generated roots. The table
is an upper bound: block restrictions also apply. Keep type-only edges acyclic too.
Directory names alone do not establish a role.

Composition is the generic role; route composition is one case. A workflow coordinates
steps or operations toward an outcome and may justify a flow block. A workspace is a
particular interface pattern with cooperating tools or panels, not a required library role.
Not every composition or capability needs a workflow or a flow block.

Capabilities may have no rendering and no routes. Do not split every component or endpoint
into a library. Extract for cohesive responsibility, ownership, or independent change
patterns even with one consumer. Promote generic helpers and UI after actual unrelated
consumers demonstrate a common contract; do not create empty anticipated libraries.

## Public contracts

Expose deliberate components, reads, intent-based commands, domain types, and scoped
provider functions. Keep writable signals, transport clients, endpoint tokens, and
implementation details private by default. Use named exports, not wildcard re-exports.

A barrel cannot authorize forbidden block consumption. Check the originating declaration
as well as the library edge. Public visibility also does not guarantee DI availability:
document the composition/provider owner and where consumers must be mounted.

Composition imports capabilities; capabilities must not import their composition owner
for shared state. Place shared ownership in an appropriate capability or supply a narrow
contract through composition. Infrastructure needed below the shell belongs behind a
lower-level contract, not a reverse import of app configuration.

## Facade granularity and coordination

One component consumes at most one facade; a screen may contain many. Each facade owns
one cohesive interaction area's VM and commands. Reuse is not a prerequisite, and each
child component does not automatically need a facade.

For example, a code workspace can host selection, tree, source, and annotation containers. Their facades
observe shared applied analysis through lower-level contracts. They do not inject each
other or copy shared state. A workspace facade handles only workspace-level presentation.

Choose integration deliberately:

- Direct command: a known consumer requests an action and needs an outcome.
- Read contract: a consumer derives from the authoritative owner.
- Flow: a use case coordinates operations and owns their combined outcome.
- Event: a fact occurred and independent listeners may react.

Events do not cure circular ownership. Check runtime feedback loops as well as imports;
define delivery scope, ordering, and failure semantics where relevant.

Preserve existing contracts unless the task includes changing them. Update callers and
enforcement together. Do not perform a broader migration as an implicit architecture fix.
