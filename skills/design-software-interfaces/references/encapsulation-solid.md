# Encapsulation, API Sufficiency, and SOLID

## Contents

- [What to hide](#what-to-hide)
- [Choosing visibility](#choosing-visibility)
- [Data and invariants](#data-and-invariants)
- [Ownership and state leaks](#ownership-and-state-leaks)
- [Sufficiency and clarity](#sufficiency-and-clarity)
- [Applying SOLID](#applying-solid)
- [Inheritance and extension](#inheritance-and-extension)
- [Validating the decision](#validating-the-decision)

## What to hide

Distinguish keeping state and operations within a controlled boundary from hiding
design decisions from other modules. A private field can still be changed through
an object returned to a caller. A public method can force callers to understand
internal machinery. Parnas provides the foundation for information hiding
[S01](sources.md#s01).

Ask: **What would a correct caller have to change if we replaced the implementation?**

| Usually hide | Expose or document when callers need it |
|---|---|
| Storage structures, internal indexes, cache layout | Result meaning, permitted staleness, promised ordering |
| SQL, provider SDKs, adapter-specific DTOs | Domain inputs and results, expected failure categories |
| Algorithm steps and private helpers | A complete operation, its conditions, and its effects |
| Locking mechanisms and internal connection pools | Concurrency guarantees and resource ownership |
| Internal buffer and cache limits | Limits callers must respect to use the API correctly |
| Retry machinery and technical logging | Possible repeated effects, cancellation, and uncertain outcomes |

Do not promise that an implementation is completely unobservable: execution time,
memory use, and I/O can be visible. Remove the **need to depend** on details and
commit only to the necessary semantic and resource guarantees. If an API
intentionally exposes a low-level resource, describe that boundary accurately.

Do not require callers to sequence `loadState()`, `checkStock()`, `decrement()`,
and `persist()` correctly to perform one `reserve()`. Determine whether the
complete operation must preserve an invariant atomically. Independent low-level
functions can still be appropriate, for example in an algorithms library.

## Choosing visibility

Choose visibility by purpose, not by a mechanical rule:

| Level | When it is sufficient | What to check |
|---|---|---|
| `private` / local function | A detail of one implementation | Can a reference to internal state bypass the boundary? |
| module/package/internal | Collaboration within one component | Does an internal type escape through a public signature? |
| `protected` | An intentional extension point for subclasses | What are the call order, invariants, and permitted overrides? |
| public/export | A supported capability for a real caller | Is the use case identified and the contract defined? |

Bloch's recommendation to minimize accessibility concerns API commitments and
conceptual complexity [S02](sources.md#s02). It does not prohibit public data or
describe every language's defaults. An `@internal` annotation can inform tools
and users without preventing imports at runtime.

Constructor and factory visibility are also part of the API. A private
constructor is useful when a factory protects a specific invariant or creation
policy. A factory that merely repeats the constructor needs a reason to exist.
Keep unstable infrastructure dependencies out of a domain value's interface.

Access modifiers organize software boundaries; they are not a substitute for
authorization, secret management, or process isolation. Check the language's
actual enforcement before claiming a runtime access guarantee.

## Data and invariants

First establish the type's purpose:

- For a **stateful domain object**, expose actions that preserve its rules:
  `changeQuantity()`, `cancel()`, or `reserve()`. An unrestricted `setStatus()` is
  dangerous when it bypasses transition rules.
- For a **value object**, allow callers to read its meaningful components while
  preserving their joint validity and value semantics. An immutable scalar does
  not automatically need a getter; inspect every nested value.
- For a **DTO or record**, the data itself can be the contract. Transport
  validation and domain validation may happen at different boundaries; name them.
- For a **collection, snapshot, or view**, specify whether membership and element
  values are captured at acquisition, what can change, and who observes changes.

Do not infer deep immutability from `readonly`, `val`, `const`, or `Readonly<T>`.
Inspect reachable objects, other references to them, callbacks, and resources.
Copying a container need not copy its elements. A read-only view can remain live;
a structural snapshot can still share mutable elements. See the
[language examples](language-examples.md) for concrete distinctions.

Preserve useful query and getter operations. Tell Don't Ask does not imply
removing every query; Fowler explicitly discusses this limit
[S12](sources.md#s12). The problem is making callers maintain an object's
invariant, not naming a method `getSomething()`.

## Ownership and state leaks

Review both directions of a boundary. A constructor or setter can retain an
object that the caller still controls; a getter, iterator, event, callback, or
cached result can expose state that the component still owns. A private field
or a read-only outer type alone does not settle either case.

For each suspected leak, trace the value from its creation through storage and
every exposed reference. Identify the promised invariant or isolation guarantee,
then show a permitted caller action that can break it. Distinguish an accidental
leak from a documented live view, shared service, mutable DTO, or transfer of
ownership. Explicit sharing still needs a coherent contract for mutation and
lifetime.

Use the [state leak and encapsulation audit](state-leak-audit.md) for the full
procedure, evidence requirements, false-positive controls, and repair choices.
For mutable values, copying only at return cannot repair an incoming alias;
copying only at intake cannot repair an outgoing alias. Choose the least costly
boundary that protects the actual promise, and check nested state before declaring
it isolated.

## Sufficiency and clarity

Compare the cost of using the API with the cost of supporting it. Removing a
method helps only if the remaining API still expresses the task clearly and
correctly. `execute(action, payload)` may be one method with dozens of implicit
contracts and invalid combinations.

Assess:

1. Whether the ordinary call is clear and avoids unnecessary mandatory setup.
2. Consistency of verbs, argument roles, results, and units.
3. Whether callers can understand the contract without reading the implementation.
4. Protection against likely mistakes and explicit handling of ambiguous cases.
5. Freedom to evolve the implementation without changing correct callers.
6. The cost of abstraction: new concepts, allocation, indirection, serialization,
   and debugging, where these costs materially affect the system.

Do not replace these checks with an invented score or a method count. Prototype
call sites and judge clarity in context [S09](sources.md#s09).

## Applying SOLID

### SRP

Identify the source of each requirement change. Pricing, accounting report
formatting, and persistence may change independently. One application use case
can legitimately coordinate several dependencies; their count alone does not
prove an SRP violation. Reason about causes of change [S04](sources.md#s04).

### OCP

Choose an established axis of variation, such as a storage provider or shipping
calculation. Define a stable contract for that role. Do not try to accommodate
every imaginable change in advance: Martin presents this as a strategic choice
[S05](sources.md#s05). A new `switch` does not always require a strategy hierarchy.

### LSP

Check that each implementation accepts the promised valid inputs and preserves
result guarantees, invariants, and permitted state histories. Substitution must
not strengthen demands on callers or weaken promises to them. This is a
behavioral relationship [S03](sources.md#s03), not just type compatibility. If
the base contract permits several result orders, an implementation can choose
any of them; bit-for-bit equality is unnecessary.

Do not label every `UnsupportedOperation` a violation without reading the
contract: rejection may be allowed. A separate capability is often clearer when
most of an interface is permanently unavailable to a particular caller.

### ISP

Map each caller role to the operations it uses. Separate unrelated groups when
this reduces dependencies, for example `CatalogReader` and `CatalogWriter`.
One object can implement both roles [S06](sources.md#s06). Creating an interface
for every method does not establish useful segregation.

### DIP

Define the contract around policy or caller needs, and make adapters conform to
it [S07](sources.md#s07). Do not leak a specific SDK's types into an application
contract when isolating that SDK is the boundary's purpose. Reuse a stable
standard type when a custom wrapper adds no value.

Keep constructor dependencies explicit. Introduce an interface for a useful role
or boundary, not to pair every `Foo` with a `FooInterface`. Distinguish DI from
DIP: passing an object from outside is an assembly mechanism; dependency
direction is a design property. A composition root can know concrete
implementations.

Use the [abstraction decision guide](abstraction-decisions.md) when choosing a
concrete dependency, a separate interface, or an orchestration role. Neither the
number of implementations nor the number of dependencies settles the decision.

## Inheritance and extension

Before exposing a hook, specify when it runs, whether it can mutate state,
whether it must call the parent implementation, which failures are allowed,
and whether it can run repeatedly. Prefer an explicitly supplied strategy or
callback when that is sufficient. Callbacks also need ownership and reentrancy
rules if they can observe or change state during a transition.

Do not make an existing public class `final` or narrow a published method's
visibility without checking subclasses. Preventing extension can be reasonable
for a new class without an inheritance contract; account for ORM frameworks, DI
proxies, and platform constraints. `final` alone does not improve an architecture
or protect a mutable object graph.

## Validating the decision

For a disputed public element, identify its users, the promise it creates, what
would break if it were hidden, and whether callers can accomplish their task
without understanding internals. For a disputed abstraction, name the dependency
it removes. If there is no useful answer, keep the decision local or propose a
reasoned alternative instead of expanding the API speculatively.

Tie an encapsulation finding to a concrete route, a caller action, and an
observable consequence. A missing modifier, getter, shared reference, or
implementation type is evidence to investigate, not a defect by itself. Separate
confirmed contract violations, plausible risks, and optional design improvements.
