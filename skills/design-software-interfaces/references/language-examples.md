# Language Examples and Caveats

## Contents

- [PHP: a complete operation instead of arbitrary writes](#php-a-complete-operation-instead-of-arbitrary-writes)
- [PHP: a domain parameter made from two values](#php-a-domain-parameter-made-from-two-values)
- [PHP: incoming and outgoing mutable aliases](#php-incoming-and-outgoing-mutable-aliases)
- [PHP: array copies and shallow clones](#php-array-copies-and-shallow-clones)
- [PHP: compatibility surface](#php-compatibility-surface)
- [TypeScript: options and alternatives](#typescript-options-and-alternatives)
- [TypeScript: readonly types and frozen objects](#typescript-readonly-types-and-frozen-objects)
- [TypeScript: privacy and type boundaries](#typescript-privacy-and-type-boundaries)
- [JavaScript: buffer views, copies, and shared storage](#javascript-buffer-views-copies-and-shared-storage)
- [Kotlin: read-only collections and snapshots](#kotlin-read-only-collections-and-snapshots)
- [Kotlin: visibility and ABI](#kotlin-visibility-and-abi)
- [Counterexamples to mechanical design](#counterexamples-to-mechanical-design)

These examples illustrate contract choices, not production storage, payments, or
concurrent reservation systems. Check syntax and feature availability against the
project's minimum version. PHP examples target PHP 8.1+ except the explicitly
marked PHP 8.2+ readonly class; PHP 8.4+ features are discussed separately.
TypeScript examples use features available in TypeScript 5.x/6.x; Kotlin examples
use standard language and collection features available in Kotlin 1.9/2.x. These
are compatibility scopes, not claims that every listed compiler was run.

Run the [state leak and encapsulation audit](state-leak-audit.md) when reviewing
ownership; use these examples to recognize language-specific paths and controls.

## PHP: a complete operation instead of arbitrary writes

Making a stock field private and adding an unrestricted setter is insufficient.
The caller needs a valid state transition, and the object must protect its local
invariant. This standalone PHP 8.1+ example performs one in-memory operation:

```php
<?php
declare(strict_types=1);

final class Stock
{
    public function __construct(private int $available)
    {
        if ($available < 0) {
            throw new InvalidArgumentException('Stock cannot be negative.');
        }
    }

    public function available(): int
    {
        return $this->available;
    }

    public function reserve(int $quantity): void
    {
        if ($quantity <= 0) {
            throw new InvalidArgumentException('Quantity must be positive.');
        }

        if ($quantity > $this->available) {
            throw new DomainException('Insufficient stock.');
        }

        $this->available -= $quantity;
    }
}
```

`available()` is a useful query; `reserve()` validates the complete transition.
A rejected call leaves state unchanged. This does not provide atomicity across
processes: a real store needs a transaction or conditional write and a conflict
policy. If callers need to manage the reservation afterward, the operation might
return a `Reservation` instead of `void`. The existence of `Stock` does not itself
justify a separate interface.

## PHP: a domain parameter made from two values

A shared invariant can justify a type even for two parameters. This PHP 8.2+
declaration represents a nonempty, half-open interval of instants `[from, to)`:

```php
<?php
declare(strict_types=1);

final readonly class DateRange
{
    public function __construct(
        public DateTimeImmutable $from,
        public DateTimeImmutable $to,
    ) {
        if ($from >= $to) {
            throw new InvalidArgumentException('Expected from < to.');
        }
    }
}

// Usage sketch; $reporter and its methods are not implemented here.
// $reporter->sales(period: new DateRange(from: $from, to: $to));
```

Public value components can preserve encapsulation. Calendar dates require a
different contract and boundary rules; a date need not have a time zone
[S31](sources.md#s31). A zone and conversion rules become necessary when mapping
local dates or times to instants. Do not treat this as a universal model of dates.

This example shares the endpoint instances and assumes their ordinary immutable
date-value operations; it does not promise independent endpoint identities or
isolation against arbitrary reinitialization hooks/subclass behavior. When the
boundary requires independently owned date objects, normalize them on intake and
return independent values as in the next example.

Replacing `DateTimeImmutable` with mutable `DateTime` would let `modify()` change
the referenced value despite `readonly` [S17](sources.md#s17). Returning that same
mutable object from a getter would not protect the invariant either. Choose an
immutable value, an appropriate copy, or explicit shared ownership.

## PHP: incoming and outgoing mutable aliases

Suppose a deadline must capture its supplied date and remain stable. This PHP
8.1+ implementation breaks that promise in both directions:

```php
<?php
declare(strict_types=1);

final class LeakyDeadline
{
    public function __construct(private readonly DateTime $dueAt)
    {
    }

    public function dueAt(): DateTime
    {
        return $this->dueAt;
    }
}

$input = new DateTime('2030-01-01T00:00:00+00:00');
$deadline = new LeakyDeadline($input);
$input->modify('+1 day');                  // Incoming alias changes stored state.
$deadline->dueAt()->modify('+1 day');      // Outgoing alias changes stored state.
echo $deadline->dueAt()->format('Y-m-d');   // 2030-01-03
```

Ordinary PHP object assignment, arguments, and returns copy an object identifier;
they do not clone the object. This sharing does not require PHP's `&` reference
operator [S34](sources.md#s34). `private readonly` protects access and reassignment
of the property, not mutation of the stored date [S17](sources.md#s17).

For this contract, capture an immutable date value on intake and return an
independent date value. This separate PHP 8.1+ example isolates both boundaries:

```php
<?php
declare(strict_types=1);

final class Deadline
{
    private readonly DateTimeImmutable $dueAt;

    public function __construct(DateTimeInterface $dueAt)
    {
        $this->dueAt = DateTimeImmutable::createFromInterface($dueAt);
    }

    public function dueAt(): DateTimeImmutable
    {
        return DateTimeImmutable::createFromInterface($this->dueAt);
    }
}

$input = new DateTime('2030-01-01T00:00:00+00:00');
$deadline = new Deadline($input);
$input->modify('+1 day');
$later = $deadline->dueAt()->modify('+1 day'); // Creates a different date value.
echo $deadline->dueAt()->format('Y-m-d');      // 2030-01-01
```

`createFromInterface()` creates a new immutable date with the input's date, time,
and zone information; `modify()` returns a new date value [S34](sources.md#s34).
The getter's additional copy makes independent result identity explicit. Sharing
an immutable value can also be sufficient when its supported operations and the
contract justify that choice. For mutable values, copying only in the getter
leaves the incoming alias; copying only in the constructor leaves an outgoing
alias. If mutation or shared identity is intentional, specify that contract
rather than applying this snapshot policy automatically.

## PHP: array copies and shallow clones

Array value copying does not clone object elements. Object cloning is shallow
unless the type defines the required copying behavior. Explicit references inside
an array can also survive an ordinary copy [S34](sources.md#s34). This standalone
PHP 8.1+ probe distinguishes these cases:

```php
<?php
declare(strict_types=1);

$stored = [
    'label' => 'Original',
    'dueAt' => new DateTime('2030-01-01T00:00:00+00:00'),
];
$copy = $stored;
$copy['label'] = 'Local';          // $stored['label'] remains 'Original'.
$copy['dueAt']->modify('+1 day');  // Both arrays still refer to the same date.

$box = new stdClass();
$box->dueAt = $stored['dueAt'];
$clonedBox = clone $box;
$clonedBox->dueAt->modify('+1 day'); // The original box's date changes too.

$quantity = 1;
$withReference = ['quantity' => &$quantity];
$referenceCopy = $withReference;
$referenceCopy['quantity'] = 9;    // Also sets $quantity and $withReference['quantity'].
```

Do not flag every returned PHP array as mutable shared state. Ordinary scalar
and nested-array values without `&` references retain value semantics; trace
objects, resources, and explicit references within them. For the date example,
mapping to immutable date values or scalar output can provide the required
snapshot. A blanket `clone`, recursive copy, or serialization round trip is not a
general ownership policy: inspect nested identities, resources, clone hooks, and
the promised behavior. PHP 8.3+ permits readonly-property reinitialization during
`__clone()`, while PHP 8.4+ forbids indirect modification such as acquiring a
reference to that property during the hook. Do not assume an arbitrary deep-copy
hook is legal on the minimum supported version [S17](sources.md#s17).

## PHP: compatibility surface

- Language-interface methods are public. In PHP 8.4+, interfaces can also declare
  public property read/write requirements; they are not limited to methods
  [S16](sources.md#s16).
- Preserve parameter names when implementing public interfaces. PHP permits
  different names, but named calls can break. Renaming `$key` to `$id` is not
  necessarily an internal refactoring [S16](sources.md#s16).
- Adding a required method requires implementations to provide it. Even an
  optional new parameter on a PHP interface method can make old implementations
  with shorter signatures incompatible. Consider a separate interface for an
  independent capability, and check callers and implementers [S16](sources.md#s16).
- Avoid constructors in behavioral interfaces unless creation is part of their
  purpose. A factory can express a necessary creation contract while leaving
  implementation assembly flexible.
- PHP 8.4+ `private(set)` limits writes, not necessarily reads. A public value with
  restricted assignment is different from a fully hidden property, and a mutable
  referenced object still needs ownership analysis [S18](sources.md#s18).
- Preserve PSR-11 and PSR-7 semantics as well as their method names. PSR-11 does
  not let consumers assume that every `get()` returns the same instance without
  an additional contract. PSR-7 message immutability does not make the reachable
  message-and-stream graph deeply immutable
  [S14](sources.md#s14), [S15](sources.md#s15).

## TypeScript: options and alternatives

Choose an options object for the operation's meaning; request data need not all
be optional. A discriminated union can express alternative pagination modes.
Numeric bounds still require runtime validation. This declaration and call-site
sketch omits the implementation of `searchProducts`:

```ts
type Pagination =
  | {
      readonly kind: 'offset';
      readonly offset: number;
      readonly limit: number;
    }
  | {
      readonly kind: 'cursor';
      readonly after?: string;
      readonly limit: number;
    };

interface SearchOptions {
  readonly pagination: Pagination;
  readonly sort?: 'priceAsc' | 'priceDesc';
}

interface ProductView {
  readonly id: string;
  readonly name: string;
}

declare function searchProducts(
  query: string,
  options: SearchOptions,
): Promise<ReadonlyArray<ProductView>>;

const result = searchProducts('keyboard', {
  pagination: { kind: 'offset', offset: 0, limit: 25 },
  sort: 'priceAsc',
});
```

Before publication, define the `limit` range, empty-query behavior, default sort,
cursor rules, errors, and ordering. Types alone do not establish these promises.
A TypeScript union neither validates arbitrary JSON nor provides general exact
object types; validate external input and define the extra-field policy.

`ReadonlyArray<ProductView>` restricts writes through that type. It does not
freeze the runtime array or create a snapshot. Provide any snapshot guarantee
through implementation and contract [S19](sources.md#s19).

## TypeScript: readonly types and frozen objects

`Readonly<T>` applies read-only properties at that type level; nested objects and
other mutable references can still change state [S19](sources.md#s19).
`Object.freeze()` operates on the supplied object's own properties and returns
that same object. It does not recursively freeze referenced objects
[S35](sources.md#s35). This complete TypeScript example demonstrates the failed
controls and then copies and freezes a known plain-data shape:

```ts
type RetryOptions = {
  policy: { maxAttempts: number };
};

const callerOwned: RetryOptions = { policy: { maxAttempts: 3 } };
const exposed: Readonly<RetryOptions> = callerOwned;
exposed.policy.maxAttempts = 7; // Allowed: the nested property is mutable.

const shallow = Object.freeze({ ...callerOwned });
shallow.policy.maxAttempts = 9; // Also changes callerOwned.policy.

const snapshot = Object.freeze({
  policy: Object.freeze({ maxAttempts: callerOwned.policy.maxAttempts }),
});
callerOwned.policy.maxAttempts = 11;
console.log(snapshot.policy.maxAttempts); // 9: no shared mutable nested object.
```

This control works for the shown shape because it copies its only mutable child
and freezes both plain objects. Validate domain constraints separately. If the
shape gains an array or another object, revisit the whole path. Freezing a
caller-owned object in place can also violate the caller's mutation contract;
copy first when independent ownership is promised. Do not generalize this
example into a universal deep-freeze helper: accessor behavior, class private
state, and resources need their own analysis.

## TypeScript: privacy and type boundaries

Distinguish TypeScript's type-checked `private`/`protected` from JavaScript `#field`
privacy enforced at runtime [S20](sources.md#s20). Neither mechanism makes an
object returned from a private field immutable, replaces authorization, or hides
an in-memory secret from the process owner.

Do not create a TypeScript interface merely to copy a class. Structural
compatibility permits an appropriate contract without explicit `implements`;
it does not prove behavioral guarantees [S21](sources.md#s21). Export consumer
types deliberately instead of exposing every internal module.

`type UserId = string` and `type OrderId = string` do not distinguish value roles.
Separate class names with matching public instance members are also insufficient.
When confusion matters, use types with distinct brands/discriminants or appropriate
private/protected members originating in different classes [S21](sources.md#s21).
Check how values are created and validated; a type assertion performs no validation.

## JavaScript: buffer views, copies, and shared storage

Object identity alone cannot establish storage isolation. Node.js `Buffer.slice()`
shares bytes, unlike ordinary typed-array `slice()`; `subarray()` is also a view
[S41](sources.md#s41). `structuredClone()` of shared memory preserves shared bytes
even through distinct wrapper objects [S42](sources.md#s42). This complete Node.js
example checks the observable behavior:

```javascript
import assert from "node:assert/strict";
import { Buffer } from "node:buffer";

const owner = Buffer.alloc(2);
owner.set([3, 5]);
const view = owner.slice(); // A view, despite the method name.
assert.notEqual(view, owner);
view[0] = 9;
assert.equal(owner[0], 9);

const copy = new Uint8Array(owner); // Copy this byte view into independent storage.
copy[0] = 1;
assert.equal(owner[0], 9);

const narrowView = owner.subarray(1, 2);
new Uint8Array(narrowView.buffer)[owner.byteOffset] = 7;
assert.equal(owner[0], 7); // Access beyond narrowView's visible range.

const shared = new Uint8Array(new SharedArrayBuffer(2));
const cloned = structuredClone(shared);
assert.notEqual(cloned, shared);
assert.notEqual(cloned.buffer, shared.buffer);
cloned[0] = 8;
assert.equal(shared[0], 8); // Distinct objects, shared underlying memory.

const ordinary = new Uint8Array([2, 6]);
const ordinaryClone = structuredClone(ordinary);
ordinaryClone[0] = 7;
assert.equal(ordinary[0], 2); // Control: ordinary buffer bytes were copied.
```

Check the actual overload and runtime. A byte view and its `.buffer` give different
copy/share behavior to constructors; some Node buffers also use pooled storage.
A view's bounds do not restrict all authority reachable through `.buffer`
[S41](sources.md#s41). If isolation is promised, inspect the reachable storage and
test relevant reads/writes; sharing alone is not a defect.

Transferring an ordinary transferable `ArrayBuffer` detaches the source; transfer
is an ownership change, not a transparent snapshot repair. Browser shared-memory
availability also has isolation restrictions [S42](sources.md#s42). Copying a
concurrently changing buffer still needs a consistency protocol if a coherent
snapshot is promised. Do not generalize this byte-copy control to arbitrary graphs.

## Kotlin: read-only collections and snapshots

`List<T>` omits collection mutation operations; it does not guarantee immutable
backing storage or immutable elements. `toList()` captures collection membership
but retains references to its elements [S36](sources.md#s36). This complete
Kotlin 1.9/2.x example contrasts a live alias, a shallow snapshot, and a projection
to independent string values:

```kotlin
data class Item(var name: String)

fun main() {
    val input = mutableListOf(Item("A"))
    val live: List<Item> = input
    val shallowSnapshot: List<Item> = input.toList()
    val nameSnapshot: List<String> = input.map { it.name }

    input.add(Item("B"))
    input[0].name = "Changed"

    check(live.size == 2)                       // The read-only alias is live.
    check(shallowSnapshot.size == 1)            // Membership was captured.
    check(shallowSnapshot[0].name == "Changed") // The mutable element is shared.
    check(nameSnapshot == listOf("A"))          // Captured string values stay A.
}
```

For a getter returning `List<Item>`, check both the collection and its elements;
`internalItems.toList()` alone cannot isolate mutable `Item` state. Returning
copied value projections can protect the owner. The projected list above is
independent of the original input, but its `List` type does not promise that every
runtime cast or Java caller is unable to mutate its backing collection. Choose
and verify an immutable collection implementation if that stronger guarantee is
required; its element types still need review. A documented live view may be the
right API, so do not report shared storage as a defect without a broken promise.

## Kotlin: visibility and ABI

Choose `private`, `internal`, and `public` deliberately. Visibility defaults to
`public`; `internal` is module-scoped. Consider explicit API mode for a published
library and check the selected toolchain [S23](sources.md#s23). `val` prevents
assignment through that declaration, not mutation of the referenced object. A
computed getter can even return a different reference each time
[S23](sources.md#s23).

Named arguments help distinguish primitive parameters of the same type, but do
not replace a responsibility review [S24](sources.md#s24). A `data class`
constructor also determines generated API such as `copy` and `componentN`;
include it in compatibility analysis instead of assuming that a new defaulted
field is automatically safe [S22](sources.md#s22).

For Kotlin/JVM, distinguish source from binary compatibility. Adding a defaulted
parameter can preserve source calls on recompilation while breaking an existing
compiled client. Preserve required old signatures and use the project's ABI
validation tools [S22](sources.md#s22). Do not transfer this platform-specific
rule unchanged to PHP or JavaScript, or prescribe experimental annotations
without checking the version.

## Counterexamples to mechanical design

| Proposal | How to assess it |
|---|---|
| Wrap a genuinely symmetric scalar operation in an options object. | Preventing argument interchange adds no benefit when symmetry includes the relevant edge cases and identity promises. |
| Replace every Boolean with an enum. | Check whether the value is genuinely binary. |
| Hide eight services in `ApplicationContext`. | The number of underlying dependencies has not decreased. |
| Store request inputs through setters so `run()` has no arguments. | Check lifecycle, isolation, reuse, and call order. |
| Make every DTO field private and add getters. | Identify an invariant or representation choice that this protects. |
| Every getter exposes implementation details. | Check what it returns and whether callers can break a promise. |
| A new interface method is compatible because the old methods remain. | Check all implementers and subclasses. |
| One universal `execute()` is the smallest API. | Count independent contracts and invalid combinations. |
| A cache must always return the same object. | Read the scope, identity, freshness, and mutation contract. |
| `readonly` makes configuration deeply immutable. | Trace nested values and other references to them. |
| A read-only list is a snapshot. | Determine whether membership and element values can still change. |
| Copying every object fixes ownership. | Check required identity, nested state, resources, and the cost of copying. |
