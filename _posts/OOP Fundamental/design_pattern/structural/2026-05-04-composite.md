---
title: "Composite Design Pattern: Treating Trees and Leaves Uniformly"
date: 2026-05-04 03:00:00 +0700
categories: [OOP Fundamental, Design Pattern]
tags: [composite, structural-pattern, python]
---

## The Problem

Many real-world structures are *recursive trees*: filesystems contain folders that contain files and other folders; menus contain items and submenus; HTML elements contain text and other elements. The leaves and the containers are different things, but a client almost always wants to do **the same operation** to both — compute total size, render, count items, delete.

A concrete example: a filesystem with `File` and `Folder`:

```python
class File:
    def __init__(self, name: str, size: int) -> None:
        self.name, self.size = name, size

class Folder:
    def __init__(self, name: str) -> None:
        self.name = name
        self.children: list = []
```

Now you want to compute the total size of a folder. Without a unifying abstraction, every traversal looks like this:

```python
def total_size(item) -> int:
    if isinstance(item, File):
        return item.size
    elif isinstance(item, Folder):
        return sum(total_size(child) for child in item.children)
    else:
        raise TypeError(f"Unknown item: {type(item)}")
```

This works, until it doesn't. The serious problems:

1. **`isinstance` checks proliferate:** `total_size`, `print_structure`, `delete`, `count_items` — every operation needs the same branching.
2. **Adding a new node type breaks all of them:** Add `Shortcut` or `CompressedFile`, and you must update every function that traverses the tree.
3. **Clients become fragile:** Code that walks the tree depends on knowing all the concrete types — exactly the coupling polymorphism is supposed to eliminate.

## The Solution: The Composite Pattern

The Composite pattern defines a **common interface** for both leaves and containers. Leaves implement the operations directly; containers implement them by delegating to their children and combining the results. The client calls the operation on the root and the same call recurses through the tree — no `isinstance`, no branching.

The key participants are:

1. **Component (Interface):** Declares the operations every node supports (e.g., `get_size()`, `print_structure()`, `delete()`).
2. **Leaf:** A node with no children. Implements the operations on its own data.
3. **Composite:** A node with children. Implements the operations by iterating its children and combining their results. May add child-management methods (`add_item`, `remove_item`).
4. **Client:** Holds a `Component` reference and calls the operations uniformly. It doesn't care whether it has a leaf or a composite.

> **Uniformity is the win.** Every operation becomes a single polymorphic call. New node types — leaf or composite — plug in without touching the existing client code or the existing nodes.

Three real-world scenarios in Python.

### Example 1: Filesystem with Multiple Leaf Types

A filesystem where `Folder` is the composite, and there are *three* kinds of leaf: `File`, `Shortcut` (a tiny pointer to another path), and `CompressedFile` (a file stored at a reduced on-disk size). All four implement the same `FileSystemItem` interface, and the `Folder` traversal works for any of them.

```python
from abc import ABC, abstractmethod


class FileSystemItem(ABC):
    @abstractmethod
    def get_size(self) -> int: ...

    @abstractmethod
    def print_structure(self, indent: str) -> None: ...

    @abstractmethod
    def delete(self) -> None: ...


class File(FileSystemItem):
    def __init__(self, name: str, size: int) -> None:
        self.name, self.size = name, size

    def get_size(self) -> int:
        return self.size

    def print_structure(self, indent: str) -> None:
        print(f"{indent}- {self.name} ({self.size} KB)")

    def delete(self) -> None:
        print(f"Deleting file: {self.name}")


class Shortcut(FileSystemItem):
    """A tiny pointer to another path — fixed 4 KB on disk."""
    def __init__(self, name: str, target_path: str) -> None:
        self.name, self.target_path = name, target_path

    def get_size(self) -> int:
        return 4

    def print_structure(self, indent: str) -> None:
        print(f"{indent}~ {self.name} -> {self.target_path} (4 KB shortcut)")

    def delete(self) -> None:
        print(f"Deleting shortcut: {self.name}")


class CompressedFile(FileSystemItem):
    """Stored at a fraction of its original size."""
    def __init__(self, name: str, original_size: int, ratio: float = 0.4) -> None:
        self.name, self.original_size, self.ratio = name, original_size, ratio

    def get_size(self) -> int:
        return int(self.original_size * self.ratio)

    def print_structure(self, indent: str) -> None:
        print(f"{indent}* {self.name} ({self.original_size} -> {self.get_size()} KB)")

    def delete(self) -> None:
        print(f"Deleting compressed file: {self.name}")


class Folder(FileSystemItem):
    def __init__(self, name: str) -> None:
        self.name = name
        self.children: list[FileSystemItem] = []

    def add_item(self, item: FileSystemItem) -> None:
        self.children.append(item)

    def get_size(self) -> int:
        return sum(child.get_size() for child in self.children)

    def print_structure(self, indent: str) -> None:
        print(f"{indent}+ {self.name}/")
        for child in self.children:
            child.print_structure(indent + "  ")

    def delete(self) -> None:
        for child in self.children:
            child.delete()
        print(f"Deleting folder: {self.name}")


if __name__ == "__main__":
    root = Folder("project")
    src  = Folder("src")
    src.add_item(File("main.py", 10))
    src.add_item(File("utils.py", 6))

    root.add_item(src)
    root.add_item(File("README.md", 2))
    root.add_item(Shortcut("docs_link", "/home/user/docs"))
    root.add_item(CompressedFile("archive.tar.gz", 500, ratio=0.4))

    root.print_structure("")
    print(f"Total size: {root.get_size()} KB")
    root.delete()
```

`Folder.get_size()` doesn't know — and doesn't need to know — that its children include three different leaf types. Adding a fourth leaf type, say `Symlink` or `EncryptedFile`, requires zero changes to `Folder` or to the client code. That's the payoff.

### Example 2: Restaurant Menu

A restaurant menu where simple items (`MenuItem`) and submenus (`SubMenu` — e.g., a "Drinks" section) share the same interface. Counting items, displaying the menu, and computing totals all work uniformly.

```python
from abc import ABC, abstractmethod


class Menu(ABC):
    @abstractmethod
    def display(self, indent: str) -> None: ...

    @abstractmethod
    def get_item_count(self) -> int: ...


class MenuItem(Menu):
    def __init__(self, name: str, price: float) -> None:
        self.name, self.price = name, price

    def display(self, indent: str) -> None:
        print(f"{indent}{self.name} - ${self.price:.2f}")

    def get_item_count(self) -> int:
        return 1


class SubMenu(Menu):
    def __init__(self, name: str) -> None:
        self.name = name
        self.children: list[Menu] = []

    def add_item(self, item: Menu) -> None:
        self.children.append(item)

    def display(self, indent: str) -> None:
        print(f"{indent}{self.name}:")
        for child in self.children:
            child.display(indent + "  ")

    def get_item_count(self) -> int:
        return sum(child.get_item_count() for child in self.children)


if __name__ == "__main__":
    drinks = SubMenu("Drinks")
    drinks.add_item(MenuItem("Cola",  1.99))
    drinks.add_item(MenuItem("Water", 0.99))

    main = SubMenu("Main Menu")
    main.add_item(MenuItem("Burger", 8.99))
    main.add_item(MenuItem("Fries",  3.99))
    main.add_item(drinks)

    main.display("")
    print(f"Total items: {main.get_item_count()}")
```

The submenu nests a submenu, and the operation just works. The recursion pattern is identical to the filesystem case — only the data and the operation names change.

### Example 3: HTML Tree

An HTML element tree — `HtmlElement` is the composite, `TextNode` is the leaf. Rendering produces nested, indented HTML by recursing through the tree.

```python
from abc import ABC, abstractmethod


class HtmlNode(ABC):
    @abstractmethod
    def render(self, indent: str) -> str: ...


class TextNode(HtmlNode):
    def __init__(self, text: str) -> None:
        self.text = text

    def render(self, indent: str) -> str:
        return indent + self.text


class HtmlElement(HtmlNode):
    def __init__(self, tag: str) -> None:
        self.tag = tag
        self.children: list[HtmlNode] = []

    def add_child(self, child: HtmlNode) -> None:
        self.children.append(child)

    def render(self, indent: str) -> str:
        result = f"{indent}<{self.tag}>\n"
        for child in self.children:
            result += child.render(indent + "  ") + "\n"
        result += f"{indent}</{self.tag}>"
        return result


if __name__ == "__main__":
    li1 = HtmlElement("li"); li1.add_child(TextNode("Item 1"))
    li2 = HtmlElement("li"); li2.add_child(TextNode("Item 2"))

    ul = HtmlElement("ul"); ul.add_child(li1); ul.add_child(li2)

    div = HtmlElement("div")
    div.add_child(TextNode("My List:"))
    div.add_child(ul)

    print(div.render(""))
```

The `<div>` doesn't care that one of its children is a `TextNode` and the other is a `<ul>` containing more `<li>` elements. They're all `HtmlNode` — that's all the renderer needs to know.

## When to Use the Composite Pattern

| Situation | Without Composite | With Composite |
|---|---|---|
| Tree with leaves and containers | `isinstance` checks in every traversal | Single polymorphic call on the root |
| Multiple operations over the tree | Repeated branching for each operation | Each node implements the operation once |
| New node types added later | Update every traversal function | Add one class, no other changes |
| Nested containers (recursive) | Manual recursion at every call site | Containers implement recursion once |

## Conclusion

The Composite pattern is the right tool when:

- **You have a part-whole hierarchy:** A tree where some nodes are atomic (leaves) and others are containers, and clients want to operate on either uniformly.
- **The operations recurse naturally:** Computing a total, rendering, deleting, counting — anything where the container's answer is built from the answers of its children.
- **The set of node types is open:** You expect to add new leaves or new composites later, and you don't want a flood of type-checking edits each time.

The pattern's strength is **uniform treatment**. A client holds a `Component` and calls a method; whether the component is a single file or an entire project tree, the call shape is identical. New node types — `Shortcut`, `CompressedFile`, `EncryptedFolder` — drop in without touching the traversal logic or the existing nodes.

A note on type safety: the classic GoF version puts `add_child` on the `Component` interface so leaves and composites share the *exact* same type. That makes traversal trivial but lets clients call `add_child` on a `File`, which has to either silently ignore it or raise. Most modern code (including the examples above) takes the alternative: child-management methods live only on the `Composite`, and clients downcast or use the composite directly when they need to mutate the tree. It's a small trade-off — slightly less uniform, but much harder to misuse.
