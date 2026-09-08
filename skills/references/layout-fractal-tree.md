# Fractal-tree layout

Use the repository's authoritative specification and installed validator. Private placement
follows dependency relationships; project configuration declares feature public surfaces.

Before edits, read applicable project rules and the actual tooling contract. In this
repository, use layered-documentation-reading before reading
root-layered-docs/1.architecture/1.index.md and its applicable linked specs.

A characteristic shape is:

```text
feature/
  operation.command.ts
  operation.command/
    private-helper.ts
  _shared/
    shared-helper.ts
```

Exact entry points, allowed imports, public declarations, shared placement, and
declaration-per-file requirements come from project specs. A _shared directory does not
automatically authorize arbitrary cross-library imports.

Block responsibility and dependency position are separate. Domain, data-access, and UI
files are not exempt from placement. Apply [declarations.md](declarations.md) and the
authoritative package rules, including physical parent/child eligibility for D2.

Use a compatible existing generator when restructuring calls for one. Do not assume it
understands Angular metadata or that every edit requires generation. Verify templates,
styles, DI references, tests, aliases, and public declarations survive moves. Do not invent
flags or ignore directives to make placement pass.

Run the relevant structure validator. If a generator is used, verify its second pass is
stable. Separately check block dependencies, wire isolation, and provider contracts.
A structurally valid tree does not prove Angular architectural correctness.

When converting away, inventory public names first: root files/shared helpers can have
different visibility in the target layout. Preserve intended contracts explicitly.
