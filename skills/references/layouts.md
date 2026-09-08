# Layout chooser

All three layouts implement the same capability and block contracts. Preserve the explicit
choice of the owning package; read only its selected layout guide.

| Layout | Public surface | Private placement | Guide |
| --- | --- | --- | --- |
| Path-depth | Modules at registered library roots | Author-selected directories | [Path-depth](layout-path-depth.md) |
| Barrel / Nx-like | Named exports in one root index.ts | Private implementation directory | [Barrel](layout-barrel.md) |
| Fractal-tree | Configured declarations under project rules | Dependency-derived placement | [Fractal-tree](layout-fractal-tree.md) |

Select one layout for authored code per package, not per app or individual library.
Different packages in the same application or repository may use different layouts.
Generated/vendor roots can retain their
required conventions and must be registered separately. Grouping directories are not libs.

Inspect project instructions, configuration, registered roots, aliases, and source together.
A random index.ts or generated Helm barrel does not determine the owning package's layout.
Interpret fractalTs according to installed tooling; not every configured package has libs.

For a new package without a preference, path-depth is lightweight. Barrel gives a compact
explicit API and stable imports. Fractal-tree supports dependency-derived placement when
compatible tooling and project rules exist.

Every layout needs cycle and resolved-boundary checks. Specifier regexes alone do not
prove privacy, especially with relative imports, grouped roots, and re-exports.

## Migration

Inventory intended exports before moving files. Update callers, aliases, provider wiring,
compiler/test inputs, docs, and enforcement together within the authorized scope.

Do not publish every root file automatically when introducing barrels. Do not assume
deleting fractal configuration produces valid path-depth visibility. Preserve intended
contracts explicitly. A broader repository migration must be part of the requested task.
