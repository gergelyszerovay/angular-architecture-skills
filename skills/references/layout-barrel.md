# Barrel / Nx-like layout

Each registered library has one root index.ts declaring named public exports.
Every other source file is private.

```text
libs/snapshot-selection/
  index.ts
  src/
    snapshot-selection.container.ts
    snapshot-selection.facade.ts
    snapshot-selection.command.ts
    snapshot-selection.store.ts
```

Cross-library imports name only the public alias, such as @app/libs/snapshot-selection.
Same-library imports are relative and bypass the barrel. No nested barrels or wildcard
re-exports. Apply [declarations.md](declarations.md) to implementation files, resolving
supporting-declaration consumers through barrels to their actual implementations.

Route-owning libraries export routes by name; lazy imports use the public barrel.
Deep route imports are not an exception. Capabilities need no routes.

This layout does not require Nx, project.json, independent builds, or workspace packages.
Register roots and roles explicitly, including grouping and generated roots.

Specifier suffix rules cannot identify a named barrel export's originating block.
Require resolved graph analysis plus declaration/re-export resolution where needed.
Installing Sheriff or dependency-cruiser alone does not establish coverage.

Verify forbidden component-to-store and facade-to-facade imports through barrels fail,
while legal container-to-facade imports pass. Banning every transitive store dependency
would incorrectly reject that legal composition.

Typed path builders must remain free of implementation dependencies. When eager builders
share a barrel with lazy routes, verify actual bundling rather than assuming isolation.
See [lint.md](lint.md).
