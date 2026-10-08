# Coupling and cohesion

Two measures determine whether a model is any good: cohesion says whether each class has one reason to exist, and coupling says how far a change travels when something needs to change. A design can have the right classes and the right relationships and still be fragile if each class tries to do everything and knows too much about its neighbors.

All examples use one domain: an order processing system with orders, customers, shipping, payments, and notifications.

## Cohesion

Cohesion measures how closely related the responsibilities inside one class are. A class with high cohesion does one thing clearly while a class with low cohesion does several unrelated things and becomes a magnet for more.

Three levels are worth recognizing by name:

- Functional cohesion: every method serves one clearly bounded purpose. An `Order` that manages its own state and enforces its own invariants, nothing else.
- Communicational cohesion: methods operate on the same data for different reasons. An `OrderManager` that both fetches orders and sends their confirmation emails — related only because both touch `Order`, not because they are the same kind of work.
- Coincidental cohesion: methods are grouped by convenience or legacy, with no meaningful relationship. A `Utils` class that formats prices, validates addresses, and sends HTTP requests is the extreme case.

The sign of low cohesion in an interview diagram is a class with methods in different conceptual categories, or a dependency list that spans infrastructure (database, email, HTTP) alongside domain logic. Both signal that the class is accumulating its neighbors' responsibilities.

### The god order class

A common first draft for order processing:

```python
class OrderService:
    def create_order(self, customer_id, items): ...
    def calculate_tax(self, order): ...
    def apply_discount(self, order, code): ...
    def validate_inventory(self, order): ...
    def charge_payment(self, order, card): ...
    def send_confirmation(self, order): ...
    def create_shipment(self, order): ...
```

Each group represents a different concern: domain invariants, pricing rules, infrastructure (payment gateway), infrastructure (email), infrastructure (shipping carrier). Changing the email template and changing the discount logic require opening the same file.

The split follows the concerns:

```python
class Order: ...                      # entity: state and its invariants
class TaxCalculator: ...              # domain service: pricing rules
class DiscountEngine: ...             # domain service: coupon logic
class InventoryChecker: ...           # domain service: reads current stock
class PaymentGateway(Protocol): ...   # port: interface to payment infrastructure
class NotificationService: ...        # infrastructure: email and SMS
class ShippingService: ...            # infrastructure: carrier integration
```

Each class now changes for one kind of reason. `Order` changes when the business rules for orders change. `NotificationService` changes when the notification channel changes. No change crosses a boundary it does not need to cross.

## Coupling

Coupling measures how much a change in one module forces changes in another. Some coupling is necessary — a class that uses another class depends on it — but unnecessary coupling makes a design brittle.

The useful levels to name:

| Level   | Description                                           | Example                                            |
| ------- | ----------------------------------------------------- | -------------------------------------------------- |
| Content | Module reads or modifies the internals of another     | `order._items.append(item)` from outside           |
| Common  | Two modules share global or class-level mutable state | `PENDING_ORDERS: list[Order] = []` at module level |
| Control | One module passes a flag telling another what to do   | `process(order, send_email=True)`                  |
| Data    | Modules share only what they need via parameters      | `calculate_tax(order: Order) -> Decimal`           |
| Message | Modules communicate only through method calls         | `order.submit()`                                   |

Data coupling and message coupling are the goals. Content and common coupling appear when encapsulation breaks down and they are design smells, not design choices.

### Control coupling

Control coupling hides branching inside the called module, which forces the caller to know about the callee's internals in order to set the flag correctly:

```python
def calculate(order: Order, include_tax: bool, apply_discount: bool) -> Decimal:
    total = sum(item.price for item in order.items)
    if include_tax:
        total *= Decimal("1.15")
    if apply_discount and order.coupon:
        total -= order.coupon.value
    return total
```

The caller must pass `True` or `False` in a way that is meaningful only if it knows whether the order has a coupon and whether the tax rule applies. The fix is separate methods whose names carry the distinction:

```python
def subtotal(order: Order) -> Decimal:
    return sum(item.price for item in order.items)

def with_tax(amount: Decimal, rate: Decimal) -> Decimal:
    return amount * (1 + rate)

def with_coupon(amount: Decimal, coupon: Coupon) -> Decimal:
    return amount - coupon.value
```

Each function does exactly one thing. The combination is the caller's responsibility, and the caller can read what it is doing.

## Law of Demeter

The Law of Demeter constrains what a method may call: only the object itself, its parameters, objects it constructs, and its direct fields. It should not chain through other objects to reach something deeper.

The rule prevents the pattern where one class reaches through another to get at a third, forming a navigation chain that ties all three together structurally.

### Violation

```python
class Order:
    def shipping_label(self) -> str:
        return f"{self.customer.address.street}, {self.customer.address.city}"
```

`Order` now knows the internal structure of both `Customer` and `Address`. If `Address` gains a new field or `Customer` changes how it exposes its address, `Order.shipping_label` changes too.

### Fix

Move the question to the object that can answer it:

```python
class Address:
    def one_line(self) -> str:
        return f"{self.street}, {self.city}"


class Customer:
    def shipping_address(self) -> str:
        return self.address.one_line()


class Order:
    def shipping_label(self) -> str:
        return self.customer.shipping_address()  # one hop
```

Each class exposes behavior, not structure. `Order` asks a question and `Customer` answers it without revealing how.

### What a violation reveals

A Law of Demeter violation usually means one of two things:

- The intermediate object is hiding something the outer object legitimately needs, which means the intermediate object needs a new method
- or the outer object is doing work that belongs to the intermediate object, which means the responsibility is in the wrong class.

The chain `a.b().c().d()` shows up in a sequence diagram as an arrow that passes through an object without that object contributing anything. That is the god-object symptom seen from the side, and it is also the coupling symptom described in [note 07](07-uml-behavioral-diagrams.md).

## Connascence

Connascence names exactly what two things must agree on. If changing one requires changing the other, they are connascent. The vocabulary is more precise than "coupled" because it identifies what must be kept in sync, which points at the fix.

The types, roughly ordered from easiest to hardest to find and fix:

- Name: two things must agree on a name. Renaming a method requires updating every call site which is expected.
- Type: two things must agree on a type. A method returning `str` where a `Decimal` is expected.
- Meaning: two things must agree on what a value means. Two modules that both hardcode `"PENDING"` for the same order status — if one changes the string, the other silently uses the old one.
- Position: the caller must pass arguments in the right order. `create_order("Alice", "USD")` and `create_order("USD", "Alice")` are both valid strings. Wrong order is a silent bug.
- Algorithm: two things must implement the same algorithm independently. A checksum computed on both the client and server. Changing one breaks the verification.
- Execution: two things must run in a specific order. `connect()` before `send()`.
- Identity: two things must refer to exactly the same object, not just an equal one. Two threads that must share the same lock instance.

The two dimensions that determine whether a connascence is serious:

- **Strength**: name and type are weak (tools find and fix them) while algorithm and identity are strong (hard to find, easy to miss).
- **Locality**: within a class or module, any level is tolerable; across package or service boundaries, even weak connascence should be questioned.

The goal is to **weaken** the strongest connascences first, and to **localize** what cannot be weakened — keep it within one class or one module rather than letting it spread across the codebase.

### Using connascence at a round

Instead of "this is too coupled," a connascence claim names what must be kept in sync and points at the fix.

"These two modules are connascent in meaning — both interpret the integer `2` as express shipping. Replacing both literals with a shared `ShippingTier` enum removes the connascence and makes future additions safe to each call site independently."

"Passing four positional strings to this constructor is a positional connascence. Introducing a `dataclass` parameter object removes the ordering dependency and makes each argument self-documenting at the call site."

Connascence is also the vocabulary for the refactoring conversation. Moving from meaning to name (replace magic values with named constants), from position to name (replace positional arguments with a parameter object), or from algorithm to name (extract the shared computation) are all steps on the same axis.
