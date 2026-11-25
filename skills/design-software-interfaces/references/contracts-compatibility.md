# Behavioral contracts, verification, and compatibility

## Contents

- [Describing a contract](#describing-a-contract)
- [Errors and effects](#errors-and-effects)
- [Ownership and lifecycle](#ownership-and-lifecycle)
- [Substitutability and optimization](#substitutability-and-optimization)
- [API evolution](#api-evolution)
- [Verification and acceptance](#verification-and-acceptance)

Use this as an engineering template. It combines practical consequences of
contract design; the checklist is not a single standard. For a small pure
transformation, do not produce a report on async work, transactions, or
distributed idempotency. Cover only what matters to its clients.

## Describing a contract

| Aspect | Question the contract should answer |
|---|---|
| Purpose | What one understandable capability does the client receive? |
| Input | Which values are valid, in which units, and within which bounds? |
| Preconditions | In what state or context is the call allowed? |
| Result | What counts as success; what are the result's shape, order, completeness, and repeatability? |
| Postconditions | Which properties are guaranteed after success? |
| Failure | Which failure categories can clients distinguish, and what remains changed? |
| Effects | What changes beyond the returned value; is there I/O, event emission, or background work? |
| Ownership | Who owns supplied or returned data and resources, and who may mutate them? |
| Timing | When does work happen; are cancellation, deadlines, or retries needed? |
| Concurrency | Are concurrent or repeated calls and callbacks supported? |
| Extension | Which variants and extension points are supported? |
| Limits | Which resource or timing properties matter, and what evidence supports them? |

Do not write comments that merely repeat types. Document what the signature
cannot show: a `[from, to)` interval, object identity, allowed stale results,
atomicity, callback order, or errors deferred until iteration. Do not promise
precise performance without measurements; state any required complexity
guarantee separately.

## Errors and effects

Choose an error mechanism consistent with the project: typed exceptions,
a result with success/failure variants, or another established convention.
Do not impose `Result` on every language or use a boolean when callers must
distinguish several outcomes.

Distinguish these where clients need different responses:

- A missing record and unavailable storage.
- An empty list and an inability to execute the query.
- Invalid input and a conflict with current state.
- Cancellation or timeout and confirmed failure before a side effect.
- Full completion, partial results, and an unknown outcome.

For a modifying operation, describe the state after failure: unchanged,
partially completed, rolled back, or requiring later reconciliation. Do not
hide an infrastructure failure behind an empty successful value. Do not create
a public exception type for every internal step when clients respond to them
identically; preserve the technical cause for diagnosis.

Separate reads and changes by meaning. A lookup may use a cache or perform
I/O, but must not unexpectedly change domain state. Under strict CQS, queries
return a value without changing observable state, and commands change state
without returning a value [S30](sources.md#s30). A name such as `get` or `find`
does not establish either guarantee. An ID, receipt, removed element, or updated
state can be a useful result from a modifying operation, including an atomic
operation; describe this as a deliberate exception to strict CQS. Separating
reads and changes does not require CQRS, a message bus, or separate stores.
CQRS separates read and write models and does not require separate databases
in every case [S30](sources.md#s30).

For retries of a significant command, establish the identity of the intent,
the key's scope and lifetime, behavior for the same key with a different
payload, which effects are protected, and the result returned on repetition.
An `idempotencyKey` field alone does not establish exactly-once processing or
retry safety.

## Ownership and lifecycle

Name the result model: an independent value, snapshot, read-only view, shared
mutable object, stream, or iterator. A read-only view may reflect owner changes
and is not necessarily a snapshot. An immutable wrapper may contain a mutable
resource: PSR-7 explicitly distinguishes message and stream behavior
[S15](sources.md#s15). Specify which nested values a snapshot isolates;
copying only the outer collection does not establish that guarantee.

Check both sides of the boundary: retained caller inputs can change after
validation, and returned values or callbacks can expose aliases to owned
state. Use the [state-leak audit](state-leak-audit.md) to trace ownership,
mutation paths, and lifetimes; visibility modifiers alone are not evidence
that state remains under the owner's control. Intentionally shared mutable
state requires an explicit contract, not an automatic defect finding.

For resources, specify who closes them, whether they remain usable after
transfer, and whether repeated `close()`/`dispose()` or iteration is supported.
For lazy results, state when I/O and failures occur. Do not make expensive
loading an unexpected side effect of an ordinary getter without an established,
documented convention.

For async or callback APIs, define completion, whether cancellation stops the
work or only the wait, callback timing, reentrancy, and subscription cleanup.
`Promise`, `Future`, and `suspend` do not answer those questions by themselves.
Include work and retained references that outlive the call when checking
the [state-leak audit](state-leak-audit.md).

Do not expose a partially initialized working object. If `unstarted` is a
valid state, make it an intentional part of the lifecycle rather than a hidden
requirement to remember three setter calls first.

## Substitutability and optimization

Apply LSP to the contract's promises [S03](sources.md#s03). A replacement must
not reject valid inputs, omit promised effects, weaken result guarantees, or
violate promised invariants and state transitions. It need not preserve
unspecified implementation details.

For a cache, compare cached and uncached paths on relevant observable
properties: arguments, dependencies, errors, results, scope, mutation, user or
tenant, configuration, and invalidation. Establish whether callers need the
same object or an equivalent value. Do not substitute another caller's cached
dependency instance when that changes the contract. If stale responses are
allowed, define the policy; do not demand identical freshness contrary to the
cache's purpose. Use the [state-leak audit](state-leak-audit.md) to check key
scope, shared result aliases, and state retained between callers.

Do not infer singleton semantics from the name `ContainerInterface`: PSR-11
allows successive `get()` results to differ depending on implementation or
configuration [S14](sources.md#s14).

## API evolution

Check compatibility along distinct dimensions:

| Dimension | Who or what can break |
|---|---|
| Caller source code | Positional or named calls, types, imports, and overload resolution |
| Contract implementers | Adapters, third-party plugins, test doubles, and classes using `implements` |
| Inheritance | Visibility, overrides, `protected` members, `final`, and constructors |
| Behavior | Defaults, errors, ordering, units, validation, effects, and freshness |
| Data or protocol | Serialization, schemas, fields, enum values, cursors, and ID formats |
| ABI, where relevant | Already compiled consumers and generated methods |

Do not assume an additive change is compatible. A new required interface method
affects implementers; a new closed enum or union variant may break exhaustive
handling; an options field affects construction and serialization. In PHP,
check parameter names and existing implementer signatures even when a new
interface parameter has a default [S16](sources.md#s16). In Kotlin/JVM, even
a defaulted parameter can change the binary signature [S22](sources.md#s22).

For a published boundary:

1. Find the current contract, stability promises, callers, and implementations.
2. Distinguish compatible fixes, extensions, and breaking changes.
3. Consider retaining the old entry point with delegation, an adapter, a
   separate capability interface, or a separate version according to the change.
4. Preserve existing call semantics; document deprecation and migration when needed.
5. Run meaningful existing scenarios and checks of the new guarantees.

If the project follows SemVer, choose the version from changes to its declared
public API. For stable releases with a major version above zero, an incompatible
public API change requires a major increment [S25](sources.md#s25). Do not evade
that requirement by relabeling an existing public capability as internal.
If the project follows another policy, apply it first and explain the consequences.

## Verification and acceptance

Choose checks by risk rather than by a target test count:

- For a new design, write realistic calls and negative examples with explicit
  errors or invalid types.
- For a fix, reproduce the observable contract violation and verify that the
  change removes it without altering adjacent promises.
- For multiple implementations, apply a shared set of contract scenarios.
- For a data boundary, check significant invalid values and combinations.
- For resources and side effects, check relevant failure and completion paths.
- For ownership and lifecycle, exercise the applicable mutation, retention,
  and caller-isolation scenarios in the [state-leak audit](state-leak-audit.md).
- For compatibility, check callers and implementers separately; check ABI only
  where it matters.

Do not replace outcome checks with tests of parameter counts, private names,
or exact internal call sequences. Do not expand the public API solely for
testing. Do not perform external modifying operations merely to verify an
example. Report tool-verified results, conclusions from analysis, and unknowns
separately.
