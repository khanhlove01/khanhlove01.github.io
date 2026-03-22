---
title: OOP Pillars — Encapsulation, Abstraction, Inheritance, Composition, Polymorphism
description: Definitions, key ideas, and short Python examples for the main OOP characteristics, plus a comparison table—aligned with an LLD intro study set.
author: Tran Minh Khanh
date: 2026-03-22 18:00:00 +0700
categories: [OOP Fundamental, Fundamental]
tags: [python, oop, encapsulation, inheritance, polymorphism, composition]
---

The **four classical pillars** are often listed as encapsulation, abstraction, inheritance, and polymorphism. **Composition** (“has-a”) is added here because it is the usual alternative to deep **inheritance** (“is-a”) in solid low-level design. Each section below follows: **definition → key ideas → one or two short examples**.

---

## 1. Encapsulation

**Definition:** Bundling **data** and the **methods** that operate on that data inside one unit, and **controlling access** so internal state cannot be changed in arbitrary or unsafe ways from outside.

**Key concepts**

- **Hide internals** — expose a small **public API** (`deposit`, `withdraw`) instead of letting callers tweak raw fields.
- **Python reality** — `_attr` (protected by convention) and `__attr` (name mangling) are **signals**, not hard walls like in Java; still use them for clarity and pair with **properties** or getters when you need validation.
- **Why it matters** — easier to **change implementation** later without breaking callers; invariants (e.g. balance ≥ 0) stay enforceable in one place.

**Example 1 — public API around internal balance**

```python
class BankAccount:
    def __init__(self, account_number: str, balance: float) -> None:
        self.account_number = account_number
        self._balance = balance  # internal; use methods to change

    def deposit(self, amount: float) -> None:
        if amount > 0:
            self._balance += amount

    def get_balance(self) -> float:
        return self._balance
```

**Example 2 — subclass may use protected members by convention**

```python
class SavingsAccount(BankAccount):
    def __init__(self, account_number: str, balance: float, rate: float) -> None:
        super().__init__(account_number, balance)
        self.rate = rate

    def apply_interest(self) -> None:
        self._balance += self._balance * (self.rate / 100)
```

---

## 2. Abstraction

**Definition:** Hiding **complexity** and showing only the **essential operations** a user of the type needs. You work with a **simple idea** (“start the vehicle”, “print this document”) without caring about every internal step.

**Key concepts**

- **Abstraction** answers *what* you can do; **encapsulation** often answers *how* state is protected while doing it—they work together but are not the same.
- **Abstract base classes (ABC)** and **interfaces** (in Python, often an ABC with only abstract methods) spell out the **contract**; concrete classes fill in **how**.
- **Goal** — reduce cognitive load and depend on **stable capabilities**, not on implementation details.

**Example 1 — abstract type, concrete cars**

```python
from abc import ABC, abstractmethod


class Vehicle(ABC):
    def __init__(self, brand: str) -> None:
        self.brand = brand

    @abstractmethod
    def start(self) -> None: ...

    def display_brand(self) -> None:
        print("Brand:", self.brand)


class Car(Vehicle):
    def start(self) -> None:
        print("Car is starting...")
```

**Example 2 — capability without caring about printer internals**

```python
class Document:
    def __init__(self, content: str) -> None:
        self._content = content

    def get_content(self) -> str:
        return self._content


class Printable(ABC):
    @abstractmethod
    def print(self, doc: Document) -> None: ...


class PdfPrinter(Printable):
    def print(self, doc: Document) -> None:
        print("PDF:", doc.get_content())
```

---

## 3. Inheritance

**Definition:** Deriving a **new class** from an **existing one**, reusing and extending behavior. The subclass **is-a** specialized kind of the base (**is-a** relationship).

**Key concepts**

- **`super()`** — reuse parent initialization and methods; override only what differs.
- **Method overriding** — same message, different behavior in subclass (ties directly to **runtime polymorphism**).
- **Watch-outs** — deep hierarchies become rigid; **Liskov** intuition: subclasses should remain usable where the base type is expected.

**Example 1 — vehicle specialization**

```python
class Car:
    def __init__(self, make: str, model: str) -> None:
        self.make = make
        self.model = model

    def start_engine(self) -> None:
        print(f"{self.make} {self.model}: engine started")


class ElectricCar(Car):
    def __init__(self, make: str, model: str, kwh: float) -> None:
        super().__init__(make, model)
        self.battery_kwh = kwh

    def charge(self) -> None:
        print(f"{self.make} {self.model}: charging battery")
```

**Example 2 — shared payment flow, different processors**

```python
class PaymentProcessor:
    def __init__(self, amount: float) -> None:
        self._amount = amount

    def process_payment(self) -> bool:
        raise NotImplementedError


class PayPalPayment(PaymentProcessor):
    def __init__(self, amount: float, email: str) -> None:
        super().__init__(amount)
        self._email = email

    def process_payment(self) -> bool:
        print(f"PayPal ${self._amount} for {self._email}")
        return True
```

---

## 4. Composition

**Definition:** Building types by **assembling** other objects (**has-a**). A `Car` **has an** `Engine`; behavior is **delegated** to the part instead of inheriting implementation from it.

**Key concepts**

- **Flexibility** — swap implementations (electric vs gas engine) at **runtime** without subclass explosion.
- **Favor composition over inheritance** when behavior varies on an **axis** that is not a stable “kind of” tree.
- **Contrast** — inheritance shares a **family**; composition models **parts and collaborators**.

**Example — car owns an engine interface**

```python
from abc import ABC, abstractmethod


class Engine(ABC):
    @abstractmethod
    def start(self) -> None: ...


class ElectricEngine(Engine):
    def start(self) -> None:
        print("Electric engine on (silent)")


class Car:
    def __init__(self, make: str, model: str, engine: Engine) -> None:
        self.make = make
        self.model = model
        self.engine = engine  # has-a Engine

    def start(self) -> None:
        print(f"{self.make} {self.model}: ", end="")
        self.engine.start()


tesla = Car("Tesla", "Model 3", ElectricEngine())
tesla.start()
```

---

## 5. Polymorphism

**Definition:** **Many shapes** of behavior behind **one interface**: the same call (e.g. `send(msg)`, `area()`) can run **different code** depending on the **actual type** of the object.

**Key concepts**

- **Runtime (dynamic) polymorphism** — usual in Python: **method overriding**; which implementation runs is chosen **while the program runs**.
- **“Compile-time” overloading** — languages like Java overload by signature; Python typically uses **default args**, `*args`, **`@singledispatch`**, or a single method with `isinstance` checks—same *idea*, different mechanism.
- **Duck typing** — “if it quacks like a duck”; no shared base class required, but ABCs help document intent.

**Example 1 — override: one interface, many senders**

```python
from abc import ABC, abstractmethod


class NotificationSender(ABC):
    @abstractmethod
    def send(self, msg: str) -> None: ...


class EmailSender(NotificationSender):
    def __init__(self, address: str) -> None:
        self.address = address

    def send(self, msg: str) -> None:
        print(f"Email to {self.address}: {msg}")


class SmsSender(NotificationSender):
    def __init__(self, phone: str) -> None:
        self.phone = phone

    def send(self, msg: str) -> None:
        print(f"SMS to {self.phone}: {msg}")


def notify(sender: NotificationSender, text: str) -> None:
    sender.send(text)  # runtime picks Email vs SMS


notify(EmailSender("a@b.com"), "hello")
```

**Example 2 — same method name, different shapes (classic `area`)**

```python
class Circle:
    def __init__(self, r: float) -> None:
        self.r = r

    def area(self) -> float:
        return 3.14159 * self.r**2


class Rectangle:
    def __init__(self, w: float, h: float) -> None:
        self.w = w
        self.h = h

    def area(self) -> float:
        return self.w * self.h


for shape in (Circle(1), Rectangle(2, 3)):
    print(shape.area())
```

---

## 6. Summary table

| Concept | One-line idea | Typical mechanism | Main benefit | Watch-out |
|--------|----------------|-------------------|--------------|-----------|
| **Encapsulation** | Hide state; expose safe operations | Private/protected by convention, methods, properties | Stable API, enforce invariants | Python: conventions, not enforcement |
| **Abstraction** | Show *what*, hide *how* | ABCs, small public surfaces | Less coupling to details | Don’t leak internals through “convenience” |
| **Inheritance** | Specialized *is-a* type | `class Child(Parent)`, `super()` | Reuse + override | Deep trees, fragile base class |
| **Composition** | *Has-a* collaborator | Attributes holding interfaces | Swap parts, flatter design | More wiring; still need clear interfaces |
| **Polymorphism** | One call, many behaviors | Overriding, protocols/ABCs | Extensible code (`notify`, `area`) | Overuse of `isinstance` chains |

---

## What pairs with this post

The companion note **[Basic OOP](/posts/basic-oop/)** covers class/object mechanics (`classmethod`, `dataclass`, `Enum`, ABC syntax). This article is about **why** those tools serve **encapsulation, abstraction, inheritance, composition, and polymorphism** in design.
