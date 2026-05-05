---
title: "Decorator Design Pattern: Stacking Behaviors at Runtime"
date: 2026-05-04 01:00:00 +0700
categories: [OOP Fundamental, Design Pattern]
tags: [decorator, structural-pattern, python]
---

## The Problem

You have a class that does its job fine, but you keep needing to add *optional* features on top of it — sometimes one, sometimes a combination. The naive answer is inheritance: subclass the base for every variant. That works for one or two features, then collapses.

A concrete example: a pizza shop. The base is a `PlainPizza`, and customers want to add cheese, olives, mushrooms — in any combination:

```python
class PlainPizza:
    def get_cost(self) -> float:    return 5.00
    def get_description(self) -> str: return "Plain pizza"
```

If you handle every combination with a subclass, you end up with `CheesePizza`, `OlivePizza`, `CheeseOlivePizza`, `CheeseMushroomPizza`, `CheeseOliveMushroomPizza`… With *N* optional toppings, you need up to **2^N** subclasses. Adding one new topping doubles the class count.

This is the classic *subclass explosion* problem. It also has companion pains:

1. **Combinations are decided at compile time:** A customer who wants cheese + olives at runtime can't get a class that doesn't already exist.
2. **Duplication:** Each combination subclass repeats the cost/description logic of the others.
3. **Open/Closed violated:** Adding a topping forces you to write a new subclass for every combination it can appear in.

## The Solution: The Decorator Pattern

The Decorator pattern lets you add responsibilities to an object **dynamically** by wrapping it in another object that shares the same interface. Each decorator forwards calls to the wrapped object and adds a small piece of behavior before or after. Because decorators implement the same interface as the component, they can wrap *each other* — giving you arbitrary combinations from a small set of building blocks.

The key participants are:

1. **Component (Interface):** Defines the operations the client uses (e.g., `Pizza` with `get_cost()` and `get_description()`).
2. **Concrete Component:** The base object being decorated (e.g., `PlainPizza`).
3. **Base Decorator:** Implements the Component interface and holds a reference to a wrapped Component. It forwards every call by default.
4. **Concrete Decorators:** Subclass the Base Decorator and add behavior — augmenting the result before or after the forwarded call.
5. **Client:** Works with a `Component` reference and never knows whether it holds a bare component or a stack of decorators.

> **Decorator vs Inheritance:** Inheritance fixes the combination at *class definition time*; Decorator composes them at *object construction time*. With decorators, `OliveDecorator(CheeseDecorator(PlainPizza()))` is a different runtime arrangement of the same building blocks — no new class needed.

Three real-world scenarios in Python.

### Example 1: Pizza Toppings

Each topping is a decorator that adds to the cost and appends to the description. Toppings can be stacked in any order and any combination — without writing a class per combination.

```python
from abc import ABC, abstractmethod


class Pizza(ABC):
    @abstractmethod
    def get_cost(self) -> float: ...

    @abstractmethod
    def get_description(self) -> str: ...


class PlainPizza(Pizza):
    def get_cost(self) -> float:        return 5.00
    def get_description(self) -> str:   return "Plain pizza"


class PizzaDecorator(Pizza):
    def __init__(self, pizza: Pizza) -> None:
        self.pizza = pizza

    def get_cost(self) -> float:        return self.pizza.get_cost()
    def get_description(self) -> str:   return self.pizza.get_description()


class CheeseDecorator(PizzaDecorator):
    def get_cost(self) -> float:        return self.pizza.get_cost() + 1.50
    def get_description(self) -> str:   return self.pizza.get_description() + ", cheese"


class OliveDecorator(PizzaDecorator):
    def get_cost(self) -> float:        return self.pizza.get_cost() + 2.00
    def get_description(self) -> str:   return self.pizza.get_description() + ", olives"


class MushroomDecorator(PizzaDecorator):
    def get_cost(self) -> float:        return self.pizza.get_cost() + 1.00
    def get_description(self) -> str:   return self.pizza.get_description() + ", mushrooms"


if __name__ == "__main__":
    plain = PlainPizza()
    print(f"{plain.get_description()} | ${plain.get_cost():.2f}")

    cheese_olive = OliveDecorator(CheeseDecorator(PlainPizza()))
    print(f"{cheese_olive.get_description()} | ${cheese_olive.get_cost():.2f}")

    loaded = MushroomDecorator(OliveDecorator(CheeseDecorator(PlainPizza())))
    print(f"{loaded.get_description()} | ${loaded.get_cost():.2f}")
```

Three toppings give eight possible pizzas. With pure inheritance that's eight classes; with decorators it's three classes plus a base — and adding a fourth topping is one new class, not eight more.

### Example 2: Notification Channels

A `Notifier` interface sends a message. The base `EmailNotifier` sends email. Decorators add SMS and Slack channels — stacked, you can fan a single call out to all of them. Adding a new channel is one new decorator.

```python
from abc import ABC, abstractmethod


class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...


class EmailNotifier(Notifier):
    def __init__(self, address: str) -> None:
        self._address = address

    def send(self, message: str) -> None:
        print(f"[Email] To {self._address}: {message}")


class NotifierDecorator(Notifier):
    def __init__(self, wrapped: Notifier) -> None:
        self._wrapped = wrapped

    def send(self, message: str) -> None:
        self._wrapped.send(message)


class SMSNotifier(NotifierDecorator):
    def __init__(self, wrapped: Notifier, phone: str) -> None:
        super().__init__(wrapped)
        self._phone = phone

    def send(self, message: str) -> None:
        self._wrapped.send(message)
        print(f"[SMS] To {self._phone}: {message}")


class SlackNotifier(NotifierDecorator):
    def __init__(self, wrapped: Notifier, channel: str) -> None:
        super().__init__(wrapped)
        self._channel = channel

    def send(self, message: str) -> None:
        self._wrapped.send(message)
        print(f"[Slack] #{self._channel}: {message}")


if __name__ == "__main__":
    base = EmailNotifier("alice@example.com")
    multi = SlackNotifier(SMSNotifier(base, "+1-555-0000"), "ops-alerts")
    multi.send("Order #1234 has failed payment.")
```

Note that each decorator can add its **own** constructor arguments (the phone number, the Slack channel) without polluting the base interface. The `Notifier` interface stays a single-method contract.

### Example 3: Function Decorators — Timing and Retry

Python has first-class support for the Decorator pattern at the function level via `@decorator` syntax. Cross-cutting concerns like timing, retry, caching, and authorization compose naturally.

```python
from functools import wraps
from time import perf_counter
from typing import Any, Callable


def timed(func: Callable[..., Any]) -> Callable[..., Any]:
    @wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed_ms = (perf_counter() - start) * 1000
            print(f"[Timed] {func.__name__} took {elapsed_ms:.1f}ms")

    return wrapper


def retry(times: int = 3) -> Callable:
    def decorator(func: Callable[..., Any]) -> Callable[..., Any]:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_exc: Exception | None = None
            for attempt in range(1, times + 1):
                try:
                    print(f"[Retry] Attempt {attempt}/{times} for {func.__name__}")
                    return func(*args, **kwargs)
                except Exception as exc:
                    last_exc = exc
            raise last_exc or RuntimeError("retry failed")

        return wrapper

    return decorator


_state = {"failures": 0}


@timed
@retry(times=2)
def flaky_operation() -> None:
    _state["failures"] += 1
    if _state["failures"] == 1:
        raise RuntimeError("simulated failure on first attempt")
    print("flaky_operation succeeded on retry")


flaky_operation()
```

The order matters: `@timed` is on top, so it wraps the *retry-wrapped* function and times the entire retry loop. Reversing the order would time each individual attempt instead. This is exactly the same wrapping mechanic as object decorators — just expressed with sugar.

## When to Use the Decorator Pattern

| Situation | Without Decorator | With Decorator |
|---|---|---|
| N optional features in any combination | 2^N subclasses | N decorator classes |
| Add behavior at runtime, not compile time | Need a different subclass | Wrap the existing object |
| Cross-cutting concerns (logging, retry, cache, auth) | Edit the core class for each one | Stack decorators around it |
| Add a new feature later | Modify or subclass existing classes | Add one new decorator |

## Conclusion

The Decorator pattern is the right tool when:

- **You need optional, combinable features:** Different callers want different combinations, and you don't want a class per combination.
- **The features are cross-cutting:** Logging, caching, retries, authorization — concerns that don't belong in the core class but need to wrap it.
- **You want runtime composition:** The combination is decided when an object is constructed, not when its class is defined.

The pattern's strength is **composition over inheritance**. Each decorator is a single, focused responsibility, and stacking them is how you build up complex behavior. When a new feature appears, you write one new decorator — the existing component and decorators stay untouched.

A note of caution: don't write a decorator that just forwards calls and adds nothing. A wrapper with no behavior is pure indirection. Decorators earn their cost by *doing something* on the way in, on the way out, or both.
