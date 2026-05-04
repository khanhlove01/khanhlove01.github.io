---
title: "Singleton Design Pattern: Problem and Practical Solutions"
date: 2026-04-28 00:00:00 +0700
categories: [OOP Fundamental, Design Pattern]
tags: [singleton, creational-pattern, python]
---

## The Problem

In software design, there are often situations where you must guarantee that **only one instance of a class exists** throughout the entire lifecycle of an application. 

Imagine an application with multiple threads that need to write to a log file, access a shared configuration, or pull from a limited pool of database connections. If each component simply instantiates its own `Logger` or `ConnectionPool`, you run into several issues:
1. **Inconsistent State:** Multiple instances might hold different configurations, leading to unpredictable behavior (e.g., one logger instance expects `DEBUG` while another expects `INFO`).
2. **Resource Exhaustion:** Creating multiple connection pools might open too many connections, overwhelming your database.
3. **Data Corruption / Race Conditions:** Concurrent operations on independent instances might step on each other's toes when interacting with shared external resources.

Here is a quick example showing what goes wrong without a Singleton. Two components create their own `ConfigManager`, but changing the config in one doesn't affect the other, leading to inconsistent state:

```python
class ConfigManager:
    def __init__(self) -> None:
        self.settings = {"theme": "light"}

# Component A creates a config manager and changes the theme
config_a = ConfigManager()
config_a.settings["theme"] = "dark"

# Component B creates its own config manager and reads the theme
config_b = ConfigManager()
print(config_b.settings["theme"]) # Output: "light" (Oops! B missed A's update)
```

We need a way to ensure that a class has exactly one instance, and provide a global point of access to it.

## The Solution: The Singleton Pattern

The Singleton pattern restricts the instantiation of a class to one "single" instance. This is typically done by:
1. Hiding the constructor (or overriding the object creation process).
2. Providing a static creation method that returns a cached instance.
3. Implementing thread safety, usually via **Double-Checked Locking**, to ensure that two threads don't accidentally create two instances at the exact same time.

In Python, we can achieve this by overriding the `__new__` method and using `threading.Lock`. 

> **How `__new__` works:** `__new__` is a static method that handles allocation. It runs first to create and return a raw, uninitialized instance of the class. By overriding it, you can intercept this process—such as in a Singleton pattern—to return an existing instance instead of a new one. `__init__` runs only after `__new__` returns an instance, handling its initialization.

Here are three real-world scenarios demonstrating the thread-safe Singleton pattern in Python.

### Example 1: A Thread-Safe Counter

A simple shared counter where multiple threads can increment a single, unified value. Notice how we use a double-checked lock inside `__new__` to ensure thread-safe initialization.

```python
import threading
from typing import Optional

class Counter:
    _instance: Optional['Counter'] = None
    _lock: threading.Lock = threading.Lock()
    
    def __new__(cls) -> 'Counter':
        # Double-checked locking
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._count = 0
                    cls._instance._counter_lock = threading.Lock()
        return cls._instance

    def increment(self) -> None:
        with self._counter_lock:
            self._count += 1

    def get_count(self) -> int:
        return self._count

if __name__ == "__main__":
    c1 = Counter()
    c2 = Counter()
    print(f"Same instance: {c1 is c2}") # True
    
    for _ in range(5):
        c1.increment()
    print(f"Count after 5 increments: {c1.get_count()}") # 5
```

### Example 2: Global Logger

A logger that enforces a consistent logging level across the entire application. Without a singleton, a race condition could occur where Thread A evaluates a log level just as Thread B changes it, leading to inconsistent output.

```python
from enum import IntEnum
import threading
from typing import Optional

class LogLevel(IntEnum):
    DEBUG = 0
    INFO = 1
    WARN = 2
    ERROR = 3

class Logger:
    _instance: Optional['Logger'] = None
    _lock: threading.Lock = threading.Lock()

    def __new__(cls) -> 'Logger':
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._logger_lock = threading.Lock()
                    cls._min_level = LogLevel.DEBUG
        return cls._instance

    def set_level(self, level: LogLevel) -> None:
        with self._logger_lock:
            self._min_level = level

    def _log(self, level: LogLevel, message: str) -> None:
        with self._logger_lock:
            if level >= self._min_level:
                print(f"[{level.name}] {message}")
                
    def debug(self, msg: str) -> None: self._log(LogLevel.DEBUG, msg)
    def info(self, msg: str) -> None:  self._log(LogLevel.INFO, msg)
    def warn(self, msg: str) -> None:  self._log(LogLevel.WARN, msg)
    def error(self, msg: str) -> None: self._log(LogLevel.ERROR, msg)

def get_logger() -> Logger:
    return Logger()

if __name__ == "__main__":
    l1 = Logger()
    l2 = get_logger()
    
    print(f"Same instance: {l1 is l2}") # True
    
    l1.set_level(LogLevel.WARN)
    
    l1.debug("Starting up")                     # Ignored
    l1.info("Server listening on port 8080")    # Ignored
    l1.warn("Connection pool running low")      # Printed
    l2.error("Failed to connect to database")   # Printed
```

### Example 3: Connection Pool

Managing a limited set of resources, like network or database connections. The Singleton ensures that the entire application shares the exact same limit and queue.

```python
import threading
from queue import Queue
from typing import Optional

class Connection:
    def __init__(self, conn_id: int) -> None:
        self.id: int = conn_id

    def __repr__(self) -> str:
        return f"Connection-{self.id}"

class ConnectionPool:
    _instance: Optional['ConnectionPool'] = None
    _lock: threading.Lock = threading.Lock()
    
    _initialized: bool
    _max: int
    _pool: Queue[Connection]

    def __new__(cls, max_connections: int = 5) -> 'ConnectionPool':
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._initialized = False
        return cls._instance

    def __init__(self, max_connections: int = 5) -> None:
        if getattr(self, "_initialized", False):
            return

        self._max = max_connections
        self._pool = Queue(maxsize=max_connections)

        # Pre-create connections
        for i in range(max_connections):
            self._pool.put(Connection(i + 1))
        
        self._initialized = True

    def get_connection(self, timeout: Optional[float] = None) -> Connection:
        """Borrow a connection from the pool."""
        return self._pool.get(block=True, timeout=timeout)

    def release_connection(self, conn: Connection) -> None:
        """Return a connection to the pool."""
        if not isinstance(conn, Connection):
            raise TypeError(f"expected Connection, got {type(conn).__name__}")
        self._pool.put(conn, block=True)

    def get_available_count(self) -> int:
        return self._pool.qsize()

def get_pool() -> ConnectionPool:
    return ConnectionPool()

if __name__ == "__main__":
    p1 = get_pool()
    p2 = ConnectionPool()
    print(f"Same instance: {p1 is p2}") # True
    
    c1 = p1.get_connection()
    c2 = p1.get_connection()
    
    print(f"Acquired: {c1}, {c2}")
    print(f"Available after acquiring 2: {p1.get_available_count()}") # 3
    
    p1.release_connection(c1)
    print(f"Available after release: {p1.get_available_count()}") # 4
```

## Conclusion
The Singleton pattern provides a highly effective mechanism to manage shared global state, coordinate resource constraints, and enforce thread safety when accessing shared components.
