# Signatures and parameter objects

## Contents

- [Assessing argument count](#assessing-argument-count)
- [Decision tree](#decision-tree)
- [Distinguishing object roles](#distinguishing-object-roles)
- [Parameter object requirements](#parameter-object-requirements)
- [Types, names, and flags](#types-names-and-flags)
- [Constructors and dependencies](#constructors-and-dependencies)
- [Builders and design cost](#builders-and-design-cost)

## Assessing argument count

Do not claim a proven universal ideal. Use this skill's 0–2 / 3 / 4+ bands to
allocate review attention, not to trigger automatic refactoring. Zero arguments
can hide global inputs, one `Context` can hide a dozen dependencies, and two
strings can allow source and destination to be swapped without a type error.

C++ Core Guidelines I.23 suggests aiming for fewer than four parameters,
allows exceptions, and labels its design rule as not directly enforceable.
This is advice from a particular guide, not an ISO requirement or a universal
quality threshold [S08](sources.md#s08). Do not rewrite a stable API to meet it.

For each signature, inspect:

- The number of independent decisions the caller must make.
- Relationships between values and shared invariants.
- The chance of swapping values without a type error.
- Optional values and valid combinations.
- Hidden inputs, data lifetimes, and effects.
- The knowledge needed to understand an ordinary call.

A symmetric operation on plain values may tolerate swapped arguments;
`copy(from, to)` generally does not. Do not assume symmetry from a function name
when identity or edge-case behavior matters. Five named arguments for a cohesive
operation may be clearer than one opaque array. Naming reduces ambiguity but
does not fix mixed responsibilities.

## Decision tree

Check these questions in order; do not introduce several patterns when one is
sufficient.

| Observation | Candidate decision | Check for unnecessary complexity |
|---|---|---|
| One parameter is fully derivable from another | Remove duplication or compute it internally | Would this add I/O, coupling, or variability in results? |
| Values form one concept or invariant | A domain value object | Does it have a meaningful name and rules of its own? |
| Data describes one application action | A command, request, or parameter object | Are services mixed with data; are all required inputs visible? |
| One operation has many optional settings | A focused options object | Are defaults and valid combinations defined? |
| Settings remain stable for an instance | Config supplied at creation | Are these actually inputs for a single request? |
| Arguments describe different tasks | Separate operations or responsibilities | Would this destroy required atomicity? |
| A mode selects among finite alternatives | An enum, union, or separate methods | Would an enum add an unnecessary concept for a simple boolean? |
| An alternative has its own algorithm | A strategy or function | Is this variability actually needed now? |
| There are many service dependencies | Review responsibility and composition | Would a wrapper merely conceal the same coupling? |
| Construction is complex and staged | A builder or factory | Would named arguments be equally clear? |

When grouping a repeated `from`/`to` pair into `DateRange`, look for a shared
invariant and domain vocabulary. That is more useful than moving arbitrary
fields into `Params`; Introduce Parameter Object illustrates a related group
[S10](sources.md#s10). Do not pass an entire large `User` instead of `UserId`
when an operation only needs the identifier and should not know other properties.

## Distinguishing object roles

Use these as working definitions; there is no universal naming standard.

| Form | What it expresses | Example |
|---|---|---|
| Value object | A domain value and its validity | `Money`, `DateRange`, `Coordinates` |
| Parameter object | Inputs to one operation or a stable group of inputs | `ExportParameters` |
| Command/request | An intent or request data at an application boundary | `PlaceOrder`, `FindProducts` |
| Options | Execution choices for one operation, usually optional | `SearchOptions` |
| Config | Component settings for a defined lifetime | `HttpClientConfig` |
| Builder | The process of constructing a result | `QueryBuilder` |

DTO describes data transfer and may overlap these roles. The name `Config`
does not make a type well designed. Establish its purpose and lifecycle;
do not force renaming when existing terminology is consistent.

Keep required domain inputs visible. For example, `search(query, options)` is
more useful than `search(options)` with an optional `query` when searching
without a query makes no sense. Conversely, a complete `ProductQuery` can
correctly hold every required input if its type actually requires and validates
them. Not every method needs a separate config type.

## Parameter object requirements

Before accepting a design, establish:

1. Which fields are required, have defaults, or may be absent.
2. How absence, `null`, `false`, `0`, an empty string, and an empty collection differ.
3. Which combinations are valid and whether types or variants can exclude others.
4. Where formats, local invariants, and external business conditions are validated.
5. Whether settings are a snapshot or a live reference, and whether nested values
   are mutable. If inputs are retained, who may change them after validation?
6. How unknown fields are handled at a dynamic boundary.
7. The consequences of adding a required field, renaming one, or changing a default.

Use the [state-leak audit](state-leak-audit.md) to check retained input aliases,
nested mutable values, and reuse of parameter objects. A shorter signature does
not establish ownership or preserve invariants after construction.

Do not turn known structured data into `array<string, mixed>` or
`Record<string, any>` just to shorten a signature. An open dictionary is valid
for actual dictionaries, metadata, or extensible schemas; define its keys,
value types, and extension policy. For external JSON, a static type declaration
is insufficient; validate the data at runtime.

For a closed configuration schema, rejecting unknown fields often catches
typos. Extensible wire formats may have different requirements; do not impose
a universal reject/ignore rule without checking compatibility and the protocol.

Do not make every field nullable merely to simplify construction. If the valid
alternatives are "existing object ID" or "new object data," express them with
separate types, a discriminator, or factories; reject conflicting combinations.

## Types, names, and flags

Name roles and units: `timeoutMs`, `from`, `to`, `pageSize`, `currency`.
Use stronger types when they protect a meaningful invariant or prevent actual
confusion. Do not wrap every primitive automatically.

Do not assume a `UserId` alias guarantees distinctness in a language with
structural compatibility. When distinctness matters, use a value object with a
distinguishing discriminant/brand, a branded primitive with a validating factory,
or a suitable nominal typing mechanism. Separate class names alone do not make
otherwise matching TypeScript instance types distinct; private/protected members
from different class declarations can distinguish them [S21](sources.md#s21).
Type distinctness does not replace validation of values at their creation boundary.

Distinguish a boolean value from a flag that selects a different task:

- `setEnabled(enabled)` can directly represent a binary state.
- `export(data, true, false, true)` requires checking what its combinations mean.
- `preview(command)` and `execute(command)` are often clearer than a switch
  controlling side effects; do not add separate methods if they merely force
  more client branching without improving the contract.

Fowler discusses useful boolean exceptions [S11](sources.md#s11). Named arguments
are particularly helpful for repeated primitive types [S24](sources.md#s24).
A default should match ordinary safe behavior and form a stable promise.
Do not disguise undocumented business policy as a convenient default.

Usually return a result instead of using output reference parameters. For
several related results, consider a named result type. Preserve platform idioms,
FFI or streaming APIs, and low-level exceptions justified by measurements.

## Constructors and dependencies

Separate required services, instance settings, and invocation-specific data.
Passing the same dependencies through a `DependencyCatalog` does not resolve
excessive dependencies. Seemann treats a long constructor as a reason to
investigate, not a prohibition [S13](sources.md#s13). Look for natural behavior
groups, use-case coordination, and cross-cutting concerns. A facade should
perform cohesive work instead of merely handing out services. Use the
[abstraction decision procedure](abstraction-decisions.md) to assess an
orchestrator and its need for a separate interface independently.

PSR-11 uses **SHOULD NOT** for passing a container to an object so that it can
retrieve its own dependencies. This is not an absolute ban on container use
in factories or the composition root. Its meta document describes a router
case and a factory exception with a replaceable factory interface
[S14](sources.md#s14). Assess a proposed dynamic registry by its actual role;
do not treat that as a blanket exception or introduce `AppContext` to evade
the recommendation.

Do not move invocation arguments into mutable fields of a shared service to
obtain a zero-argument method. This introduces initialization order, interferes
with repeated or concurrent use, and can mix data from different invocations.
State is legitimate when an instance actually models a session or process with
that lifetime and explicit transitions. Trace callbacks, closures, and cached
objects as well as fields using the [state-leak audit](state-leak-audit.md).

## Builders and design cost

A builder can suit staged construction, independent optional parts, or a need
to guide valid ordering. Define reuse, value copying, and when invariants are
checked. Do not expose a partially valid working object before construction
succeeds. If a builder is reusable, establish whether later builder changes
can affect earlier results; follow the [state-leak audit](state-leak-audit.md)
for ownership checks.

Do not promise speed gains from fewer parameters: a wrapper can add allocation,
copying, and indirection. Measure only a consequential hot path. For a public
library, also check compatibility of the parameter object's constructor or
factory; the object itself becomes part of the API.
