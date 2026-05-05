---
title: "Chain of Responsibility Design Pattern: Pass the Request Down the Line Until Someone Handles It"
date: 2026-05-05 07:00:00 +0700
categories: [OOP Fundamental, Design Pattern]
tags: [chain-of-responsibility, behavioral-pattern, python]
---

## The Problem

You have a request and a set of possible handlers. Some requests can be served by the first handler; others need to bubble up the line. The naive approach is one big `if/elif` ladder that decides which handler to call. That works for two cases, then becomes a knot.

A concrete example: an ATM dispensing cash. It has $100, $50, $20, and $10 bills, and it should hand out the fewest notes possible. The first attempt:

```python
def dispense(amount: int) -> None:
    if amount >= 100:
        h = amount // 100
        print(f"{h} x $100")
        amount %= 100
    if amount >= 50:
        f = amount // 50
        print(f"{f} x $50")
        amount %= 50
    if amount >= 20:
        t = amount // 20
        print(f"{t} x $20")
        amount %= 20
    if amount >= 10:
        print(f"{amount // 10} x $10")
```

This works. Now the bank wants to add $5 bills, then $2, then take $20 out of circulation in some regions. The serious problems:

1. **Adding a denomination edits the function:** Every change touches the same place — high blast radius.
2. **Reorder is fragile:** If you accidentally check $20 before $50, you over-pay in $20 bills. The order is implicit.
3. **No reuse:** A different machine that dispenses different denominations needs its own copy of the same shape.

## The Solution: The Chain of Responsibility Pattern

The Chain of Responsibility pattern wires handlers into a **linked list**. Each handler decides whether it can handle (or partly handle) the request, then passes the (possibly modified) request to the next handler in the chain. The caller talks only to the first handler; the chain figures out the rest.

The key participants are:

1. **Handler (Interface):** Declares `handle(request)` and a way to set the next handler (`set_next`).
2. **Base Handler:** Holds the link to the next handler and provides a default `forward(request)` that delegates to it.
3. **Concrete Handlers:** Each one decides what to do with the request — handle it fully, handle it partially and forward, or just forward unchanged.
4. **Client:** Builds the chain, sends the request to the first handler.

> **Stop on first vs run all:** Two flavors of the pattern. *Stop on first* — the first handler that "matches" terminates the chain (e.g., a ticket-routing system: the lowest-priority agent who can handle it does, no one else). *Run all* — every handler that applies runs, the request is passed through unconditionally (e.g., a discount stack: loyalty + bulk + coupon all stack). Both are valid; pick based on whether handlers are alternatives or contributors.

Three real-world scenarios in Python.

### Example 1: ATM Cash Withdrawal (Partial Handling)

The classic case. Each handler dispenses as many of *its* denomination as the remaining amount allows, then forwards the leftover to the next handler. Every handler runs, but each only takes what fits.

```python
from __future__ import annotations
from abc import ABC, abstractmethod


class CashRequest:
    def __init__(self, amount: int) -> None:
        self.amount = amount


class CashHandler(ABC):
    @abstractmethod
    def set_next(self, handler: CashHandler) -> None: ...
    @abstractmethod
    def dispense(self, request: CashRequest) -> None: ...


class BaseCashHandler(CashHandler):
    def __init__(self, denomination: int) -> None:
        self.denomination = denomination
        self._next: CashHandler | None = None

    def set_next(self, handler: CashHandler) -> None:
        self._next = handler

    def dispense(self, request: CashRequest) -> None:
        if request.amount >= self.denomination:
            count = request.amount // self.denomination
            print(f"  Dispensing {count} x ${self.denomination}")
            request.amount %= self.denomination
        if self._next is not None:
            self._next.dispense(request)


class HundredHandler(BaseCashHandler):
    def __init__(self) -> None: super().__init__(100)


class FiftyHandler(BaseCashHandler):
    def __init__(self) -> None: super().__init__(50)


class TwentyHandler(BaseCashHandler):
    def __init__(self) -> None: super().__init__(20)


class TenHandler(BaseCashHandler):
    def __init__(self) -> None: super().__init__(10)


if __name__ == "__main__":
    h100, h50, h20, h10 = HundredHandler(), FiftyHandler(), TwentyHandler(), TenHandler()
    h100.set_next(h50); h50.set_next(h20); h20.set_next(h10)

    print("--- Withdraw $380 ---")
    request = CashRequest(380)
    h100.dispense(request)
    print(f"Remaining: ${request.amount}")

    print("\n--- Withdraw $275 ---")
    request = CashRequest(275)
    h100.dispense(request)
    print(f"Remaining: ${request.amount}")
```

Adding a new denomination — say, $5 — is one new class and one extra `set_next` call. Removing one is one missing `set_next` link. The handlers themselves never change.

### Example 2: Support Ticket Escalation (Stop-on-First-Match)

A request goes to the front line first. If they can handle it (within their priority budget), they do. Otherwise they escalate to the next agent up the chain. The chain terminates as soon as someone qualified picks it up — or falls off the end if no one can.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from enum import Enum


class Priority(Enum):
    LOW      = 1
    MEDIUM   = 2
    HIGH     = 3
    CRITICAL = 4


class SupportTicket:
    def __init__(self, ticket_id: str, issue: str, priority: Priority) -> None:
        self.ticket_id = ticket_id
        self.issue = issue
        self.priority = priority
        self.resolved_by: str | None = None


class SupportAgent(ABC):
    def __init__(self, name: str, max_priority: Priority) -> None:
        self._name = name
        self._max_priority = max_priority
        self._next: SupportAgent | None = None

    def set_next(self, agent: SupportAgent) -> SupportAgent:
        self._next = agent
        return agent

    def handle(self, ticket: SupportTicket) -> None:
        if ticket.priority.value <= self._max_priority.value:
            self._resolve(ticket)
            ticket.resolved_by = self._name
            return
        if self._next is not None:
            print(f"  [{self._name}] {ticket.priority.name} too high — escalating")
            self._next.handle(ticket)
        else:
            print(f"  [{self._name}] no further handler — UNRESOLVED")

    @abstractmethod
    def _resolve(self, ticket: SupportTicket) -> None: ...


class FrontlineAgent(SupportAgent):
    def __init__(self) -> None:
        super().__init__("Frontline", Priority.LOW)

    def _resolve(self, t: SupportTicket) -> None:
        print(f"  [Frontline] resolved {t.ticket_id} via FAQ.")


class TechnicalAgent(SupportAgent):
    def __init__(self) -> None:
        super().__init__("Technical", Priority.MEDIUM)

    def _resolve(self, t: SupportTicket) -> None:
        print(f"  [Technical] resolved {t.ticket_id} after log review.")


class SeniorEngineer(SupportAgent):
    def __init__(self) -> None:
        super().__init__("Senior", Priority.HIGH)

    def _resolve(self, t: SupportTicket) -> None:
        print(f"  [Senior] resolved {t.ticket_id} via hotfix.")


class IncidentCommander(SupportAgent):
    def __init__(self) -> None:
        super().__init__("Incident-Commander", Priority.CRITICAL)

    def _resolve(self, t: SupportTicket) -> None:
        print(f"  [IC] resolved {t.ticket_id} — war room, post-mortem scheduled.")


if __name__ == "__main__":
    front = FrontlineAgent()
    front.set_next(TechnicalAgent()).set_next(SeniorEngineer()).set_next(IncidentCommander())

    tickets = [
        SupportTicket("TKT-001", "How do I reset my password?",        Priority.LOW),
        SupportTicket("TKT-002", "App crashes on login for 3 users",    Priority.MEDIUM),
        SupportTicket("TKT-003", "Payment service: intermittent 500s",  Priority.HIGH),
        SupportTicket("TKT-004", "Production database unreachable",     Priority.CRITICAL),
    ]
    for ticket in tickets:
        print(f"\n  Incoming {ticket.ticket_id} ({ticket.priority.name})")
        front.handle(ticket)
        print(f"  Resolved by: {ticket.resolved_by}")
```

Each agent declares the highest priority *it* can handle. The chain handles routing automatically: a low-priority ticket stops at Frontline; a critical incident bubbles up to the top.

### Example 3: E-commerce Discount Chain (Run-All Variant)

A different flavor of the pattern. Every discount handler runs unconditionally; each one decides whether to apply its discount based on the order's properties. The order accumulates the discounts.

```python
from __future__ import annotations
from abc import ABC, abstractmethod


class Order:
    def __init__(self, total: float, is_loyal: bool,
                 item_count: int, coupon_code: str = "") -> None:
        self.total = total
        self.is_loyal = is_loyal
        self.item_count = item_count
        self.coupon_code = coupon_code


class DiscountHandler(ABC):
    @abstractmethod
    def set_next(self, handler: DiscountHandler) -> None: ...
    @abstractmethod
    def apply(self, order: Order) -> None: ...


class BaseDiscountHandler(DiscountHandler):
    def __init__(self) -> None:
        self._next: DiscountHandler | None = None

    def set_next(self, handler: DiscountHandler) -> None:
        self._next = handler

    def forward(self, order: Order) -> None:
        if self._next is not None:
            self._next.apply(order)


class LoyaltyDiscount(BaseDiscountHandler):
    def apply(self, order: Order) -> None:
        if order.is_loyal:
            cut = order.total * 0.10
            order.total -= cut
            print(f"  Loyalty -10%   (-${cut:.2f})")
        self.forward(order)


class BulkDiscount(BaseDiscountHandler):
    def apply(self, order: Order) -> None:
        if order.item_count >= 10:
            cut = order.total * 0.05
            order.total -= cut
            print(f"  Bulk    -5%    (-${cut:.2f})")
        self.forward(order)


class CouponDiscount(BaseDiscountHandler):
    def apply(self, order: Order) -> None:
        if order.coupon_code == "SAVE20":
            cut = order.total * 0.20
            order.total -= cut
            print(f"  Coupon  -20%   (-${cut:.2f})")
        self.forward(order)


if __name__ == "__main__":
    order = Order(total=200.00, is_loyal=True, item_count=15, coupon_code="SAVE20")

    loyalty = LoyaltyDiscount()
    bulk    = BulkDiscount()
    coupon  = CouponDiscount()
    loyalty.set_next(bulk); bulk.set_next(coupon)

    print(f"Original: ${order.total:.2f}")
    loyalty.apply(order)
    print(f"Final:    ${order.total:.2f}")
```

The same chain shape, with a small change in the handler logic — each handler always forwards, never short-circuits — gives you a totally different behavior. The handlers compose naturally; reordering them changes the math (a 10% then 5% discount is *not* the same as 5% then 10% on the original).

## When to Use the Chain of Responsibility Pattern

| Situation | Without Chain | With Chain |
|---|---|---|
| Several handlers, one runs (or stops the chain) | `if/elif` ladder dispatching by type/priority | Each handler decides for itself |
| Several handlers, all may contribute | Flat for-loop with rule logic embedded | Linked handlers with self-contained rules |
| Add a handler | Edit the dispatcher | Insert a node in the chain |
| Reorder handlers | Edit the dispatcher | Re-link the chain |

## Conclusion

The Chain of Responsibility pattern is the right tool when:

- **You have a request and several possible handlers:** And the choice between them — or the order they run in — is policy, not structure.
- **Handlers are independent:** Each one knows its own rule and how to forward; none of them knows about the others.
- **The chain is a knob:** Different deployments, different regions, different feature flags can use different chains without touching the handlers.

The pattern's strength is **plug-and-play composition**. Each handler is a single, focused responsibility; the chain is just a list. New handler? New class. New ordering? New wiring. The dispatcher disappears — there isn't one anymore.

A note on shape: the canonical pattern uses a singly-linked list (each handler points to the next). In practice, many systems use a *list* — the framework iterates handlers in order and lets each one act. Same idea, slightly different mechanics. Both are "Chain of Responsibility" in the sense that matters: handlers are autonomous, ordered, and composable.
