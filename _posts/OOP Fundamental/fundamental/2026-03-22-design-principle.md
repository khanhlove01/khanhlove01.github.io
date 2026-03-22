---
title: Design Principles for Maintainable Code
description: DRY, KISS, YAGNI, Law of Demeter, separation of concerns, coupling & cohesion, and composition over inheritance—each with a plain definition and before/after style examples.
author: Tran Minh Khanh
date: 2026-03-22 20:00:00 +0700
categories: [OOP Fundamental, Fundamental]
tags: [design, dry, kiss, yagni, solid-adjacent, lld]
---

**Design principles** are guardrails: they do not replace judgment, but they reduce **duplication**, **accidental complexity**, and **rigid structure**. Below, each item starts with a **short idea**, then **what the name means**, then a **contrast**—usually “without the principle” vs “with the principle”—so the tradeoff is obvious.

---

## 1. DRY

**Idea:** Every piece of knowledge should have **one authoritative place** in the system. When rules change, you should not hunt dozens of copies.

**Stands for:** **D**on’t **R**epeat **Y**ourself.

### Not DRY (repeated logic)

```python
def email_receipt_order(order_id: str, total: float, to: str) -> None:
    if "@" not in to or "." not in to:
        raise ValueError("bad email")
    # send mail...


def email_welcome_user(name: str, to: str) -> None:
    if "@" not in to or "." not in to:  # same rule copied
        raise ValueError("bad email")
    # send mail...
```

### With DRY (single validation)

```python
def assert_valid_email(to: str) -> None:
    if "@" not in to or "." not in to:
        raise ValueError("bad email")


def email_receipt_order(order_id: str, total: float, to: str) -> None:
    assert_valid_email(to)
    ...


def email_welcome_user(name: str, to: str) -> None:
    assert_valid_email(to)
    ...
```

**Caveat:** DRY is about **knowledge**, not character-level deduplication. Two lines that *look* similar but encode **different business rules** should sometimes stay separate ([rule of three](https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming))—wait until the pattern is real).

---

## 2. KISS

**Idea:** Prefer the **simplest design that works**. Extra abstraction, frameworks, or indirection have a cost: readers must carry more in their heads.

**Stands for:** **K**eep **I**t **S**imple, **S**tupid (the “stupid” is humility toward complexity—not insulting teammates).

### Not KISS (over-engineered for the problem)

```python
class AbstractStrategyFactoryBuilder:
    def create_strategy_factory(self): ...
    # layers before we even have two implementations
```

### With KISS (direct until pressure appears)

```python
def discount(price: float, is_member: bool) -> float:
    return price * 0.9 if is_member else price
```

When you have **several** discount rules *in production*, introduce structure—KISS says **don’t start** there.

---

## 3. YAGNI

**Idea:** Build **what you need now**, not every feature you imagine “someday.” Unused generality is still code you must read, test, and secure.

**Stands for:** **Y**ou **A**in’t **G**onna **N**eed **I**t.

### Not YAGNI (speculative hooks)

```python
class ReportExporter:
    def export(self, fmt: str, theme: str, locale: str, future_ai_mode: bool):
        # future_ai_mode never used; formats that don't exist yet
        ...
```

### With YAGNI (minimal surface)

```python
class ReportExporter:
    def to_pdf(self, rows: list[dict]) -> bytes:
        ...
```

Add `to_csv` or plugins when a **real** requirement shows up.

---

## 4. Law of Demeter

**Idea:** A method should talk to **its direct collaborators**, not dig through their internals (“only one dot” is a rough mnemonic—more precisely: **limited structural knowledge**).

**Stands for:** A **law** of **good manners** between objects (Demeter = Greek goddess of the harvest; name is historical). Often summarized as **don’t talk to strangers**.

### Not Demeter (long train, tight to structure)

```python
# Order knows how Customer stores Address details
city = order.customer.address.city.upper()
```

### With Demeter (ask the neighbor)

```python
city = order.customer_city()  # Customer exposes intent
# or order.delivery_city() if that's the real question
```

**Why it helps:** fewer classes break when `Address` moves or `customer` becomes optional.

---

## 5. Separation of concerns

**Idea:** Split the program so each part answers **one kind of question** (I/O vs domain rules vs persistence). Changes in one concern should not ripple everywhere.

**Stands for:** Not an acronym—**concerns** are axes of change (UI, business rules, storage, logging, etc.).

### Mixed concerns

```python
def place_order_http(request):
    data = json.loads(request.body)
    if data["qty"] < 1:
        return error(400)
    conn = sqlite3.connect("shop.db")
    conn.execute("INSERT INTO orders ...", ...)
    send_sms(data["phone"], "Thanks!")
    return ok()
```

### Separated concerns (same story, clearer boundaries)

```python
def place_order_http(request):
    cmd = parse_place_order(request)      # HTTP / validation shape
    result = order_service.place(cmd)     # domain
    return to_http_response(result)       # HTTP again
```

Files or modules can follow the same split even inside a small service.

---

## 6. Coupling and cohesion

**Idea:** You want **loose coupling** (modules depend on **stable, narrow** contracts) and **high cohesion** (things that change together live **together**).

**Stands for:** **Coupling** = how tightly A depends on B’s **internals**. **Cohesion** = how focused a module’s responsibilities are.

### High coupling, low cohesion (kitchen-sink module)

```python
# user_utils.py
def hash_password(s): ...
def send_email(to, body): ...
def calculate_tax(amount, region): ...
def render_invoice_html(inv): ...
```

Everything imports this file; any change risks unrelated callers.

### Lower coupling, higher cohesion

```python
# auth/passwords.py
def hash_password(s): ...

# mail/sender.py
def send_email(to, body): ...

# billing/tax.py
def calculate_tax(amount, region): ...
```

Callers depend on **small** modules; each file has **one** reason to change.

---

## 7. Composition over inheritance — and why?

**Idea:** Prefer **building behavior from parts** (`has-a`) over **deep derivation trees** (`is-a`) when behavior varies or combines in many ways.

**Stands for:** Not an acronym—a rule of thumb from the [GoF](https://en.wikipedia.org/wiki/Design_Patterns) era: **favor object composition over class inheritance**.

### Why inheritance hurts here

- **Fragile base class** — a change in the parent breaks distant children.
- **Explosion of subclasses** — every mix of features becomes a new type (`RedFastElectricSedan`…).
- **Diamond-shaped** multiple inheritance (in languages that allow it) complicates method resolution.

### Not composition (inheritance for variation)

```python
class Notifier:
    def notify(self, msg: str) -> None: ...


class EmailNotifier(Notifier): ...
class SmsNotifier(Notifier): ...
class EmailAndSmsNotifier(EmailNotifier, SmsNotifier):  # gets awkward fast
    ...
```

### With composition (plug collaborators)

```python
class Notifier:
    def __init__(self, channels: list):  # each channel: .send(msg)
        self.channels = channels

    def notify(self, msg: str) -> None:
        for c in self.channels:
            c.send(msg)
```

You **add or reorder** channels without new class names. This pairs well with **[class relationships](/posts/class-relationship/)** (composition vs association) and **[OOP pillars](/posts/four-characteristics-oop/)** (composition vs inheritance in behavior).

---

## How these fit together

| Principle | Main question it asks |
|-----------|------------------------|
| **DRY** | Did I duplicate a **rule** that will change together? |
| **KISS** | Is this the **smallest** clear solution? |
| **YAGNI** | Am I coding for a **real** requirement or a guess? |
| **Law of Demeter** | Am I exposing **stable facades** instead of chains? |
| **Separation of concerns** | Can I **swap** UI/DB/rules without rewriting everything? |
| **Coupling / cohesion** | Are dependencies **narrow** and modules **focused**? |
| **Composition > inheritance** | Should this variation be a **part** instead of a **subtype**? |

---

## Related notes

- **[Class relationships](/posts/class-relationship/)** — aggregation, composition, dependency in diagrams.
- **[SOLID](/posts/solid/)** — named letters for single responsibility, open/closed, Liskov, interface segregation, and dependency inversion (with DIP and injection).
- **[Basic OOP](/posts/basic-oop/)** — language mechanics (`ABC`, `classmethod`, etc.) behind clean boundaries.
