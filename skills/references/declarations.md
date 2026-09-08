# File and declaration policy

Use one primary declaration per implementation file, with the explicit supporting cases
below. This is not a literal limit of one AST declaration node. This policy applies
independently of directory layout; the selected layout governs placement and visibility.

## Primary declarations and permitted companions

- A function or class file has one primary function or class. Arrow functions and function
  expressions assigned to variables count as functions, not supporting constants.
- Supporting types, constants, and enums used only in that file stay inline and unexported.
  This does not permit unrelated private helper functions beside the primary function.
- A type-only or constant-only file exports exactly one declaration, except for a coupled
  pair. Supporting private declarations must be used only in that file or be the schema
  checks described below.
- Supporting declarations may be co-exported from their owning implementation when needed
  by that implementation and its direct consumer. A second exported function is not
  a supporting declaration.
- A constant and its associated type that cannot change independently stay together:
  an as-const value with its derived type, an enum with its derived union, or a schema with
  its hand-written type. Do not interpret arbitrary “tightly coupled code” as an exception.
- Two mutually recursive functions may share a file when separating them would create a
  circular import. Name the file after the primary entry. Normally only it is exported;
  export both only when both have external callers. Do not generalize this exception to an
  arbitrary collection of functions or classes.
- Each root-level let/const statement declares at most one variable. Multiple bindings
  are prohibited even if they would otherwise qualify as supporting constants.

Class methods and function-local declarations belong to their enclosing declaration.
A command factory returning several methods remains one primary function. Function overload
signatures describe the same function; a checker must resolve them with its implementation
rather than count them as independent functions.

## Consumers and layout

Consumer analysis follows the actual declaring symbol through aliases and re-exports;
type-only uses count. Importing a primary function does not automatically make that importer
a consumer of every supporting declaration in its file.

For a supporting co-export, the symbol is used in its declaring implementation and by one
direct consuming implementation. Other consumer patterns require a separate declaration
file in the owning library, or a deliberately shared owner if needed across libraries.
Public re-export plumbing is not an additional implementation consumer. This ownership
test does not prescribe parent/child directories; apply the selected layout's placement
rules separately.

Public visibility remains layout-specific. A supporting export does not authorize a deep
import; expose it through the permitted public surface or keep its consumers internal.
Any declaration sharing must also respect block dependencies and wire isolation.

## Schema/type pairs

Use a type-first policy: hand-write the type, then its schema in the same
file. Do not use z.infer as the primary domain type definition. Include the two unexported
compatibility assignments below; they are permitted supporting declarations:

```ts
export type OrderItem = { id: string };
export const OrderItemSchema = z.object({ id: z.string() });
const _exhaustive: z.infer<typeof OrderItemSchema> = {} as OrderItem;
const _reverse: OrderItem = {} as z.infer<typeof OrderItemSchema>;
```

This fragment illustrates declaration grouping; imports and required documentation are
omitted. For recursive Zod schemas, use z.lazy() and annotate the schema with z.ZodType<T>
referencing the hand-written type. Wire types/schemas remain private to data access;
form and URL pairs stay with their input owner. Co-location does not permit wire contracts
to cross the view-model seam.

## Naming, documentation, and special files

Name implementation files after their primary declaration in kebab-case. For block layouts,
document the exact transformation for block suffixes and factory prefixes and enforce it;
do not guess transformations per file. Name type/schema pairs after the domain concept,
with the configured block suffix where applicable. Suffix schema constants with Schema.

Document every top-level declaration with an immediately preceding TSDoc comment.
Functions require @param for every parameter and @returns, including void returns.
Classes require @param for constructor parameters. Destructured parameters with a named
type annotation are exempt from @param; those with inline type annotations are not.

Where the layout permits barrels, they contain named re-exports only, not implementation
declarations. No wildcard or default exports. Tests/stories use explicitly configured
exclusions. Register generated, configuration, and bootstrap
files separately when their actual contracts require it; do not silently exempt all files
with convenient names or exempt application code housed beside configuration.

## Enforcement contract

File-local AST checks cover primary declaration kinds/counts, single variable bindings,
named exports, filename agreement, TSDoc, and recognized coupled-pair/schema patterns.
Overloads and callable variable initializers need correct classification.

Symbol/reference analysis covers supporting-declaration consumer eligibility, aliases and re-exports,
mutual recursion, external callers of the secondary function, and shared ownership.
Validate layout-specific placement separately. A count-only ESLint rule cannot establish
these relationships.

Test allowed primary-plus-private-type, eligible supporting co-export, coupled pair,
type/schema/check assignments, overload group, and mutual recursion. Reject unrelated
second functions/classes, extra standalone exports, multiple variable bindings, and
supporting exports with ineligible consumers. Include aliases, barrels, and type-only uses
in dependency-aware fixtures. Report unsupported checks as gaps: this document specifies
the policy, it does not install or implement a validator. See [lint.md](lint.md).
