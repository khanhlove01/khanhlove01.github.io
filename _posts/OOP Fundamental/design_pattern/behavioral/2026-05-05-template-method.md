---
title: "Template Method Design Pattern: Locking the Algorithm Skeleton, Letting Steps Vary"
date: 2026-05-05 01:00:00 +0700
categories: [OOP Fundamental, Design Pattern]
tags: [template-method, behavioral-pattern, python]
---

## The Problem

You have several algorithms that all follow the **same overall sequence of steps**, but the individual steps differ. The naive approach is to copy the whole algorithm into each subclass and tweak the parts that change. That works for two variants, then becomes a maintenance trap.

A concrete example: brewing beverages. Tea and coffee both follow the same shape — boil water, brew, pour into a cup, add condiments — only the brew and condiments steps differ:

```python
class TeaMaker:
    def prepare(self) -> None:
        print("Boiling water...")
        print("Steeping tea bag...")
        print("Pouring into cup...")
        print("Adding lemon...")

class CoffeeMaker:
    def prepare(self) -> None:
        print("Boiling water...")
        print("Dripping coffee...")
        print("Pouring into cup...")
        print("Adding sugar and milk...")
```

The two `prepare` methods share most of their code, but each is a separate copy. The serious problems:

1. **Duplicated structure:** The "boil → brew → pour → condiments" skeleton is repeated. Add a step (warm the cup) and you must edit every variant.
2. **No enforced order:** Nothing stops a future subclass from pouring before brewing. The required sequence lives only in convention.
3. **No place for shared logic:** Steps that *should* be identical (boiling water, pouring) have no canonical home — they are reimplemented in every subclass.

## The Solution: The Template Method Pattern

The Template Method pattern moves the **algorithm's skeleton** into a single non-overridable method on a base class. That method calls a sequence of helper methods, some of which are concrete (shared across all subclasses) and some abstract (subclasses must supply). The skeleton's order is fixed; only the variable steps change.

The key participants are:

1. **Abstract Base Class:** Defines the *template method* — a concrete method that calls a fixed sequence of steps. It also declares the steps as either concrete (default implementation) or abstract (must override).
2. **Concrete Steps:** Methods on the base class that are the same for every subclass (e.g., `_boil_water`, `_pour_in_cup`).
3. **Abstract Steps:** Methods declared `@abstractmethod` — each subclass must supply its own version.
4. **Hook Methods:** Concrete methods on the base class with a *default* implementation that subclasses *may* override (e.g., `apply_discount` does nothing by default; subclasses override when relevant).
5. **Concrete Subclasses:** Implement the abstract steps and optionally override hooks. They never override the template method itself.

> **Template Method vs Strategy:** Both vary part of an algorithm. Template Method varies *steps* through inheritance — the skeleton is fixed in a base class. Strategy varies the *whole algorithm* through composition — the caller plugs a different object in. Template Method when the shape is invariant and only steps change; Strategy when the entire algorithm is interchangeable.

Three real-world scenarios in Python.

### Example 1: Beverage Maker

The minimal case. `BeverageMaker.prepare_beverage()` is the template method; `_boil_water` and `_pour_in_cup` are shared concrete steps; `brew` and `add_condiments` are abstract.

```python
from abc import ABC, abstractmethod


class BeverageMaker(ABC):
    def prepare_beverage(self) -> None:
        self._boil_water()
        self.brew()
        self._pour_in_cup()
        self.add_condiments()

    def _boil_water(self) -> None:
        print("Boiling water...")

    def _pour_in_cup(self) -> None:
        print("Pouring into cup...")

    @abstractmethod
    def brew(self) -> None: ...

    @abstractmethod
    def add_condiments(self) -> None: ...


class TeaMaker(BeverageMaker):
    def brew(self) -> None:
        print("Steeping the tea bag...")

    def add_condiments(self) -> None:
        print("Adding lemon...")


class CoffeeMaker(BeverageMaker):
    def brew(self) -> None:
        print("Dripping coffee through filter...")

    def add_condiments(self) -> None:
        print("Adding sugar and milk...")


if __name__ == "__main__":
    TeaMaker().prepare_beverage()
    CoffeeMaker().prepare_beverage()
```

The subclasses cannot accidentally call steps in the wrong order — they don't even know the order; they just provide the variable pieces. The "warm the cup" change happens in exactly one place: the base class.

### Example 2: Order Processor

A real-world processor for an e-commerce order. The skeleton is invariant — every order is validated, totaled, optionally discounted, paid for, and confirmed — but each processor type implements those steps differently.

```python
from abc import ABC, abstractmethod


class Order:
    def __init__(self, id: str, subtotal: float) -> None:
        self.id, self.subtotal = id, subtotal


class OrderProcessor(ABC):
    def process_order(self, order: Order) -> None:
        self.validate_order(order)
        self.calculate_total(order)
        self.apply_discount(order)
        self.process_payment(order)
        self.send_confirmation(order)
        print(f"Order {order.id} complete.")

    @abstractmethod
    def validate_order(self, order: Order) -> None: ...

    @abstractmethod
    def calculate_total(self, order: Order) -> None: ...

    @abstractmethod
    def process_payment(self, order: Order) -> None: ...

    def apply_discount(self, order: Order) -> None:
        pass  # Hook — no discount by default

    def send_confirmation(self, order: Order) -> None:
        print(f"Sending email confirmation for {order.id}")


class StandardOrderProcessor(OrderProcessor):
    def validate_order(self, order: Order) -> None:
        print("Validating standard order...")

    def calculate_total(self, order: Order) -> None:
        total = order.subtotal + 5.99
        print(f"Standard total: ${total:.2f}")

    def process_payment(self, order: Order) -> None:
        print("Charging via standard gateway...")


class PrimeOrderProcessor(OrderProcessor):
    def validate_order(self, order: Order) -> None:
        print("Validating Prime order — checking membership...")

    def calculate_total(self, order: Order) -> None:
        print(f"Prime total: ${order.subtotal:.2f} (free shipping)")

    def apply_discount(self, order: Order) -> None:
        print("Applying 10% Prime member discount.")

    def process_payment(self, order: Order) -> None:
        print("Charging via Prime billing...")


class InternationalOrderProcessor(OrderProcessor):
    def validate_order(self, order: Order) -> None:
        print("Validating international order — customs, restrictions...")

    def calculate_total(self, order: Order) -> None:
        total = order.subtotal + 24.99 + order.subtotal * 0.15
        print(f"International total: ${total:.2f} (incl. shipping + customs)")

    def process_payment(self, order: Order) -> None:
        print("Charging with currency conversion...")

    def send_confirmation(self, order: Order) -> None:
        print(f"Sending multi-language confirmation for {order.id}")


if __name__ == "__main__":
    StandardOrderProcessor().process_order(Order("ORD-001", 49.99))
    PrimeOrderProcessor().process_order(Order("ORD-002", 149.99))
    InternationalOrderProcessor().process_order(Order("ORD-003", 89.99))
```

`apply_discount` is a **hook**: the base provides a no-op default, and only `PrimeOrderProcessor` overrides it. `send_confirmation` is also a hook with a meaningful default — only `InternationalOrderProcessor` overrides to add multi-language. Hooks are how the pattern stays open to extension without forcing every subclass to think about every step.

### Example 3: Build Pipeline (Hooks Made Explicit)

A CI pipeline where the skeleton is fetch → compile → test → package → deploy → notify. The Java pipeline overrides every step including deploy; the Python pipeline relies on the *default-skip* hook for `deploy()`.

```python
from abc import ABC, abstractmethod


class BuildPipeline(ABC):
    def run_build(self) -> None:
        self._fetch_source()
        self.compile_sources()
        self.run_tests()
        self.package_artifact()
        self.deploy()
        self._notify_team()

    def _fetch_source(self) -> None:
        print("Fetching source from repo...")

    def _notify_team(self) -> None:
        print("Notifying team: build complete.")

    @abstractmethod
    def compile_sources(self) -> None: ...

    @abstractmethod
    def run_tests(self) -> None: ...

    @abstractmethod
    def package_artifact(self) -> None: ...

    def deploy(self) -> None:
        print("Skipping deployment (not configured).")  # Hook default


class JavaBuildPipeline(BuildPipeline):
    def compile_sources(self) -> None: print("Compiling with javac...")
    def run_tests(self) -> None:        print("Running JUnit tests...")
    def package_artifact(self) -> None: print("Packaging as JAR...")
    def deploy(self) -> None:           print("Deploying JAR to Nexus.")


class PythonBuildPipeline(BuildPipeline):
    def compile_sources(self) -> None: print("Running pylint (no compile)...")
    def run_tests(self) -> None:        print("Running pytest...")
    def package_artifact(self) -> None: print("Packaging as wheel...")
    # deploy() not overridden — uses the default-skip hook


if __name__ == "__main__":
    JavaBuildPipeline().run_build()
    print()
    PythonBuildPipeline().run_build()
```

The Python pipeline gets safe behavior for free — it doesn't have to know that `deploy` exists. When CI later adds a *security scan* step in the base class, every existing subclass gets it without being touched, as long as a sensible default is provided.

## When to Use the Template Method Pattern

| Situation | Without Template Method | With Template Method |
|---|---|---|
| Algorithm shape repeated across variants | Whole algorithm copied per subclass | Skeleton defined once in the base class |
| Step order matters | Convention only, easy to break | Order is enforced by the template method |
| Some steps are shared, some vary | Shared steps duplicated | Shared steps live on the base; abstract methods declare the variable ones |
| Optional steps that some variants skip | Conditional logic in every subclass | Hook methods with default behavior |

## Conclusion

The Template Method pattern is the right tool when:

- **The high-level algorithm is invariant:** The sequence of steps is the same across variants; only the contents of certain steps change.
- **You want to enforce that order:** The base class owns the skeleton and subclasses cannot reorder steps without overriding the template method (which they shouldn't).
- **You want to share concrete steps:** Steps that are identical across variants live on the base class — there is exactly one copy.

The pattern's strength is **inversion of control**: the base class calls into the subclass at well-defined extension points, instead of the subclass calling into the base. Subclasses never decide *when* something happens — they only decide *what* happens at the points where they're asked.

A note on hooks: a hook with a no-op or sensible default is the seam that keeps the pattern open to extension. Without hooks, every new step in the skeleton breaks every existing subclass. With hooks, the skeleton can grow without forcing every subclass to grow with it.
