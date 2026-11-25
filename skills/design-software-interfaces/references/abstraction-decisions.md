# Choosing abstractions and orchestration

## Contents

- [Two independent decisions](#two-independent-decisions)
- [Selection procedure](#selection-procedure)
- [When a separate interface is useful](#when-a-separate-interface-is-useful)
- [When an orchestrator is appropriate](#when-an-orchestrator-is-appropriate)
- [Common cases](#common-cases)
- [Explaining the decision](#explaining-the-decision)

Use this as an engineering decision procedure, not a language requirement or
universal standard. Start with real use cases, project conventions, and the cost
of changing an already published contract. Do not rename existing types merely
to fit a pattern.

## Two independent decisions

**Responsibility** identifies the component's work: holding a value, protecting
invariants, calculating a policy, adapting a provider, or coordinating an
application use case.

**Contract and implementation form** determines how clients use the component
and what can vary independently: a function/callable, concrete class, role
expressed as an interface/protocol, abstract family, or module API.

Thus, "interface or orchestrator" combines two different decisions.
`PlaceOrderHandler` can be a concrete orchestrator that depends on
`PaymentGateway` without having its own `PlaceOrderHandlerInterface`. If its
consumers need a separate entry-point contract, the same orchestrator can
implement one. Names such as `Handler`, `Service`, and `Manager` do not establish
design quality.

## Selection procedure

1. **Describe the capability from the client's perspective.** Name its inputs,
   results, effects, and invariants. If consumers need unrelated capabilities,
   investigate separate roles; do not copy the entire class into one interface
   [S27](sources.md#s27).
2. **Choose the simplest sufficient implementation.** A standalone transformation
   often needs only a function. A class can suit values with invariants, cohesive
   behavior, identity, a lifecycle, or persistent dependencies. Closures can also
   retain configuration or state; choose for clarity, not by a blanket rule.
   Concrete types do not need a hierarchy by default [S08](sources.md#s08).
3. **Check the need for an independent boundary.** Name an existing consumer,
   the replaceable detail, and the present benefit of a separate contract. If
   there is no demonstrable benefit, keep a concrete dependency or module API.
   Otherwise, choose an interface, protocol, callable type, or another idiomatic
   mechanism. First check for a suitable standard-library or ecosystem contract.
4. **Check behavioral substitutability.** Compare accepted inputs, effects,
   results, errors, and lifecycle, not just method names. Two classes with
   `send()` do not necessarily implement the same role [S03](sources.md#s03).
5. **Identify who coordinates the use case.** If participants, step order,
   results, and failures must be coordinated, name the owner of that work.
   An orchestrator can be a function or class. Assess its need for a separate
   interface independently; not every handler needs an inbound port.
6. **Evaluate inheritance separately.** Shared code alone does not justify an
   abstract base class. Require a meaningful family, LSP, and agreed shared
   invariants, hooks, and `protected` members. Consider composition first for
   behavior reuse. In C++, a pure abstract class can express an interface;
   do not apply PHP's terminology literally across languages
   [S08](sources.md#s08).

When either a class or a closure retains state, also check its ownership and
lifetime. A private field or captured variable can still hold data from another
request or expose a mutable alias. Use the
[state-leak audit](state-leak-audit.md) for the detailed tracing procedure.

## When a separate interface is useful

Introduce one when at least one applicable reason produces a concrete benefit:

- A consumer depends on a **stable role** while the provider evolves
  independently; express the contract in the consumer's terms instead of
  reproducing an entire SDK.
- Alternative implementations, decorators, plugins, or adapters already need
  a common behavioral promise. Verify their actual substitutability.
- Application policy needs isolation from an external integration or device,
  including tests without a real provider
  [S07](sources.md#s07), [S32](sources.md#s32).
- A meaningful nondeterministic dependency needs controlled substitution:
  time, randomness, networking, or the filesystem. An interface is one option;
  a callable or a value supplied for the current invocation may suffice.
- A published contract, framework, or established architecture already defines
  the interaction protocol the component must support.

**Implementation count is not a threshold.** A single payment-provider adapter
can justify a port today. Several similar local classes may lack a shared
semantic role. "There might be a second implementation someday" is a weak
reason without a consumer, a concrete source of variation, and a current benefit.

An interface costs effort to understand, maintain, document, and evolve with
its implementers. Public methods or a DI container alone do not justify a
separate role. Do not create an interface copy of every pure value object or
simple service just for mocking. Equally, do not reject a useful I/O testing
boundary because production currently has only one implementation.

A mocking tool's difficulty with `final` or private members does not establish
the need for a new public API. First examine the testing approach and existing
boundary. If SDK independence is the goal, wrapping the SDK in an interface
while retaining its types throughout the signatures does not achieve it.

## When an orchestrator is appropriate

Introduce an orchestrator when an application use case needs a clear entry
point and its work coordinates several participants. This can include loading
data, invoking domain operations, passing results, choosing the next step,
owning a local transaction, and handling failures. Service Layer describes
an application boundary and the coordination of operation responses
[S28](sources.md#s28).

Look for these signs of a cohesive orchestrator:

- One understandable goal and result; changes to the order or composition of
  steps belong to this responsibility.
- Dependencies have clear roles and are explicit at construction or invocation.
  Six services can be justified by one use case; do not bundle them in a
  `DependencyBag` merely to shorten the constructor [S13](sources.md#s13).
- In a domain-object model, invariants and reusable domain rules stay with
  their owning objects or policies. The orchestrator decides how the use case
  proceeds without duplicating formulas and state checks throughout handlers.
- In a simple application, a **Transaction Script** may include domain checks
  in the use-case procedure. Do not prohibit all business logic in handlers or
  introduce dozens of domain classes without need [S29](sources.md#s29).
- Success, failure, possible partial effects, retry behavior, and recovery
  responsibilities are defined where relevant.
- Invocation-specific data has a deliberate lifetime. A shared orchestrator
  must not accidentally reuse one caller's state for a later or overlapping caller; a
  session-oriented orchestrator needs explicit lifecycle transitions. Follow
  the [state-leak audit](state-leak-audit.md) when checking this boundary.

A sequence of calls does not by itself require another layer. Extraction can
be justified by a common application boundary for HTTP, CLI, and background
jobs, use-case reuse, transaction ownership, or infrastructure isolation. A
one-line delegate can preserve a useful boundary; length does not prove it
useless. If it separates no responsibility and establishes no contract,
consider a direct call. Do not add `Controller → Manager → Service → Handler`
with equivalent forwarding methods unless each boundary has its own reason.

Split an orchestrator when you find independent use cases or reasons to change,
a reusable cohesive subprocess, or a separate domain policy. Do not create a
class for every step or hide the problem behind generic `execute(type, context)`.
Cross-cutting concerns may fit middleware or decorators when that reduces real
dependencies without hiding semantics.

**Failures and consistency.** A local database transaction does not by itself
undo an external HTTP call or make broker publication atomic. First define
the needed guarantees and behavior when the outcome is uncertain. If a data
commit must be linked to later event delivery, consider a transactional outbox
and its cost; it does not automatically provide exactly-once delivery
[S33](sources.md#s33). Do not impose an outbox, saga, or queue on a simple use
case without the corresponding requirement. Do not promise that compensation
always fully reverses an external effect.

## Common cases

| Situation | Starting choice and checks |
|---|---|
| Local string normalization without identity | A function; add an interface only for a separate substitution role. |
| `Money` or a range with a shared invariant | A concrete value object; the type expresses the data and rule contract. |
| Simple deterministic calculation with persistent configuration | A concrete class or closure; introduce a strategy only for justified variation. |
| One external payment provider; application code needs SDK isolation | A consumer-defined `PaymentGateway` and adapter may already be justified with one production implementation. |
| A use case depends on the current time | Supply invocation time, a callable, or a clock contract according to meaning and lifecycle; do not mock an entire object chain. |
| Placing an order coordinates six participants | A concrete orchestrator with explicit dependencies; assess cohesion and failures, and add its own interface only when consumers need one. |
| Two classes have identical methods but incompatible guarantees | A common interface is premature; clarify the role and semantics or keep separate contracts. |
| A forwarding wrapper preserves a stable boundary between modules | It can be useful; identify the independent change or responsibility it separates. |
| A base class exists only to reuse five lines | Consider a shared helper or component first; inheritance needs its own contract. |

## Explaining the decision

Give a short explanation: **role → consumer → chosen form → isolated
dependency or protected invariant → cost and reconsideration condition**.
For an existing API, include migration consequences. Do not invent future
integrations or requirements to justify an abstraction. If information is
limited, state a provisional choice and the specific fact that could change
it; do not block a useful decision over minor unknowns.
