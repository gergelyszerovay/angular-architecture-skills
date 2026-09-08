# Path-depth layout

Public modules sit directly at registered library roots; descendants are private.

```text
libs/snapshot-selection/
  snapshot-selection.command.ts
  snapshot-selection.container.ts
  internal/
    snapshot-selection.facade.ts
    snapshot-selection.store.ts
```

Cross-library imports use a public module alias, for example
@app/libs/snapshot-selection/snapshot-selection.command. Same-library imports are relative.
No index barrels or wildcard re-exports.

Private pages, facades, and flows belong in the interior. Public route modules can import
private descendants; route providers do not require root/public placement.

Register roots explicitly. When discovering candidates beneath the configured libs root,
grouping directories contain no source and traversal stops at an established library.
Do not classify source directories inside a library as new libraries. Confirm candidates
against configuration and generated/test roots.

Enforce privacy through resolved ownership, not slash counts. Check relative imports,
aliases, explicit extensions, and re-exports. Public placement does not authorize forbidden
block consumption. Apply [declarations.md](declarations.md) for primary declarations,
supporting declarations, and the direct-consumer adaptation of D2.
