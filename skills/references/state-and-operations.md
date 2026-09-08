# State ownership and asynchronous operations

## State contracts

For each meaningful stateful block, document ownership and behavior in existing code/docs
at the level its complexity warrants. Do not mechanically create a document per block.

- Authoritative values, derived views, drafts, caches, and persisted values.
- Public reads and intent-based commands; who may change each value.
- Provider owner and lifetime: application, route, component instance, or operation.
- Cleanup/reset triggers, including identity changes and route reuse where applicable.
- Failure behavior and relationships with other owners.

Choose the narrowest scope covering actual consumers. Multiple panes can have independent
component-scoped capability instances. Sharing does not automatically imply root scope.
Verify router reuse/provider behavior rather than assuming navigation destroys injectors.
Dispose work at its actual owner.

Avoid mutable module-level singletons. Read-only signals do not make nested objects
immutable; protect published result objects too.

## Execution contracts

| Kind | Behavior |
| --- | --- |
| Query/resource | May rerun from input changes when repetition is appropriate; owns identity and result state. |
| Command | Runs explicitly, including writes, scans, materialization, and refresh; reports an outcome. |
| Subscription | Receives external changes within a declared lifetime; maps events or requests invalidation. |

HTTP method or backend naming does not determine the block. Expensive source materialization
may return data but still requires an explicit command.

Where overlap is possible, choose latest-wins, serialization, reject-while-busy,
deduplication by key, or independent operations by key. One pending boolean cannot represent
multiple independent requests accurately.

## Result publication

For a multi-operation load, specify inputs and generation/request identity, candidate
results, commit preconditions, publication boundary, failure visibility, and cleanup/retry.

Cancellation does not prove freshness. Reject stale completions even after requesting
abort. Cache identity includes every input affecting validity; live results may require
an observation or generation identity.

For a code workspace, selection owns drafts and an analysis flow owns the applied source
pair. Package results identify their applied generation. Tree/source facades derive views
from that owner. Search selects declarations through the navigation capability.

Atomic publication means atomic client visibility, not a distributed transaction.
Previously completed external side effects require their own recovery/cleanup contract.

Connectors adapt browser history, identity, or filesystem events into explicit commands.
Do not copy state between owners through a network of effects. Distinguish inbound URL
restoration from outbound updates to avoid loops. Persist stable identities rather than
process-local handles, and define when restored inputs become applied.
