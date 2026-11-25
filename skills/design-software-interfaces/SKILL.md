---
name: design-software-interfaces
description: >-
  Design, review, and refactor software interfaces: APIs of classes, functions,
  modules, libraries, ports, protocols, and extension points. Use for
  encapsulation, information hiding, state leaks, mutable aliases, ownership,
  shared caches, public/private/protected boundaries, minimal sufficient APIs,
  SOLID, behavioral contracts, method signatures, argument counts, typed
  parameter/options/config objects, builders, dependency injection, and API
  compatibility. Use when choosing a function, concrete class, interface,
  abstract base, or orchestrator, or auditing whether callers can bypass
  invariants or observe state across requests or tenants. Not for visual UI/UX,
  screen layouts, or styling. Apply to relevant API decisions, not every routine
  code edit. Use the language-neutral core with PHP, TypeScript/JavaScript,
  and Kotlin guidance.
---

# Design Software Interfaces

Design a contract clients can understand and use correctly without knowing its
implementation. Reduce unnecessary dependencies and compatibility obligations
while supporting complete real use cases. Do not optimize method or parameter
counts independently of meaning.

Treat an interface as the entire observable contract: operations, types, results,
errors, effects, lifecycle, ownership, and behavioral guarantees. Distinguish
this from a language-level `interface` declaration, which expresses only part
of that contract.

## Workflow

1. **Identify the boundary.** Establish language and minimum version, clients,
   contract owner, application/library status, implementations, and compatibility
   constraints. Read local instructions, APIs, call sites, and tests. Without code,
   state assumptions; do not invent requirements or block useful work on minor gaps.
2. **Write client scenarios.** Show ordinary, boundary/failure, and advanced calls.
   One meaningful call can suffice for a small function. Identify what the client
   must know or coordinate.
3. **Name invariants and changeable decisions.** Identify valid states, related
   values, atomic transitions, state/resource owners, and implementation decisions
   clients should not depend on.
4. **Choose the form and sufficient surface.** Decide the implementation role and
   need for an independent contract separately. Name a client and use case for
   each exposed member. Replace fragile client-side sequences with a cohesive
   operation when correctness requires it.
5. **Trace ownership and authority.** For encapsulation/state audits, follow
   retained inputs, outputs, nested aliases, callbacks, resources, caches, and
   lifecycle boundaries using [the state-leak audit](references/state-leak-audit.md).
   Establish an observable contract violation before labeling exposure a bug.
6. **Review signatures and SOLID.** Assess call-site clarity, types, independent
   concepts, valid combinations, and dependency roles. Use the decision tree in
   [Signatures](references/signatures.md), not a mechanical refactoring threshold.
7. **Specify behavior.** Record relevant preconditions, postconditions, errors,
   effects, ownership, lifecycle, and limits. Do not fill irrelevant template sections.
8. **Check evolution.** Consider callers, external implementers, subclasses,
   serialization, and ABI where applicable; check behavior separately. A new
   default argument is not proof of compatibility.
9. **Verify the decision.** Inspect real calls and meaningful scenarios. For a
   behavior fix, reproduce the observable violation and verify the correction;
   for a design, provide testable acceptance criteria. Distinguish executed checks,
   source-based analysis, and assumptions. Compilation alone does not prove design
   quality. Label pseudocode/incomplete snippets and never claim execution without it.

## Choose a function, class, interface, or orchestrator

Choose responsibility first, then representation and substitution boundaries.
**An orchestrator is an implementation role** that a function or concrete class
can fulfill; a separate interface is optional. A concrete class already has an API.

| Form | Choose when |
|---|---|
| Function / callable | A cohesive operation needs only a call contract. A closure can capture configuration/state; state alone does not require a class. |
| Concrete class | Cohesive behavior, invariants, identity, lifecycle, or persistent dependencies justify an object; an independent substitution boundary is not needed. |
| Interface / protocol / port | A consumer benefits from substitution, independent evolution, integration isolation, plugins, or controlled I/O/time/randomness dependencies. |
| Abstract base class | An intentional inheritance contract and shared family implementation/state justify it. Similar code alone does not. In C++, a pure abstract class can itself represent an interface. |
| Orchestrator / application service | One use case coordinates participants, sequencing, results, and failures. This role combines with the forms above. |

For each new interface, name its consumer, semantic role, and the dependency it
actually separates. One production implementation can justify an interface;
two matching signatures do not establish a common behavioral contract. Do not
create `FooInterface` for every `Foo` merely for DI, SOLID, or mock-tool convenience.
Reuse appropriate existing ecosystem/project contracts.

For an orchestrator, inspect use-case cohesion, domain-rule ownership, explicit
dependencies, and failure state. Dependency count is a signal, not a limit.
Allow a simple transaction script; keep established domain invariants with their
owners. Judge a forwarding layer by its boundary's value, not its line count.

Read [Abstraction decisions](references/abstraction-decisions.md) when selecting
or reconsidering these forms.

## Encapsulation baseline

- Start with the narrowest **sufficient** visibility: private for class details,
  module/package/internal for collaborating implementation, public/export for
  supported contracts. This is design policy, not a claim about language defaults.
  Check compatibility and migration before narrowing an existing public API.
- Hide mutable representation, caches, algorithms, helper steps, and provider
  details when clients need not depend on them. Expose relevant guarantees and effects.
- Protect both incoming and outgoing ownership boundaries. A getter can return
  mutable internals, and a constructor can retain caller-controlled mutable input.
  Follow nested values and capabilities; read-only syntax is not a proof of isolation.
- Select values, snapshots, restricted views, ownership transfer, or documented
  sharing according to the intended contract. Do not prescribe universal deep
  cloning/freezing or silently change identity, liveness, or resource semantics.
- Distinguish domain objects from data carriers. Public typed DTO/record/value
  properties are valid when data is the contract and required invariants remain
  protected. Avoid meaningless getter/setter pairs.
- Treat `protected` as a subclass contract. Do not expose internals for hypothetical
  reuse or tests. Prefer composition unless extension rules justify inheritance;
  account for frameworks/proxies before choosing `final`/`sealed`.
- Keep service dependencies explicit at construction or invocation, according to
  their role and lifetime. Hidden implementation does not justify retrieving
  hidden dependencies from a global container.
- Do not treat private, missing exports, or `@internal` as security isolation.
  Apply appropriate authorization and runtime/process boundaries where required.

## Find state leaks and encapsulation violations

Use [State-leak audit](references/state-leak-audit.md) for the complete procedure.

1. Establish the promised owner, allowed mutation/observation, sharing model, and
   lifetime. If these are unclear, report a contract gap rather than inventing a promise.
2. Trace both directions across the boundary, through the reachable object graph
   and underlying storage/resources: inputs, getters, collections, clones, builders,
   closures, events, iterators, serializers, and injected/shared dependencies.
3. Check time and scope: mutation after validation, partially initialized objects,
   reentrant callbacks, retained work after disposal, and request/tenant state in
   longer-lived services or caches.
4. Demonstrate the path from an allowed external action to a broken invariant,
   isolation rule, snapshot/lifecycle promise, or forbidden capability. Record
   evidence, confidence, and a minimal reproduction; search matches are leads only.
5. Repair at the responsible boundary and verify both the failure case and a
   legitimate-use control. Preserve supported sharing, identity, and ownership.

Do not equate public access, a getter, a shared entity, or a live read-only view
with a bug. Do not equate private fields, `readonly`, `val`, a shallow copy, or a
frozen outer object with deep immutability. Account for the actual language and
whether protection is promised at compile time or runtime.

## SOLID as a dependency review

| Principle | Inspect | Avoid automatic rules |
|---|---|---|
| SRP | Independent reasons for change and owners of requirements. | One method per class. |
| OCP | A demonstrated extension axis and stable substitution point. | Plugins for imagined changes or a ban on modifying code. |
| LSP | Accepted inputs, promised results/effects, invariants, lifecycle, and semantics. | Treating signature compatibility as sufficient. |
| ISP | A cohesive client role without unrelated capabilities. | One interface per method regardless of benefit. |
| DIP | Policies depend on needed contracts; details implement them. | One copied interface per concrete class. |

## Signature review policy

Do not claim a universal optimal parameter count. Use **this skill's heuristic**:
0–2 is often convenient; inspect calls with 3; analyze and justify 4+. These are
review prompts, not quality scores or mandatory limits. Two interchangeable
primitive arguments can be riskier than five clearly named arguments.

1. Group values into a domain type when they share meaning or an invariant:
   `Money`, `DateRange`, `PageRequest`. Do not introduce types just to lower a count.
2. Keep clear independent inputs explicit; use precise types, names, and language
   idioms. Do not move per-call inputs into mutable service fields, globals, or a
   required setter sequence.
3. Consider typed `Options` for operation settings, `Command`/`Request` for one
   action's data, and `Config` for instance configuration. Keep these distinct
   from injected services.
4. When parameters select different operations, consider separate methods, an
   enum/discriminated union, or a strategy. Choose the simplest sufficient form;
   a genuine binary value can appropriately be boolean.
5. For many constructor services, inspect responsibilities and cohesive behavioral
   groups. `DependencyBag`, `AppContext`, and containers do not reduce coupling
   by hiding it. Preserve justified explicit orchestration.
6. Consider builders for genuinely staged construction. Validate the resulting
   object before use, document reuse/ownership, and do not require a builder for
   three straightforward named arguments.

Check required fields, defaults, combinations, validation, mutability, copying,
and compatibility for every parameter object. A bag of twenty unrelated fields
still leaves the client with twenty independent decisions.

## Observable qualities of a good interface

- A real client solves its task without private knowledge or workarounds.
- Names, types, units, and calls express intent.
- Necessary states are representable; invalid states are prevented or rejected
  at defined boundaries. Early input validation does not guarantee workflow atomicity.
- Reads do not unexpectedly change domain state; command effects are explicit.
  A useful result from a mutating command can deliberately depart from strict CQS.
  Separating reads and writes does not require CQRS.
- Errors, absence, empty results, conflicts, and cancellation are distinguishable
  where the client must react differently.
- Aliases, lazy collections, callbacks, resources, and background work preserve
  promised ownership, isolation, and lifecycle.
- Implementations are interchangeable within the contract. Optimizations and
  caches preserve promised semantics; specify freshness, identity, and scope when relevant.
- Each abstraction pays for itself through clarity, a protected invariant, or
  reduced real coupling; state its costs and constraints.

## Deliverable

Scale the answer to the task. For substantial work, provide:

1. Context, clients, and assumptions.
2. Chosen form and justification, including whether a separate interface is needed
   and whether the implementation orchestrates a use case; public API and real calls.
3. Hidden/exposed elements and reasons; alternatives and costs for disputed choices.
4. Invariants, results, errors, ownership, and relevant effects.
5. Signature/parameter-object rationale and concrete SOLID conclusions.
6. Compatibility impact and migration for existing published APIs.
7. Checks performed, limitations, and testable acceptance criteria.

For an audit, tie each finding to code/call sites, the promised contract, an
ownership/authority path where relevant, and an observable consequence. Distinguish
confirmed defects, plausible risks, contract gaps, and optional improvements.
Report negative controls and discard false positives. A long argument list or
getter alone is not a defect.

## References

- [Abstraction decisions](references/abstraction-decisions.md): functions, classes,
  interfaces, bases, orchestrators, and examples.
- [Encapsulation and SOLID](references/encapsulation-solid.md): visibility,
  representation, behavior, inheritance, and useful abstractions.
- [State-leak audit](references/state-leak-audit.md): ownership/authority tracing,
  mutation and lifecycle probes, evidence, repairs, and false-positive controls.
- [Signatures](references/signatures.md): arguments, flags, defaults,
  options/config/request/value objects, DI, and builders.
- [Contracts and compatibility](references/contracts-compatibility.md): errors,
  ownership, async, caches, acceptance scenarios, and evolution.
- [Language examples](references/language-examples.md): PHP, TypeScript/JavaScript,
  and Kotlin. Read the relevant language section.
- [Primary sources](references/sources.md): source status and applicability.
  Recheck official documentation when a decision depends on a version.

Distinguish adopted specification requirements, author/platform guidance, and
this skill's synthesis. Do not label SOLID, parameter thresholds, "everything
private," or mandatory config objects as ISO/PSR/RFC requirements. Analyze HTTP/RPC
protocol contracts separately rather than automatically transferring local API rules.
