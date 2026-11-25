# Audit State Leaks and Encapsulation

## Contents

- [Define the promise](#define-the-promise)
- [Discover and trace paths](#discover-and-trace-paths)
- [Inspect time and scope](#inspect-time-and-scope)
- [Produce a behavioral witness](#produce-a-behavioral-witness)
- [Choose a bounded repair](#choose-a-bounded-repair)
- [Reject false positives](#reject-false-positives)
- [Report findings](#report-findings)

Use this procedure when state, resources, or authority cross a class, module,
callback, request, or tenant boundary. Scale the audit to the actual contract;
do not impose concurrency or distributed-system analysis on a pure local function.
The procedure is this skill's engineering synthesis. Use language documentation
for exact copying, visibility, lifetime, and callback semantics.

## Define the promise

Treat a state leak as a reachable path through which an external participant can
observe, change, retain, or control state beyond the intended contract. State can
also bleed between clients through internal shared storage without a public getter.
Establish the violated promise; the mere existence of an alias is not a defect.
Track information-hiding coupling separately when provider types or representation
details escape without a mutable-state violation.

For each significant boundary value, record the following distinctions:

| Dimension | Questions |
|---|---|
| Owner | Who may change it: caller, component, borrowed owner, or explicit co-owners? Is ownership transferred? |
| Reachability | Which external aliases, nested values, closures, handles, or receivers reach it? |
| Mutability | Can container structure, elements, keys, object internals, or underlying resources change? |
| Observation | Is the result a value snapshot at a defined time, a live view, or deferred work? |
| Identity | Is the same instance promised, only an equivalent value, or neither? |
| Scope | Is state valid for a call, instance, request, user/tenant, or process? |
| Lifetime and authority | Who can close, detach, cancel, emit, revoke, subscribe, or retain it, and until when? |

Read documentation, callers, tests, invariants, and established project behavior.
When ownership is unspecified, identify a contract gap or conditional risk instead
of asserting that every constructor copies or every getter returns a snapshot.
Distinguish compile-time restrictions from runtime promises and actual trust boundaries.

## Discover and trace paths

1. **Inventory entry and exit points.** Begin with public/exported APIs and
   protected extension points; include indirect calls from dependencies, callbacks,
   serializers, and framework hooks. Follow the used paths rather than only the
   class containing the field. Use `rg --files` and focused `rg` searches to find
   relevant definitions and call sites; inspect implementations and consumers.
2. **Trace inbound aliases.** Follow constructor/factory arguments, setter values,
   options/config, builders, injected services, and registry/global values into
   stored fields or captures. Can the caller change a value after validation or
   after the component has promised to retain a stable configuration?
3. **Trace outbound aliases and capabilities.** Follow properties/getters, returned
   arrays/collections, iterator yields, event payloads, callback arguments and
   receivers, closures, error objects, and resource handles. Follow delegated return
   values too, including the result of an unsubscribe/disposal closure: a chainable
   helper can return the internal object even when the outer API looks narrow.
   A narrower declared type or wrapper may still pass on the original object or
   excess control. For serialized/debug/error output, also check whether exposed
   representation or data violates an actual observation boundary; a deliberate
   public DTO is not a leak merely because it exposes values.
4. **Follow the relevant graph.** For each assignment, view, spread, clone, copy,
   conversion, or factory, trace both object identity and underlying storage or
   resources. Distinct wrappers, including distinct buffer objects, can still
   share mutable backing storage. Inspect accessible ranges and escape hatches:
   a view's `.buffer` can expose storage outside the view's bounds. A new outer
   object can share mutable children, keys, references, streams, or captured services
   [S41](sources.md#s41), [S42](sources.md#s42).
   Stop where immutable values or an explicit sharing contract resolve the concern.
5. **Inspect the actual retained representation.** When defensive copying is
   appropriate, normalize/copy what is needed and validate that representation
   before publication; validating a mutable original does not validate a later copy
   [S37](sources.md#s37). Inspect copy hooks, getters, and lazy conversions that can
   run foreign code. Copying alone does not create a coherent concurrent snapshot.

For a candidate, write the concrete path:

**Origin → retained/captured object or storage → boundary operation → reachable state or
capability → external action → observable consequence.**

Use searches such as `return`, `yield`, `emit`, `subscribe`, `getBody`, `toArray`,
`toList`, `snapshot`, `readonly`, `clone`, `copy`, or cache writes as leads, tailored
to the language. A keyword match, access modifier, or accessor name proves nothing
by itself; dynamic dispatch and indirect aliasing can also evade text searches.

### Language checks

- **PHP:** Distinguish shared object identity from explicit `&` variable aliases.
  Ordinary array value copies do not recursively clone their objects; explicit
  references inside arrays require separate inspection. `clone` starts as a
  shallow property copy and can run `__clone`. `readonly` does not make a stored
  mutable object or resource immutable [S17](sources.md#s17), [S34](sources.md#s34).
- **TypeScript/JavaScript:** Follow nested aliases through `Readonly<T>`, readonly
  arrays, spreads, and slices. Check actual runtime objects where runtime protection
  is promised. `Object.freeze` restricts the target's property surface rather than
  recursively freezing every reachable object or all built-in internal state
  [S19](sources.md#s19), [S35](sources.md#s35).
- **Kotlin:** Separate a read-only collection interface from an immutable backing
  collection; inspect other mutable aliases and element objects. Collection copies
  and generated data-class `copy()` are shallow. `val` alone also does not establish
  deep immutability or getter-result identity [S23](sources.md#s23), [S36](sources.md#s36).

Use [Language examples](language-examples.md) for compact probes and controls.

## Inspect time and scope

Trace construction, publication, validation, callbacks, suspension/yield points,
completion, disposal, and reuse. Check whether foreign code can observe an
intermediate state, reenter an unfinished transition, or use retained authority
after the promised lifetime. Reentrancy can occur in a single thread.

Inspect `this`/self escape through constructor registration, callbacks, overridable
calls, static storage, and factories. Account for subclass initialization. CERT's
constructor-publication guidance supplies examples; its Java memory-model rules
must not be imposed on other languages [S38](sources.md#s38).

For callback boundaries, inspect **payload, receiver, timing, exception, and
retention**, not only function types. For example, Node.js `EventEmitter` invokes
listeners synchronously and gives ordinary function listeners the emitter as
`this`. A subscription-only wrapper forwarding the listener unchanged can therefore
leak emitter authority or enable reentry [S39](sources.md#s39). Establish whether
the wrapper actually promises narrower authority before reporting a defect.
Check returned cleanup functions as well: a direct return of `emitter.off(...)`
returns the emitter. Track each registration's authority and allowed cancellation
repeatability, especially when the same callback is registered more than once;
one handle must not cancel another registration contrary to its contract.

Follow data into singletons, caches, pools, subscriptions, and background tasks.
Check captures of request-scoped or tenant-specific collaborators by longer-lived
objects; .NET's captive-dependency guidance illustrates this lifetime mismatch
without prescribing another container's rules [S40](sources.md#s40).

For caches, inspect all inputs on which promised results depend: arguments,
tenant/user and authorization context when relevant, configuration, dependency
identity, freshness, and invalidation. Check cached mutable results as well as keys:
one client can contaminate another through the same returned object. Do not add
every context field to every key blindly; derive partitioning from the contract.

Check cleanup after success, failure, cancellation, and reuse. Inspect lazy results
that outlive closed resources and callbacks that run after disposal. For streams,
include cursor movement, close, and detach authority, not only content mutation;
PSR-7 explicitly permits mutable streams inside immutable-style messages
[S15](sources.md#s15).

## Produce a behavioral witness

Prefer the smallest supported client interaction that demonstrates the broken
promise. Observe public results, subsequent operations, controlled collaborators,
or lifecycle signals. Do not assert private field names or incidental call order.
Static reasoning can prove a violation; label it separately from an executed test.

| Probe | Observe | Legitimate-use control |
|---|---|---|
| Change an input or nested child after construction | Does later behavior violate retained-value semantics? | A deliberately live configuration should respond according to its contract. |
| Mutate a returned value, nested element, or exposed handle | Does the owner change without a supported transition? | A detached result may change independently; intentional shared entities may accept valid mutations. |
| Obtain a claimed snapshot, then mutate the owner through its API | Does the old result change against its snapshot promise? | A documented live view is expected to reflect owner changes. |
| Change an event payload or use its callback receiver | Can a listener corrupt later behavior or gain forbidden authority? | Deliberately shared payload/control contracts remain valid. |
| Reenter through a synchronous callback mid-transition | Is an invalid intermediate state externally observable? | A guarded call or explicit intermediate-state protocol may be correct. |
| Run client A, client B, then A again in one process | Do request/tenant state or cached results cross the wrong scope? | Shared immutable configuration and correctly partitioned results are valid. |
| Control two overlapping operations with barriers/deferred collaborators | Can one operation overwrite context retained by the other? | Explicit synchronization, ownership transfer, or independent state may satisfy the contract. |
| Consume deferred output or invoke retained callbacks around disposal | Is promised resource/callback lifetime violated? | Respect documented borrowing scopes and permitted in-flight completion. |

Use deterministic interleavings, fake clocks, or explicit completion barriers when
needed; avoid timing sleeps and real external side effects. Test contract-relevant
error paths, not an exhaustive interleaving matrix without a concrete risk.

For a fix, retain a regression that fails under the original behavior and passes
afterward, plus a relevant control demonstrating that valid sharing, identity,
liveness, or lifecycle behavior remains intact. Run applicable existing tests;
do not claim full absence of state leaks from a few probes.

## Choose a bounded repair

Repair the boundary responsible for the invariant and state the tradeoff:

- Normalize necessary input into owned primitive/immutable values; validate the
  representation actually retained. Copy only the depth needed by the guarantee.
- Return a detached value/snapshot or a suitable restricted view. Preserve a live
  view if liveness is part of the API; changing it to a snapshot may break clients.
- Replace excess authority with a narrow operation/capability. Inspect callback
  receivers and escape hatches as well as nominal method lists.
- Restore invariants before calling foreign code, defer publication until ready,
  or enforce a documented reentrancy protocol.
- Keep request data local, partition state correctly, bound borrowed work to its
  scope, and arrange ownership-aware cleanup. Do not replace an explicit lifecycle
  with hidden globals or blanket locking.
- Document sharing or transfer only when it matches intended requirements and
  actual caller obligations. Do not repair a violated existing promise merely by
  weakening its documentation; assess compatibility and migration.

Do not prescribe arbitrary deep cloning, JSON round trips, recursive freezing, or
serialization as universal remedies. They can change identity, type behavior,
resource ownership, callbacks, cycles, or costs; custom hooks can execute during
conversion. A coherent snapshot of a concurrently changing graph needs an appropriate
consistency mechanism such as synchronization, immutable versions, or ownership
transfer. Select it for the actual execution model and contract.

## Reject false positives

Before classifying a violation, check these controls:

- Public DTO/value properties intentionally expose data while preserving required
  invariants. A getter can be legitimate and need not return a copy.
- A read-only live view may reflect owner mutations; read-only does not promise a
  time-stable snapshot. A structural snapshot may share independently immutable items.
- Shared entities can legitimately expose operations that preserve the required
  invariants. Also check any enclosing aggregate invariant before accepting that sharing.
- Caller mutation after a valid ownership-transfer agreement can violate the
  caller's obligations rather than the component's promise.
- Compile-time-only restrictions do not promise resistance to casts/reflection.
  Such probes matter when runtime isolation is promised or they model supported
  untyped/foreign-language consumers. Analyze actual security boundaries separately.
- A retained reference is not automatically an unwanted memory retention problem,
  nor is every state exposure a confidentiality vulnerability. State the demonstrated
  impact and intended observers instead of inflating severity.

## Report findings

For each significant finding, provide:

1. **Contract and evidence:** invariant/ownership/lifetime promise and its source.
2. **Path:** relevant code locations and reference/capability propagation.
3. **Trigger and consequence:** supported action, timing/scope, and observable break.
4. **Classification:** confirmed violation, plausible risk, contract gap, intentional
   behavior, or optional improvement; explain material uncertainty.
5. **Repair and compatibility:** responsible boundary, smallest adequate change,
   costs, and any effect on identity, sharing, liveness, or public API.
6. **Verification:** executed reproduction/regression, static proof, source check,
   and relevant negative controls; explicitly identify unexecuted checks.

Combine findings that share one cause and repair. Do not count each getter or alias
as a separate defect when it is merely another path to the same violated boundary.
