---
title: "Prototype Design Pattern: Problem and Practical Solutions"
date: 2026-04-28 00:00:00 +0700
categories: [OOP Fundamental, Design Pattern]
tags: [prototype, creational-pattern, python]
---

## The Problem

Sometimes you need to create a new object that is very similar to an existing one — with only minor differences. The naive approach is to construct the new object from scratch:

```python
original = Circle(color="Red", radius=5.0, points=[Point(x=0.0, y=0.0)])

# Want a slightly different copy:
cloned = Circle(
    color=original.color,
    radius=10.0,
    points=original.points,   # BUG: shares the same list!
)
```

This has two distinct problems:

1. **Tedious and fragile:** You must manually copy every field. Add a new field to `Circle` and you must remember to update every place that does this manual copying.
2. **Shallow copy pitfall:** Assigning `points=original.points` passes the *same list reference* — mutating the clone's list will also mutate the original. This is one of the most common bugs in Python:

```python
cloned.points.append(Point(x=9.0, y=9.0))
print(len(original.points))  # 2 — BUG! original was modified
```

We need a way to produce an independent copy of an object — one that owns all of its data — without the caller needing to know the internal structure of the class.

## The Solution: The Prototype Pattern

The Prototype pattern delegates the cloning responsibility to the object itself. Each class implements a `clone()` method that returns a fully independent copy. The caller only calls `.clone()` — it never needs to know which fields exist or which ones are mutable.

The key participants are:
1. **Prototype (Abstract):** Declares the `clone()` interface (optional in Python — duck typing works fine).
2. **Concrete Prototype:** Each class implements `clone()`, typically using `copy.deepcopy(self)` to handle nested mutable structures correctly.
3. **Client:** Calls `original.clone()` and gets back a fully independent instance it can freely mutate.

> **`copy.deepcopy` vs `copy.copy`:** `copy.copy` creates a *shallow* copy — the top-level object is new, but nested objects (lists, other class instances) are still shared references. `copy.deepcopy` recursively copies the entire object graph, giving you a fully independent clone. Unless you are certain all fields are immutable scalars, always use `deepcopy` in `clone()`.

Here are three real-world scenarios demonstrating the Prototype pattern in Python.

### Example 1: Cloneable Shapes

A `Cloneable` abstract base class defines the `clone()` interface. `Circle` and `Rectangle` each implement it using `copy.deepcopy`. The demo proves that mutating the clone — including nested `Point` objects inside the list — does not affect the original.

```python
from __future__ import annotations

import copy
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List


@dataclass
class Point:
    x: float
    y: float


class Cloneable(ABC):
    @abstractmethod
    def clone(self) -> Cloneable:
        pass


class Circle(Cloneable):
    def __init__(
        self,
        color: str,
        radius: float,
        points: List[Point] | None = None,
    ) -> None:
        self.color: str = color
        self.radius: float = radius
        self.points: List[Point] = list(points) if points is not None else []

    def clone(self) -> Circle:
        # Nested mutables (list, Point instances): need a deep copy of the graph.
        return copy.deepcopy(self)

    def print_info(self) -> None:
        pts = ", ".join(f"({p.x},{p.y})" for p in self.points) or "(none)"
        print(f"Circle [Color: {self.color}, Radius: {self.radius}, Points: {pts}]")


class Rectangle(Cloneable):
    def __init__(
        self,
        color: str,
        width: float,
        height: float,
        points: List[Point] | None = None,
    ) -> None:
        self.color: str = color
        self.width: float = width
        self.height: float = height
        self.points: List[Point] = list(points) if points is not None else []

    def clone(self) -> Rectangle:
        # Do not use Rectangle(..., points=self.points): that passes the same list
        # reference (and the same Point objects), so append/pop and in-place edits
        # to points would be visible on both original and clone.
        return copy.deepcopy(self)

    def print_info(self) -> None:
        pts = ", ".join(f"({p.x},{p.y})" for p in self.points) or "(none)"
        print(f"Rectangle [Color: {self.color}, W: {self.width}, H: {self.height}, Points: {pts}]")


if __name__ == "__main__":
    original: Circle = Circle(
        color="Red",
        radius=5.0,
        points=[Point(x=0.0, y=0.0), Point(x=1.0, y=0.0)],
    )
    cloned: Circle = original.clone()
    cloned.radius = 10.0
    cloned.points.append(Point(x=2.0, y=2.0))
    cloned.points[0].x = 99.0

    print("Circle: clone has extra point and moved first point; original unchanged?")
    original.print_info()
    cloned.print_info()

    print()

    rect: Rectangle = Rectangle(
        color="Blue",
        width=4.0,
        height=6.0,
        points=[Point(x=0.0, y=0.0)],
    )
    cloned_rect: Rectangle = rect.clone()
    cloned_rect.width = 8.0
    cloned_rect.points[0].y = 3.0

    print("Rectangle: width only on clone; point edit on clone does not touch original")
    rect.print_info()
    cloned_rect.print_info()
```

The key insight from the comment in `Rectangle.clone()`: manually passing `points=self.points` to the constructor is *not* a safe clone — it shares the list reference. `copy.deepcopy` is the correct and complete solution.

### Example 2: Resume Template

A common real-world use: you have a template object with shared baseline data, and you want to spin off variants by cloning and tweaking. Here a base `Resume` is cloned into a Backend and a Frontend variant — each gets its own independent skills list.

```python
from __future__ import annotations

import copy
from typing import List


class Resume:
    def __init__(
        self,
        name: str,
        email: str,
        title: str,
        skills: List[str],
    ) -> None:
        self.name: str = name
        self.email: str = email
        self.title: str = title
        self.skills: List[str] = list(skills)

    def clone(self) -> Resume:
        # skills is mutable; deepcopy gives a new list (and new strings are fine).
        return copy.deepcopy(self)

    def add_skill(self, skill: str) -> None:
        self.skills.append(skill)

    def print_resume(self) -> None:
        print("--- Resume ---")
        print(f"Name: {self.name}")
        print(f"Email: {self.email}")
        print(f"Title: {self.title}")
        print(f"Skills: {self.skills}")


if __name__ == "__main__":
    template: Resume = Resume(
        name="Alice",
        email="alice@mail.com",
        title="Software Engineer",
        skills=["Java", "SQL", "Git"],
    )

    backend: Resume = template.clone()
    backend.title = "Backend Engineer"
    backend.add_skill(skill="Spring Boot")
    backend.add_skill(skill="Kafka")

    frontend: Resume = template.clone()
    frontend.title = "Frontend Engineer"
    frontend.add_skill(skill="React")
    frontend.add_skill(skill="TypeScript")

    template.print_resume()
    backend.print_resume()
    frontend.print_resume()
```

`template` is never mutated — it stays as the master baseline. Both `backend` and `frontend` own their own `skills` lists and can grow independently. This is the **prototype registry** idiom: keep one well-constructed prototype and stamp out variants cheaply.

### Example 3: Document with Nested Sections

A more complex graph: a `Document` contains a list of `Section` objects, each of which contains a list of `Paragraph` objects. Cloning at the `Document` level must recursively clone the entire tree — exactly what `copy.deepcopy` does automatically.

```python
from __future__ import annotations

import copy
from typing import List


class Paragraph:
    def __init__(self, text: str, bold: bool = False) -> None:
        self.text: str = text
        self.bold: bool = bold

    def clone(self) -> Paragraph:
        return copy.deepcopy(self)

    def __repr__(self) -> str:
        return f"**{self.text}**" if self.bold else self.text


class Section:
    def __init__(self, title: str, paragraphs: List[Paragraph]) -> None:
        self.title: str = title
        self.paragraphs: List[Paragraph] = list(paragraphs)

    def clone(self) -> Section:
        return copy.deepcopy(self)

    def add_paragraph(self, p: Paragraph) -> None:
        self.paragraphs.append(p)

    def print_section(self) -> None:
        print(f"  ## {self.title}")
        for p in self.paragraphs:
            print(f"    {p}")


class Document:
    def __init__(self, title: str, author: str, sections: List[Section]) -> None:
        self.title: str = title
        self.author: str = author
        self.sections: List[Section] = list(sections)

    def clone(self) -> Document:
        return copy.deepcopy(self)

    def add_section(self, s: Section) -> None:
        self.sections.append(s)

    def print_doc(self) -> None:
        print(f"# {self.title} by {self.author}")
        for s in self.sections:
            s.print_section()


if __name__ == "__main__":
    template: Document = Document(
        title="Annual Report",
        author="Team",
        sections=[
            Section(
                title="Introduction",
                paragraphs=[Paragraph(text="Welcome to the report.", bold=False)],
            ),
            Section(
                title="Summary",
                paragraphs=[Paragraph(text="Key findings below.", bold=True)],
            ),
        ],
    )

    q1_report: Document = template.clone()
    q1_report.title = "Q1 Report"
    q1_report.sections[0].paragraphs[0].text = "Q1 was strong."
    q1_report.sections[1].add_paragraph(Paragraph(text="Revenue up 15%.", bold=False))

    print("=== TEMPLATE ===")
    template.print_doc()
    print("\n=== Q1 REPORT ===")
    q1_report.print_doc()
```

Editing `q1_report.sections[0].paragraphs[0].text` touches a deeply nested object. Because `deepcopy` recursed into the entire `Document → Section → Paragraph` graph, the template's paragraph text is completely untouched.

## Shallow Copy vs Deep Copy: The Critical Distinction

This is the most important concept to internalize when implementing Prototype:

| | `copy.copy` (shallow) | `copy.deepcopy` (deep) |
|---|---|---|
| New top-level object? | ✅ Yes | ✅ Yes |
| New nested lists? | ❌ No — shared reference | ✅ Yes — fully independent |
| New nested objects? | ❌ No — shared reference | ✅ Yes — recursively copied |
| When safe to use | All fields are immutable scalars (`int`, `str`, `float`) | Any field is a list, dict, or another mutable object |
| Performance | Faster | Slower (proportional to object graph size) |

**Rule:** If in doubt, use `copy.deepcopy`. The bugs from a wrong shallow copy are subtle and painful — mutations on the "clone" silently corrupt the original.

## Conclusion

The Prototype pattern is the right tool when:
- **Cloning is cheaper than constructing:** The original object was expensive to build (e.g., loaded from a database, computed from a heavy algorithm), and you want many variants of it.
- **You need template instances:** Keep one well-configured prototype and stamp out variants by cloning and tweaking, rather than re-running the full initialization logic each time.
- **The object graph is complex:** Rather than manually copying every field and nested structure, delegate the responsibility to `clone()` — the class knows its own internals best.

The pattern is deceptively simple in Python (`return copy.deepcopy(self)` is often all you need), but the shallow-vs-deep copy distinction is a genuine trap that catches even experienced developers.
