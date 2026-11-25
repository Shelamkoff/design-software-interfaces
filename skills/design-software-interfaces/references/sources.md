# Primary Sources and Applicability

Relevant source material checked: **2026-11-22**.

**Status of rules.** Apply PSR and SemVer requirements within a project's declared
adoption of the relevant specification; distinguish `MUST`, `SHOULD`, and `MAY`.
Language documentation describes version-specific behavior. Research supports
reasoning; author/platform guidelines supply scoped recommendations. This skill's
thresholds, decision trees, audit procedure, and checklists are engineering
synthesis unless explicitly attributed. Do not present them as universal standards.

## Contents

- Foundations: [S01 Parnas](#s01), [S02 Bloch](#s02), [S03 Liskov and Wing](#s03).
- SOLID: [S04 SRP](#s04), [S05 OCP](#s05), [S06 ISP](#s06), [S07 DIP](#s07).
- Signatures and collaboration: [S08 C++](#s08), [S09 Swift](#s09), [S10 Parameter Object](#s10), [S11 Flags](#s11), [S12 Tell Don't Ask](#s12), [S13 Constructor Over-injection](#s13).
- PHP: [S14 PSR-11](#s14), [S15 PSR-7](#s15), [S16 Interfaces](#s16), [S17 Properties](#s17), [S18 Visibility](#s18), [S34 Ownership and copies](#s34).
- TypeScript/JavaScript: [S19 Object types](#s19), [S20 Classes](#s20), [S21 Compatibility](#s21), [S35 Object.freeze](#s35), [S39 Node.js events](#s39).
- Binary storage and transfer: [S41 Buffer views](#s41), [S42 Structured cloning](#s42).
- Kotlin and API evolution: [S22 Compatibility](#s22), [S23 Visibility/properties](#s23), [S24 Functions](#s24), [S25 SemVer](#s25), [S26 .NET parameters](#s26), [S31 LocalDate](#s31), [S36 Collections/copy](#s36).
- Architecture and behavior: [S27 Role Interface](#s27), [S28 Service Layer](#s28), [S29 Transaction Script](#s29), [S30 CQS/CQRS](#s30), [S32 Ports and Adapters](#s32), [S33 Outbox](#s33).
- Ownership and lifecycle: [S37 CERT mutable inputs/outputs](#s37), [S38 Constructor escape](#s38), [S40 DI lifetimes](#s40).

## S01

**David L. Parnas — On the Criteria To Be Used in Decomposing Systems into Modules
(1972).** Research paper, university-hosted author text:
[paper](https://www.cs.lafayette.edu/~gexia/cs301/resources/parnas.html).

Use for decomposition around hidden design decisions and the cost of unnecessary
ordering guarantees. Information hiding extends beyond `private`. The HTML
transcription warns that digitization may contain inaccuracies.

## S02

**Joshua Bloch — Bumper-Sticker API Design.** Author guidance:
[article](https://www.infoq.com/articles/API-Design-Joshua-Bloch/).

Use client scenarios, minimal accessibility, meaningful names, and conceptual
simplicity. These are heuristics, with examples influenced by Java API experience;
method counts alone do not measure usability.

## S03

**Barbara Liskov and Jeannette M. Wing — A Behavioral Notion of Subtyping (1994).**
Research with a formal model:
[paper](https://www.cs.columbia.edu/~wing/publications/LiskovWing94.pdf).

Evaluate substitution through behavior, preconditions, postconditions, invariants,
and history constraints. Matching signatures are insufficient; differences that
the contract permits do not themselves violate subtyping.

## S04

**Robert C. Martin — The Single Responsibility Principle.** Author explanation:
[article](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html).

Relate responsibilities to reasons for change and the needs of a coherent business
function. Do not derive a one-method-per-class rule.

## S05

**Robert C. Martin — The Open-Closed Principle.** Author explanation:
[article](https://objectmentor.com/resources/articles/ocp.pdf).

Strategic Closure limits protection to selected likely changes; closure against
every change is impossible. Do not use OCP to require speculative extension points
or forbid corrections to existing code.

## S06

**Robert C. Martin — The Interface Segregation Principle.** Author explanation:
[article](https://objectmentor.com/resources/articles/isp.pdf).

Separate unrelated operation groups around client needs; one object can implement
several roles. Method count does not replace cohesion or justify one interface
for every method.

## S07

**Robert C. Martin — The Dependency Inversion Principle.** Author explanation:
[article](https://objectmentor.com/resources/articles/dip.pdf).

Use for the direction of dependencies among policy, abstractions, and details.
The discussion includes C without abstract classes, so DIP is not a requirement
for a language-level interface per class or for a DI container.

## S08

**C++ Core Guidelines — I.23, I.24, C.2, C.3, C.9, C.10, C.121, C.129.**
Evolving professional guidance:
[guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

I.23 encourages fewer than four parameters and allows exceptions; it is neither
an ISO C++ requirement nor a universal limit. I.24 highlights interchangeable
adjacent arguments. C.2 concerns invariants and data representation; C.9 concerns
minimizing exposure. C.3/C.10 and C.121/C.129 support deliberate separation of
interfaces, concrete types, and implementation inheritance. A pure abstract class
can express an interface in C++; do not generalize C++ cost claims to other languages.

## S09

**Swift — API Design Guidelines.** Official platform guidance:
[guidelines](https://www.swift.org/documentation/api-design-guidelines/).

Use for call-site clarity, naming, argument labels, and defaults. Transfer the
priority of clarity over brevity thoughtfully; do not impose Swift conventions
unchanged on PHP, TypeScript, or Kotlin.

## S10

**Martin Fowler — Introduce Parameter Object.** Author refactoring catalog:
[entry](https://refactoring.com/catalog/introduceParameterObject.html).

Use for grouping semantically related inputs into a named concept. The entry
sets no universal numerical threshold and does not justify a generic config bag
of unrelated data or dependencies.

## S11

**Martin Fowler — Flag Argument.** Author guidance:
[article](https://martinfowler.com/bliki/FlagArgument.html).

Evaluate flags selecting different operations and methods expressing client intent.
The author allows useful boolean cases, including forwarding an existing boolean
value; do not ban the type universally.

## S12

**Martin Fowler — Tell Dont Ask.** Author discussion:
[article](https://martinfowler.com/bliki/TellDontAsk.html).

Look for decisions misplaced away from the data and behavior they belong with.
The article also rejects eliminating every query/getter: information access and
layer separation can be appropriate.

## S13

**Mark Seemann — On Constructor Over-injection.** Author DI guidance:
[article](https://blog.ploeh.dk/2018/08/27/on-constructor-over-injection/).

Treat many dependencies as a diagnostic signal, not a self-sufficient defect.
Dependency catalogs can hide symptoms. Extract cohesive behavior or decorators
when actual responsibilities justify them.

## S14

**PHP-FIG — PSR-11 Container Interface and Meta Document.** Specification and rationale:
[PSR-11](https://www.php-fig.org/psr/psr-11/),
[meta](https://www.php-fig.org/psr/psr-11/meta/).

Preserve `SHOULD NOT` for injecting a container into an ordinary object to obtain
its own dependencies; do not restate it as `MUST NOT`. The meta document discusses
routers and factories, qualifying its factory example with a replaceable factory
interface. Repeated `get()` calls do not universally guarantee the same instance.

## S15

**PHP-FIG — PSR-7 HTTP Message Interfaces.** Specification:
[PSR-7](https://www.php-fig.org/psr/psr-7/).

Message modification methods preserve the original message state and return an
instance with the change. Streams can remain mutable: content, cursor position,
close, and detach operations have observable consequences. Do not infer an
immutable resource graph from immutable-style messages.

## S16

**PHP Manual — Object Interfaces and Signature Compatibility.** Official language
documentation: [interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php),
[signatures](https://www.php.net/manual/en/language.oop5.basic.php).

Check public interface methods, implementation compatibility, and parameter names
for named calls. Adding an optional parameter still requires checking existing
implementers. PHP 8.4+ interfaces can also require public property read/write
capabilities; they are not restricted exclusively to methods.

## S17

**PHP Manual — Properties.** Official language documentation:
[properties](https://www.php.net/manual/en/language.oop5.properties.php).

`readonly` does not prohibit interior mutation of a stored object/resource.
PHP 8.3 allows readonly property reinitialization in `__clone()`; PHP 8.4 prohibits
indirect modification through a reference in that hook. Check version labels in
the current manual, including newer additions, before applying them to PHP 8.4/8.5.

## S18

**PHP Manual — Visibility.** Official language documentation:
[visibility](https://www.php.net/manual/en/language.oop5.visibility.php).

Use the actual semantics of public/protected/private and PHP 8.4+ asymmetric
property visibility. Starting design with minimal access is a policy, not a
claim that PHP makes every declaration private by default.

## S19

**TypeScript Handbook — Object Types and Utility Types.** Official language
documentation: [objects](https://www.typescriptlang.org/docs/handbook/2/objects.html),
[Readonly](https://www.typescriptlang.org/docs/handbook/utility-types.html#readonlytype).

Use typed options and optional/readonly properties accurately. `readonly` and
`Readonly<T>` constrain assignments through types; they do not establish deep
or runtime immutability. Follow nested values and other mutable aliases.

## S20

**TypeScript Handbook — Classes.** Official language documentation:
[classes](https://www.typescriptlang.org/docs/handbook/2/classes.html).

Distinguish type-checking restrictions of TypeScript private/protected from
JavaScript `#field` privacy at runtime. Neither replaces application authorization
or protects secrets from the owner of the execution environment.

## S21

**TypeScript Handbook — Type Compatibility.** Official language documentation:
[compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html).

Account for structural compatibility and its limitations. Compiler acceptance
does not prove domain invariants or behavioral substitution. Class names alone
do not distinguish matching instance types; discriminants/brands or private/protected
members with the appropriate declaration origin can distinguish intended roles.

## S22

**Kotlin — Backward Compatibility Guidelines for Library Authors.** Official
guidance: [compatibility](https://kotlinlang.org/docs/api-guidelines-backward-compatibility.html).

Separate source, binary, and behavioral compatibility. A new default argument can
preserve source calls while changing a JVM signature needed by compiled clients.
Check platform and generated signatures; do not prescribe experimental migration
annotations without version/toolchain verification.

## S23

**Kotlin — Visibility Modifiers, API Guidelines: Simplicity, and Properties.**
Official documentation/guidance:
[visibility](https://kotlinlang.org/docs/visibility-modifiers.html),
[simplicity](https://kotlinlang.org/docs/api-guidelines-simplicity.html),
[properties](https://kotlinlang.org/docs/properties.html).

Use deliberate visibility, including module-level `internal`, and avoid needless
public abstractions. Kotlin's default visibility is public. `val` does not imply
deep immutability; a computed getter can produce a fresh value on each access.

## S24

**Kotlin — Coding Conventions and Functions.** Official guidance/documentation:
[conventions](https://kotlinlang.org/docs/coding-conventions.html),
[functions](https://kotlinlang.org/docs/functions.html).

Use named arguments and defaults for readable calls, especially for repeated
primitive types. These features can avoid unnecessary wrappers but do not repair
an incoherent function responsibility.

## S25

**Semantic Versioning 2.0.0.** Versioning specification:
[SemVer](https://semver.org/spec/v2.0.0.html).

First identify the declared public API and adopted version policy. For stable
nonzero-major versions, incompatible public API changes require a major version.
Adding a method/field does not prove compatibility; check implementers, callers,
and behavioral guarantees.

## S26

**Microsoft — Parameter Design, Framework Design Guidelines.** Officially published
historical guidance:
[parameter design](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/parameter-design).

Use optionally for .NET naming, types, and input validation. The page identifies
the material as originating in the 2008 edition and warns that some advice may
be outdated. Do not use it as a universal current standard or the main basis here.

## S27

**Martin Fowler — Role Interface (22 December 2006).** Author guidance:
[article](https://martinfowler.com/bliki/RoleInterface.html).

Define an interface around a particular supplier/consumer interaction rather than
copying all public class methods. The article does not set an implementation-count
threshold; this skill's selection algorithm also weighs present benefits and costs.

## S28

**Randy Stafford — Service Layer, Patterns of Enterprise Application Architecture.**
Author pattern catalog:
[entry](https://martinfowler.com/eaaCatalog/serviceLayer.html).

Define an application boundary and available operations, coordinating responses
and transactions. It does not require a language-level interface for every service.
The page is hosted by Martin Fowler; the pattern author is Randy Stafford.

## S29

**Martin Fowler — Transaction Script.** Author pattern catalog:
[entry](https://martinfowler.com/eaaCatalog/transactionScript.html).

Organize business logic as a procedure for a request/use case, extracting shared
subtasks where useful. Do not impose a rich domain model on every small application
or prohibit all domain checks in a handler.

## S30

**Martin Fowler — Command Query Separation and CQRS.** Author explanations:
[CQS](https://martinfowler.com/bliki/CommandQuerySeparation.html),
[CQRS](https://martinfowler.com/bliki/CQRS.html).

Strict CQS uses queries without observable state change and commands without
returned values; useful exceptions are discussed. CQRS separates read/write models.
Separate databases are an option, not a necessary consequence of either principle.

## S31

**Kotlin kotlinx-datetime — LocalDate.** Official library documentation:
[type contract](https://kotlinlang.org/api/kotlinx-datetime/kotlinx-datetime/kotlinx.datetime/-local-date/).

A calendar date has no time zone and does not identify an instant. Use this as a
counterexample to requiring a zone for every date. Mapping local dates/times to
instants requires an explicit conversion contract.

## S32

**Alistair Cockburn — Hexagonal Architecture, original 2005 article.**
Author architecture description:
[article](https://alistair.cockburn.us/hexagonal-architecture).

Separate application semantics from UI and infrastructure through ports/adapters,
including operation without real external devices. The article does not prescribe
a fixed port count or an interface declaration for every class.

## S33

**Chris Richardson — Transactional Outbox.** Author pattern catalog:
[entry](https://microservices.io/patterns/data/transactional-outbox.html).

Store an outgoing message in the same transaction as the relevant data, then relay
it. Publication can repeat, so outbox alone does not prove exactly-once delivery.
Apply it to actual consistency/delivery requirements, not every orchestrator.

## S34

**PHP Manual — Object identity, references, copying, and immutable dates.**
Official documentation:
[objects and references](https://www.php.net/manual/en/language.oop5.references.php),
[object cloning](https://www.php.net/manual/en/language.oop5.cloning.php),
[arrays](https://www.php.net/manual/en/language.types.array.php),
[what references do](https://www.php.net/manual/en/language.references.whatdo.php),
[DateTimeImmutable::createFromInterface](https://www.php.net/manual/en/datetimeimmutable.createfrominterface.php),
[DateTimeImmutable::modify](https://www.php.net/manual/en/datetimeimmutable.modify.php).

Distinguish an object identifier from an explicit variable reference. Ordinary
array copies preserve value semantics while nested objects and explicit reference
elements need separate ownership analysis. Object cloning starts shallow.
Use date normalization/new-value operations with their documented versions;
do not generalize the date example to arbitrary object graphs.

## S35

**ECMAScript 2025 — Object.freeze and SetIntegrityLevel.** Language specification:
[Object.freeze](https://tc39.es/ecma262/2025/multipage/fundamental-objects.html#sec-object.freeze),
[SetIntegrityLevel](https://tc39.es/ecma262/2025/multipage/abstract-operations.html#sec-setintegritylevel).

These algorithms set object extensibility and own-property restrictions; they do
not recursively freeze referenced values. Infer guarantees from the relevant
operations and descriptors, not the method name. Freezing an outer plain-data
object does not establish immutability of its entire reachable graph.

## S36

**Kotlin — Collections Overview, Create Collections, and Data Classes.** Official
documentation: [collection interfaces](https://kotlinlang.org/docs/collections-overview.html),
[collection copies](https://kotlinlang.org/docs/constructing-collections.html#copy),
[data-class copy](https://kotlinlang.org/docs/data-classes.html#copying).

A read-only interface does not exclude other mutable aliases. Collection copies
capture membership while retaining references to elements; generated data-class
`copy()` is shallow. Distinguish structural snapshots from deep immutable values.

## S37

**SEI CERT Oracle Coding Standard for Java — OBJ05-J and OBJ06-J.** Secure coding
guidance: [mutable output exposure](https://cmu-sei.github.io/secure-coding-standards/sei-cert-oracle-coding-standard-for-java/rules/object-orientation-obj/obj05-j/),
[mutable input retention](https://cmu-sei.github.io/secure-coding-standards/sei-cert-oracle-coding-standard-for-java/rules/object-orientation-obj/obj06-j/).

Use complementary input/output examples to trace representation exposure. A shallow
container copy can suffice for immutable elements. When copying defensively, validate
the retained copy rather than a changeable original. Keep Java examples and threat
assumptions scoped; this is not a requirement to clone every argument in every language.

## S38

**SEI CERT Oracle Coding Standard for Java — TSM01-J.** Secure coding guidance:
[constructor escape](https://cmu-sei.github.io/secure-coding-standards/sei-cert-oracle-coding-standard-for-java/rules/thread-safety-miscellaneous-tsm/tsm01-j/).

Inspect publication of partially initialized objects through callbacks, foreign
calls, or shared storage, including subclass initialization. The transferable
audit question is premature observation; Java memory-ordering rules and remedies
must remain language-specific.

## S39

**Node.js — Events.** Official runtime documentation:
[listener receiver](https://nodejs.org/api/events.html#passing-arguments-and-this-to-listeners),
[synchronous dispatch](https://nodejs.org/api/events.html#asynchronous-vs-synchronous),
[off](https://nodejs.org/api/events.html#emitteroffeventname-listener).

Ordinary EventEmitter listeners receive the emitter as `this`; dispatch invokes
listeners synchronously. Inspect receivers and reentrancy when wrapping event APIs.
`off()` also returns the emitter, so inspect forwarded cleanup return values.
This is documented EventEmitter behavior, not a defect by itself or a rule for
every event/callback system.

## S40

**Microsoft — .NET Dependency Injection Guidelines.** Official platform guidance:
[captive dependencies](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines#captive-dependency).

A longer-lived service can retain a shorter-lived dependency beyond its intended
scope. Use this as a concrete lifetime-mismatch example; do not assume every DI
container enforces .NET's registrations or scope-validation behavior.

## S41

**Node.js — Buffer.** Official runtime documentation:
[buffers and typed arrays](https://nodejs.org/api/buffer.html#buffers-and-typedarrays),
[underlying buffer](https://nodejs.org/api/buffer.html#bufbuffer),
[buffer views](https://nodejs.org/api/buffer.html#static-method-bufferfromarraybuffer-byteoffset-length).

Distinguish byte copies from views and inspect the selected overload. `Buffer.slice()`
shares storage; `.buffer` may expose more memory than the visible range. Pooled
allocations require care when reasoning about independent ownership.

## S42

**WHATWG HTML Standard — Structured Data.** Platform specification:
[serialization](https://html.spec.whatwg.org/multipage/structured-data.html#structuredserializeinternal),
[deserialization](https://html.spec.whatwg.org/multipage/structured-data.html#structureddeserialize),
[transfer](https://html.spec.whatwg.org/multipage/structured-data.html#structuredserializewithtransfer).

Shared-buffer serialization preserves shared backing data; ordinary-buffer
serialization copies bytes. Deserialization can create distinct objects around
shared data. Transferring an eligible ordinary buffer detaches its source.
Respect host restrictions, including browser cross-origin isolation requirements;
do not equate cloning, copying, sharing, and ownership transfer.
