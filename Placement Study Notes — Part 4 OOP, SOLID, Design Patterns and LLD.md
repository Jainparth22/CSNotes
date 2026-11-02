# Placement Study Notes — Part 4: OOP, SOLID, Design Patterns, UML and Low-Level Design

*Companion to Parts 1 to 3. Code is shown in Python 3 and C++17; for the larger LLD problems the main flow is in both and secondary classes in whichever is shorter (the mapping is mechanical).*

## 0. How to use these notes

Every topic has: **definition**, **why it matters**, **code**, **interview answer**, **loopholes** (the traps and wrong answers). LLD problems follow a fixed template: requirements → entities → class design → patterns used → core code → edge cases and concurrency → test cases → extensions.

**Contents:** O1 Four pillars (classes, encapsulation, abstraction, inheritance, polymorphism) · O2 Interfaces vs abstract classes · O3 Association, aggregation, composition, composition over inheritance · O4 SOLID · O5 Coupling, cohesion, dependency injection, extensibility · O6 Immutability, lifecycle, errors, resource management · O7 Thread safety in class design · O8 Unit testing and mocking · P1 to P11 Design patterns (Singleton, Factory, Builder, Strategy, Observer, Decorator, Adapter, Facade, Command, State, Repository) · U1 UML class and sequence diagrams · L0 LLD interview framework · L1 to L12 LLD problems (Parking Lot, Splitwise, LRU Cache, TTL Cache, Elevator, Library, Movie Booking, Food Delivery, Vending Machine, Logging Framework, Notification Framework, Task Scheduler).

---

# PART O: OBJECT-ORIENTED PROGRAMMING AND DESIGN PRINCIPLES

## O1. Classes, objects and the four pillars

**Class:** a blueprint combining **data (fields)** and **behaviour (methods)**. **Object:** an instance of a class with its own state. **Constructor** initializes; **destructor** (C++) cleans up; `this`/`self` is the current object.

### Encapsulation

**Definition:** bundle data with the methods that operate on it and **hide internal state**, exposing only a controlled interface, so the object can always protect its **invariants** (rules that must always be true).

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        if balance < 0: raise ValueError('negative opening balance')
        self._owner, self._balance = owner, balance          # leading underscore = private by convention
    @property
    def balance(self): return self._balance                  # read-only view
    def deposit(self, amt):
        if amt <= 0: raise ValueError('amount must be positive')
        self._balance += amt
    def withdraw(self, amt):
        if amt <= 0 or amt > self._balance: raise ValueError('invalid withdrawal')
        self._balance -= amt
```

```cpp
class BankAccount {
    std::string owner_; double balance_;                      // private by default in a class
public:
    BankAccount(std::string owner, double balance = 0) : owner_(std::move(owner)), balance_(balance) {
        if (balance < 0) throw std::invalid_argument("negative opening balance");
    }
    double balance() const { return balance_; }               // const getter
    void deposit(double amt) { if (amt <= 0) throw std::invalid_argument("bad amount"); balance_ += amt; }
    void withdraw(double amt) { if (amt <= 0 || amt > balance_) throw std::invalid_argument("invalid"); balance_ -= amt; }
};
```

**Interview answer:** "Encapsulation is hiding state behind methods so invariants cannot be broken from outside; getters and setters alone are not encapsulation if they expose the internals unconditionally."

**Loopholes:** public fields or setter-for-everything is *not* encapsulation; returning a reference/pointer to an internal mutable collection leaks the state (return a copy or a const view); Python's `_x` and `__x` are conventions/name mangling, not real access control; `friend` in C++ intentionally breaks encapsulation.

### Abstraction

**Definition:** expose **what** an object does and hide **how** it does it; depend on a simplified model/interface. Encapsulation hides data; abstraction hides complexity and implementation. Example: you call `payment.pay(100)` without knowing whether it uses UPI or a card.

### Inheritance

**Definition:** a subclass acquires the fields and methods of a base class (**is-a** relationship) and can extend or override them.

```python
class Animal:
    def __init__(self, name): self.name = name
    def speak(self): raise NotImplementedError
class Dog(Animal):
    def speak(self): return f'{self.name} says Woof'
```

```cpp
class Animal {
public:
    explicit Animal(std::string n) : name(std::move(n)) {}
    virtual ~Animal() = default;                     // virtual destructor: REQUIRED for polymorphic deletion
    virtual std::string speak() const = 0;           // pure virtual => abstract class
protected:
    std::string name;
};
class Dog : public Animal {
public:
    using Animal::Animal;
    std::string speak() const override { return name + " says Woof"; }     // override catches signature typos
};
```

Types: single, multilevel, hierarchical, multiple (C++/Python yes; Java classes no), hybrid. **Diamond problem** (D inherits B and C, both inherit A): C++ solves with `virtual` inheritance; Python with MRO (C3 linearization, see `D.__mro__`); Java forbids multiple class inheritance but allows multiple interfaces.

**Loopholes:** inheritance is **is-a**, not code reuse (use composition for reuse); the **fragile base class** problem; deep hierarchies are rigid; `protected` data is still tight coupling.

### Polymorphism

**Definition:** one interface, many implementations. **Compile-time (static):** function/operator overloading, templates. **Run-time (dynamic):** the method executed depends on the **actual object type**, via virtual dispatch.

```cpp
void describe(const Animal& a) { std::cout << a.speak(); }      // calls Dog::speak when a refers to a Dog
std::vector<std::unique_ptr<Animal>> zoo;                        // store by pointer to avoid slicing
zoo.push_back(std::make_unique<Dog>("Rex"));
for (auto& a : zoo) std::cout << a->speak() << '\n';
```

**How virtual dispatch works (C++):** each class with virtual functions has a **vtable** (array of function pointers); each object holds a hidden **vptr** to its class's vtable; a virtual call is `obj->vptr[i](...)`, one extra indirection. Python resolves every method dynamically through the object's attribute lookup (duck typing).

**Loopholes (C++ specific)**

- **Object slicing:** `Animal a = Dog(...)` copies only the base part; polymorphism is lost. Use references or pointers.
- A base class without a **virtual destructor** leaks/UB when deleting a derived object through a base pointer.
- Never call a virtual function from a **constructor or destructor** (it dispatches to the class being constructed, not the derived one).
- Overriding vs overloading vs hiding: a derived function with the same name but a different signature **hides** all base overloads; always mark `override`.
- Virtual calls cost an indirection and block inlining; overuse in hot loops matters. Default arguments are resolved statically, not dynamically.
- Python overloading does not exist (the last definition wins); use default arguments or `functools.singledispatch`.

---

## O2. Interfaces vs abstract classes

|  | Interface | Abstract class |
| --- | --- | --- |
| Purpose | a **capability/contract** (`Comparable`, `PaymentMethod`) | a **partial implementation** shared by related classes (is-a) |
| State (fields) | none (constants only) | yes |
| Constructors | no | yes |
| Implemented methods | none (Java 8+ allows `default` methods) | any mix of abstract and concrete |
| Multiple | a class can implement many | typically extends one (C++/Python can multiply inherit) |
| Use when | unrelated classes share behaviour; you want loose coupling and DI | related classes share code and state; you want a template-method skeleton |

```cpp
class IPaymentMethod {                      // C++ interface = abstract class with only pure virtual methods
public:
    virtual ~IPaymentMethod() = default;
    virtual bool pay(double amount) = 0;
};
class UpiPayment : public IPaymentMethod { public: bool pay(double a) override { /* ... */ return true; } };
```

```python
from abc import ABC, abstractmethod
class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...
class UpiPayment(PaymentMethod):
    def pay(self, amount): return True
# PaymentMethod() raises TypeError: abstract classes cannot be instantiated.
# typing.Protocol gives structural ('duck-typed') interfaces without inheritance.
```

**Template method (abstract class use):** the base class fixes the algorithm skeleton, subclasses fill in steps.

**Interview answer:** "Use an interface to define a contract unrelated classes can fulfil; use an abstract class when subclasses share state and implementation. Prefer programming to interfaces."

**Loopholes:** an abstract class can have constructors and fields, an interface cannot; "abstract class vs interface" differs by language (C++ has no `interface` keyword; Python uses ABC/Protocol); you cannot instantiate an abstract class.

---

## O3. Association, aggregation, composition and composition over inheritance

| Relationship | Meaning | Lifetime | UML | Example |
| --- | --- | --- | --- | --- |
| **Association** | objects know/use each other ("uses-a") | independent | plain line `---` / arrow | Teacher and Student; Doctor and Patient |
| **Aggregation** | whole-part, part **can exist without** the whole ("has-a", weak) | independent | hollow diamond `<>---` | Department has Professors (professors survive if the department closes); Team has Players |
| **Composition** | whole-part, part **cannot exist without** the whole, owned exclusively ("owns-a", strong) | part dies with the whole | filled diamond `<#>---` | House has Rooms; Order has OrderLines |
| **Inheritance** | is-a | n/a | hollow triangle arrow | Dog is an Animal |
| **Dependency** | temporarily uses (parameter, local variable) | transient | dashed arrow | `OrderService` uses `Logger` in one method |

```cpp
class Engine {};
class Car {                                     // COMPOSITION: Car owns its Engine (member by value or unique_ptr)
    Engine engine_;
};
class Team {                                    // AGGREGATION: Team refers to players that live elsewhere
    std::vector<Player*> players_;              // non-owning pointers (or shared_ptr/weak_ptr)
};
```

```python
class Room: ...
class House:
    def __init__(self): self.rooms = [Room(), Room()]      # composition: created and owned by House
class Team:
    def __init__(self, players): self.players = players    # aggregation: players passed in, exist independently
```

**Composition over inheritance:** build behaviour by **combining small objects** (has-a) instead of extending a class (is-a). Inheritance couples the subclass to base internals, forces the whole hierarchy on you, and explodes combinatorially (`FlyingSwimmingDuck`...). With composition you swap behaviour at run time.

```python
# Inheritance explosion: Duck -> FlyingDuck, SwimmingFlyingDuck, ... 
# Composition with Strategy:
class Duck:
    def __init__(self, fly_behavior, quack_behavior):
        self.fly, self.quack = fly_behavior, quack_behavior
    def perform_fly(self): return self.fly.fly()
class FlyWithWings:  def fly(self): return 'flapping'
class NoFly:         def fly(self): return 'cannot fly'
rubber = Duck(NoFly(), Squeak())            # behaviour assembled, changeable at run time
```

**Interview answer:** "Association is a general 'uses' link; aggregation is a weak whole-part where parts outlive the whole; composition is a strong whole-part where the parts die with the whole. I prefer composition over inheritance because it is more flexible and less coupled."

**Loopholes:** the aggregation/composition difference is about **lifetime ownership**, not syntax; inheritance is still right for a true, stable is-a relationship (and for frameworks' extension points); a `Square` is-a `Rectangle` mathematically but not behaviourally (see Liskov).

---

## O4. SOLID principles

### S: Single Responsibility Principle (SRP)

**A class should have one reason to change** (one responsibility, one actor it serves).

```python
# VIOLATION: one class computes, formats and persists
class Report:
    def calculate(self): ...
    def to_pdf(self): ...
    def save_to_db(self): ...
# FIX: separate responsibilities
class ReportCalculator: ...
class PdfRenderer: ...
class ReportRepository: ...
```

**Loophole:** SRP does not mean "one method per class"; it is about cohesion around one reason to change. Over-splitting creates needless classes.

### O: Open/Closed Principle (OCP)

**Open for extension, closed for modification:** add behaviour by adding new code (new classes), not by editing tested code.

```python
# VIOLATION: every new discount edits this function
def discount(kind, amt):
    if kind == 'student': return amt * 0.9
    elif kind == 'senior': return amt * 0.8
# FIX: strategy/polymorphism
class Discount(ABC):
    @abstractmethod
    def apply(self, amt): ...
class StudentDiscount(Discount):  apply = lambda self, a: a * 0.9
class SeniorDiscount(Discount):   apply = lambda self, a: a * 0.8
def checkout(amt, d: Discount): return d.apply(amt)      # new discount = new class, checkout untouched
```

```cpp
struct Discount { virtual ~Discount() = default; virtual double apply(double a) const = 0; };
struct StudentDiscount : Discount { double apply(double a) const override { return a * 0.9; } };
double checkout(double a, const Discount& d) { return d.apply(a); }
```

**Loophole:** a long `if/else` or `switch` on a type code is the smell; do not abstract before there is a real second variant (premature abstraction).

### L: Liskov Substitution Principle (LSP)

**Subtypes must be usable wherever the base type is expected without breaking correctness**: same or weaker preconditions, same or stronger postconditions, preserve invariants, no new exceptions.

```python
# VIOLATION: Square breaks Rectangle's contract (setting width should not change height)
class Rectangle:
    def set_width(self, w): self.w = w
    def set_height(self, h): self.h = h
    def area(self): return self.w * self.h
class Square(Rectangle):
    def set_width(self, w): self.w = self.h = w
    def set_height(self, h): self.w = self.h = h
# client: r.set_width(5); r.set_height(4); assert r.area() == 20   # fails for Square (16)
# Also: Penguin extends Bird with fly() that throws => violates LSP. FIX: separate FlyingBird / SwimmingBird abstractions,
# or model Shape with area() and make Rectangle and Square independent immutable implementations.
```

**Loophole:** LSP is about **behaviour**, not just matching signatures; overriding a method to throw `NotSupported` or do nothing is the classic sign.

### I: Interface Segregation Principle (ISP)

**Clients should not be forced to depend on methods they do not use**; prefer several small, role-specific interfaces.

```python
# VIOLATION: a Robot must implement eat()
class Worker(ABC):
    @abstractmethod
    def work(self): ...
    @abstractmethod
    def eat(self): ...
# FIX
class Workable(ABC):
    @abstractmethod
    def work(self): ...
class Eatable(ABC):
    @abstractmethod
    def eat(self): ...
class Human(Workable, Eatable): ...
class Robot(Workable): ...
```

**Loophole:** do not split interfaces so finely that nothing is cohesive; split by client role.

### D: Dependency Inversion Principle (DIP)

**High-level modules should not depend on low-level modules; both depend on abstractions.** Abstractions should not depend on details. (Dependency Injection is the technique that supplies the implementation.)

```python
# VIOLATION: OrderService is hard-wired to MySQL
class OrderService:
    def __init__(self): self.db = MySQLDatabase()
# FIX: depend on an abstraction, inject the implementation
class OrderRepository(ABC):
    @abstractmethod
    def save(self, order): ...
class OrderService:
    def __init__(self, repo: OrderRepository): self.repo = repo      # constructor injection
    def place(self, order): self.repo.save(order)
svc = OrderService(MySQLOrderRepository())      # production
svc = OrderService(InMemoryOrderRepository())   # tests: no database needed
```

```cpp
class OrderRepository { public: virtual ~OrderRepository() = default; virtual void save(const Order&) = 0; };
class OrderService {
    OrderRepository& repo_;
public:
    explicit OrderService(OrderRepository& r) : repo_(r) {}      // injected dependency
    void place(const Order& o) { repo_.save(o); }
};
```

**Loophole:** DIP is not the same as DI (principle vs technique) nor as IoC containers; the *interface is owned by the high-level module*; creating an interface for every class is over-engineering.

**One-line memory aid:** S = one job; O = extend, don't edit; L = subtypes keep promises; I = small interfaces; D = depend on abstractions.

---

## O5. Coupling, cohesion, dependency injection, extensibility

**Coupling:** how much one module knows about/depends on another (want **low**). **Cohesion:** how strongly the responsibilities inside one module belong together (want **high**). Types of coupling from worst to best: content (reaching into internals) → common (shared globals) → control (passing flags that steer behaviour) → stamp (passing whole structures) → data (passing only needed values) → message/interface coupling.

**Dependency injection (DI):** an object receives its dependencies from outside instead of creating them.

| Style | Example | Note |
| --- | --- | --- |
| Constructor injection | `OrderService(repo)` | preferred: dependencies explicit, object always valid |
| Setter injection | `svc.set_logger(l)` | optional dependencies; object can be half-configured |
| Interface/method injection | `process(order, clock)` | per-call collaborators |

**Benefits:** swappable implementations, easy testing with fakes, fewer hidden dependencies. A **composition root** (usually `main`) wires everything; DI containers (Spring, Dagger) automate it. **Service locator** and global singletons hide dependencies (avoid).

**Extensibility and maintainability:** program to interfaces; avoid `if type == ...` chains; keep functions small; name things clearly; avoid global state; use configuration instead of hard-coded constants; prefer composition; write tests; follow YAGNI (do not build what you do not need) and DRY (but do not merge things that merely look alike); **Law of Demeter** (talk to friends, not strangers: avoid `a.getB().getC().doX()`).

---

## O6. Immutability, object lifecycle, error handling and resource management

**Immutable objects** never change after construction (`final`, `const`, frozen dataclass, tuple, `str`). Benefits: thread-safe without locks, safe to share/cache/use as map keys, easier reasoning. Cost: modifications create new objects (use builders or copy-with).

```python
from dataclasses import dataclass
@dataclass(frozen=True)
class Money:
    amount: int          # store money as integer minor units (paise/cents), never float
    currency: str
    def add(self, other: 'Money') -> 'Money':
        if self.currency != other.currency: raise ValueError('currency mismatch')
        return Money(self.amount + other.amount, self.currency)
```

**Object lifecycle:** allocation → construction (establish invariants; fail fast with exceptions) → use → destruction. C++: constructors, copy/move, destructors (rule of 0/3/5, RAII; Part 2 D1/D2). Python: `__init__`, context managers (`with`), `__del__` is unreliable (rely on `with`/`close()`). Java: GC plus try-with-resources. Two-phase init (`init()` after constructor) leaves half-built objects; avoid.

**Error handling principles**

- **Validate early** (constructors, public methods); **fail fast** with clear messages.
- Use exceptions for exceptional conditions; use return values/optional for expected outcomes (not found).
- Create **domain exceptions** (`InsufficientFundsError`, `SlotNotAvailableError`) deriving from a base app exception.
- Never swallow exceptions silently (`except: pass`); catch specific types; preserve the cause (`raise X from e`); log once at the boundary.
- **Exception safety (C++):** *basic* (no leaks, invariants hold), *strong* (all or nothing; copy-and-swap), *no-throw* (destructors, swap, moves should not throw). Destructors must not throw.
- **Resource management:** C++ RAII (`unique_ptr`, `lock_guard`, file wrappers); Python `with`; Java try-with-resources. Always release locks, files, sockets, DB connections, even on error.

```python
class InsufficientFundsError(Exception): ...
def withdraw(acct, amt):
    if amt > acct.balance: raise InsufficientFundsError(f'need {amt}, have {acct.balance}')
with open('log.txt', 'a') as f:      # closed automatically even if an exception occurs
    f.write('ok')
```

---

## O7. Thread safety and concurrency in class design

- **Prefer immutability and confinement** (each thread owns its data); share only what you must.
- **Encapsulate the lock inside the class:** callers should not need to know how to synchronize. Guard **all** accesses to a shared field with the same lock.
- **Make compound operations atomic:** `check-then-act` and `read-modify-write` must be one critical section (see TOCTOU, Part 2 A9).
- **Do not leak internal mutable state** (return copies/snapshots); do not call unknown/external code (callbacks) while holding a lock; keep critical sections small; acquire multiple locks in a **global order** or with `std::scoped_lock`.
- Use higher-level tools: concurrent queues, atomics, `ConcurrentHashMap`, thread pools, `std::shared_mutex` for read-mostly data; document which methods are thread-safe.

```python
import threading
class Counter:
    def __init__(self): self._n, self._lock = 0, threading.Lock()
    def incr(self):
        with self._lock: self._n += 1
    def value(self):
        with self._lock: return self._n
```

```cpp
class Counter {
    mutable std::mutex m_; long n_ = 0;
public:
    void incr() { std::lock_guard<std::mutex> g(m_); ++n_; }
    long value() const { std::lock_guard<std::mutex> g(m_); return n_; }
};
```

**Loopholes:** a thread-safe collection does not make a sequence of calls on it atomic; double-checked locking without proper atomics is broken; locks around only the *writes* are not enough; deadlocks from lock-order inversion; holding a lock while doing I/O.

---

## O8. Unit testing, mocking and testable architecture

**Testable design:** inject dependencies (clock, repository, payment gateway, random generator), keep business logic in pure functions/classes with no I/O, avoid static/global state and hidden `new`s, program to interfaces. Hard-to-test code usually has design problems.

**Test doubles**

| Double | Purpose |
| --- | --- |
| **Dummy** | fills a parameter, never used |
| **Stub** | returns canned answers |
| **Fake** | working lightweight implementation (in-memory repository) |
| **Mock** | records calls so the test can verify **interactions** (called once with X) |
| **Spy** | real object that also records calls |

**Structure (Arrange, Act, Assert); one behaviour per test; deterministic (inject a fake clock instead of `time.now()`); fast; independent.**

```python
import pytest
from unittest.mock import Mock

def test_order_saved_and_customer_notified():
    repo, notifier = Mock(), Mock()                       # Arrange: mocks injected
    svc = OrderService(repo, notifier)
    svc.place(Order(id=1, amount=500))                    # Act
    repo.save.assert_called_once()                        # Assert interaction
    notifier.send.assert_called_with('Order 1 confirmed')

def test_withdraw_more_than_balance_raises():
    acct = BankAccount('A', 100)
    with pytest.raises(ValueError):
        acct.withdraw(500)
```

C++: GoogleTest (`TEST`, `EXPECT_EQ`, `EXPECT_THROW`) with GoogleMock (`MOCK_METHOD`, `EXPECT_CALL`) over an interface.

**What to test:** happy path, boundaries (0, 1, max), invalid input, empty/null, error paths, concurrency where relevant, idempotency. **Pyramid:** many unit tests, fewer integration tests, few end-to-end tests.

**Loopholes:** over-mocking couples tests to implementation (they break on refactoring); test behaviour, not private methods; coverage percentage is not quality; flaky tests (time, randomness, shared state) must be fixed; mocking the class under test proves nothing.

---

# PART P: DESIGN PATTERNS

**What a pattern is:** a named, proven solution to a recurring design problem, with known trade-offs. Patterns are a **shared vocabulary**, not a checklist: use one only when the problem actually appears.

| Category | Problem it solves | Patterns in your list |
| --- | --- | --- |
| **Creational** | how objects are created | Singleton, Factory, Builder |
| **Structural** | how objects are composed | Decorator, Adapter, Facade, (Repository is an architectural pattern) |
| **Behavioral** | how objects communicate and share responsibility | Strategy, Observer, Command, State |

**Quick chooser**

| You need to... | Use |
| --- | --- |
| exactly one shared instance | Singleton (but consider DI first) |
| create objects without hard-coding the class | Factory |
| construct a complex object step by step, many optional fields | Builder |
| swap an algorithm/policy at run time | Strategy |
| notify many objects when something changes | Observer |
| add responsibilities dynamically, stackable | Decorator |
| make an incompatible interface fit what the client expects | Adapter |
| hide a messy subsystem behind one simple API | Facade |
| turn requests into objects: queue, log, undo/redo | Command |
| behaviour depends on the object's current state, avoid giant if/else | State |
| isolate data access from business logic | Repository |

---

## P1. Singleton

**Intent:** exactly **one instance** of a class with a global access point (configuration, logger, connection pool, cache).

```python
import threading
class Config:
    _instance, _lock = None, threading.Lock()
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:                       # double-checked locking (safe in Python)
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance.settings = {}
        return cls._instance
assert Config() is Config()
# In Python a module-level object is already a singleton: simplest and idiomatic.
```

```cpp
class Config {
    Config() = default;                                   // private constructor
public:
    static Config& instance() { static Config c; return c; }   // Meyers singleton: thread-safe since C++11, lazy
    Config(const Config&) = delete; Config& operator=(const Config&) = delete;
    std::unordered_map<std::string, std::string> settings;
};
```

**Loopholes:** it is **global state** (hidden dependencies, hard to test, hard to reset between tests); naive lazy init is **not thread-safe**; double-checked locking is broken without proper memory ordering in some languages; serialization/reflection/cloning can create a second instance; inheritance is awkward; in multi-process or multi-server systems "one instance" is only per process. **Better:** create it once in `main` and **inject** it (a "singleton" lifetime without global access).

---

## P2. Factory (Simple Factory, Factory Method, Abstract Factory)

**Intent:** centralize object creation so clients depend on an **interface**, not on concrete classes.

- **Simple factory:** one function/class with an `if/switch` returning the right type (not a GoF pattern, but common).
- **Factory Method:** a base class declares `create()`, subclasses decide the concrete product.
- **Abstract Factory:** an interface that creates **families** of related objects (`WindowsUIFactory` makes Windows button + checkbox; `MacUIFactory` makes Mac ones).

```python
class Notification(ABC):
    @abstractmethod
    def send(self, to, msg): ...
class Email(Notification):
    def send(self, to, msg): print(f'email to {to}: {msg}')
class Sms(Notification):
    def send(self, to, msg): print(f'sms to {to}: {msg}')

class NotificationFactory:
    _registry = {'email': Email, 'sms': Sms}               # registry = open for extension (OCP)
    @classmethod
    def create(cls, kind) -> Notification:
        try: return cls._registry[kind]()
        except KeyError: raise ValueError(f'unknown channel {kind}')
    @classmethod
    def register(cls, kind, klass): cls._registry[kind] = klass
NotificationFactory.create('sms').send('+91...', 'OTP 123456')
```

```cpp
struct Notification { virtual ~Notification() = default; virtual void send(const std::string& to, const std::string& msg) = 0; };
struct Email : Notification { void send(const std::string& to, const std::string& m) override { std::cout << "email to " << to << ": " << m << "\n"; } };
struct Sms   : Notification { void send(const std::string& to, const std::string& m) override { std::cout << "sms to " << to << ": " << m << "\n"; } };
std::unique_ptr<Notification> make(const std::string& kind) {
    if (kind == "email") return std::make_unique<Email>();
    if (kind == "sms")   return std::make_unique<Sms>();
    throw std::invalid_argument("unknown channel");
}
```

**Loopholes:** a factory with a growing `if/else` chain violates OCP (use a registry/map of creators); returning a raw owning pointer in C++ (return `unique_ptr`); factory vs builder: factory chooses **which** class, builder assembles **one complex** object; do not use a factory for a class that is never going to have alternatives.

---

## P3. Builder

**Intent:** construct a complex object **step by step**, separating construction from representation; avoids **telescoping constructors** (many optional parameters) and half-built objects.

```python
from dataclasses import dataclass, field
@dataclass(frozen=True)
class HttpRequest:
    url: str; method: str; headers: dict; body: str | None; timeout: int

class HttpRequestBuilder:
    def __init__(self, url): self._url, self._method, self._headers, self._body, self._timeout = url, 'GET', {}, None, 30
    def method(self, m): self._method = m; return self           # fluent: return self
    def header(self, k, v): self._headers[k] = v; return self
    def body(self, b): self._body = b; return self
    def timeout(self, t): self._timeout = t; return self
    def build(self):
        if self._method in ('POST', 'PUT') and self._body is None: raise ValueError('body required')   # validate here
        return HttpRequest(self._url, self._method, dict(self._headers), self._body, self._timeout)
req = HttpRequestBuilder('https://api.x.com').method('POST').header('A', 'b').body('{}').timeout(5).build()
```

```cpp
class HttpRequest {
    friend class HttpRequestBuilder;
    std::string url_, method_ = "GET", body_; int timeout_ = 30;
    HttpRequest() = default;
public:
    const std::string& url() const { return url_; }
};
class HttpRequestBuilder {
    HttpRequest r_;
public:
    explicit HttpRequestBuilder(std::string url) { r_.url_ = std::move(url); }
    HttpRequestBuilder& method(std::string m) { r_.method_ = std::move(m); return *this; }
    HttpRequestBuilder& body(std::string b) { r_.body_ = std::move(b); return *this; }
    HttpRequestBuilder& timeout(int t) { r_.timeout_ = t; return *this; }
    HttpRequest build() const {
        if ((r_.method_ == "POST" || r_.method_ == "PUT") && r_.body_.empty()) throw std::invalid_argument("body required");
        return r_;
    }
};
```

**Loopholes:** the product should be **immutable** after `build()`; validate required fields in `build()`, not in setters; a builder is mutable and **not thread-safe**; Python/C++ named/designated arguments (`dataclass`, C++20 designated initializers) often make a builder unnecessary for simple cases; a **Director** (predefined recipes) is optional.

---

## P4. Strategy

**Intent:** define a family of interchangeable algorithms, encapsulate each, and let the client pick one **at run time**. It is the OCP-friendly replacement for `if/else` on a type.

```
Context --has-a--> <<interface>> Strategy --- ConcreteStrategyA / ConcreteStrategyB
```

```python
class PricingStrategy(ABC):
    @abstractmethod
    def price(self, base): ...
class Regular(PricingStrategy):  price = lambda self, b: b
class Festive(PricingStrategy):  price = lambda self, b: b * 0.8
class Cart:
    def __init__(self, strategy: PricingStrategy): self.strategy, self.items = strategy, []
    def total(self): return sum(self.strategy.price(i) for i in self.items)
cart = Cart(Festive()); cart.items = [100, 200]; cart.strategy = Regular()      # swap at run time
# In Python a strategy can be just a function: Cart(strategy=lambda b: b * 0.8)
```

```cpp
class Cart {
    std::function<double(double)> pricing_;               // strategy as a callable: no class hierarchy needed
    std::vector<double> items_;
public:
    explicit Cart(std::function<double(double)> p) : pricing_(std::move(p)) {}
    void add(double x) { items_.push_back(x); }
    void setStrategy(std::function<double(double)> p) { pricing_ = std::move(p); }
    double total() const { double t = 0; for (double x : items_) t += pricing_(x); return t; }
};
Cart c([](double b) { return b * 0.8; });
```

**Real uses:** payment methods, sorting comparators, compression, shipping cost, routing algorithms, retry/backoff policies, parking fee calculation.

**Loopholes:** it adds objects/indirection for algorithms that never change; the client must know the strategies to pick one (often combined with a factory); strategy (**how** to do something, chosen by the client) vs state (**what mode** the object is in, changes itself); strategies should be stateless or clearly scoped.

---

## P5. Observer (publish-subscribe)

**Intent:** one-to-many dependency: when the **subject** changes, all registered **observers** are notified automatically; the subject does not know their concrete types.

```python
class EventBus:
    def __init__(self): self._subs = {}
    def subscribe(self, event, handler): self._subs.setdefault(event, []).append(handler)
    def unsubscribe(self, event, handler): self._subs.get(event, []).remove(handler)
    def publish(self, event, data):
        for h in list(self._subs.get(event, [])):          # iterate over a COPY: handlers may unsubscribe
            try: h(data)
            except Exception as e: print('observer failed:', e)   # one bad observer must not break the rest
bus = EventBus()
bus.subscribe('order_placed', lambda o: print('email for', o))
bus.subscribe('order_placed', lambda o: print('update inventory for', o))
bus.publish('order_placed', 'order#1')
```

```cpp
class Observer { public: virtual ~Observer() = default; virtual void update(const std::string& event) = 0; };
class Subject {
    std::vector<Observer*> obs_; std::mutex m_;
public:
    void attach(Observer* o) { std::lock_guard<std::mutex> g(m_); obs_.push_back(o); }
    void detach(Observer* o) { std::lock_guard<std::mutex> g(m_); obs_.erase(std::remove(obs_.begin(), obs_.end(), o), obs_.end()); }
    void notify(const std::string& e) {
        std::vector<Observer*> snapshot; { std::lock_guard<std::mutex> g(m_); snapshot = obs_; }   // do not call out while locked
        for (auto* o : snapshot) o->update(e);
    }
};
```

**Real uses:** UI events, stock price tickers, notification systems, MVC (model notifies views), message brokers (pub/sub at system scale).

**Loopholes:** the **lapsed listener** memory leak (observers never unsubscribed keep the subject holding them; use weak references or explicit `detach`); notification **order** is not guaranteed; an exception in one observer must not stop the others; modifying the observer list during notification; **update storms/cascades** and infinite loops (observer changes the subject); synchronous notification blocks the publisher (use a queue for async); push vs pull model (send data vs let observers query).

---

## P6. Decorator

**Intent:** attach extra responsibilities to an object **dynamically** by **wrapping** it in objects that implement the **same interface**; a flexible alternative to subclassing.

```python
class Coffee(ABC):
    @abstractmethod
    def cost(self): ...
    @abstractmethod
    def description(self): ...
class Espresso(Coffee):
    cost = lambda self: 50
    description = lambda self: 'Espresso'
class AddOn(Coffee):                                   # decorator base: HAS-A Coffee and IS-A Coffee
    def __init__(self, inner: Coffee): self.inner = inner
class Milk(AddOn):
    def cost(self): return self.inner.cost() + 10
    def description(self): return self.inner.description() + ' + Milk'
class Sugar(AddOn):
    def cost(self): return self.inner.cost() + 2
    def description(self): return self.inner.description() + ' + Sugar'
order = Sugar(Milk(Espresso()))                        # stack in any order and count
print(order.description(), order.cost())               # Espresso + Milk + Sugar 62
```

```cpp
struct Coffee { virtual ~Coffee() = default; virtual int cost() const = 0; virtual std::string description() const = 0; };
struct Espresso : Coffee { int cost() const override { return 50; } std::string description() const override { return "Espresso"; } };
class Milk : public Coffee {
    std::unique_ptr<Coffee> inner_;
public:
    explicit Milk(std::unique_ptr<Coffee> c) : inner_(std::move(c)) {}
    int cost() const override { return inner_->cost() + 10; }
    std::string description() const override { return inner_->description() + " + Milk"; }
};
auto order = std::make_unique<Milk>(std::make_unique<Espresso>());
```

**Python's `@decorator` syntax** is the function-level cousin (wrap a function to add logging, caching, auth, retry). **Real uses:** Java I/O streams (`BufferedReader(new FileReader(...))`), middleware chains, logging/caching/retry wrappers around a service, encryption/compression layers.

**Loopholes:** many small wrapper objects are hard to debug; order matters (`Compress(Encrypt(x))` is not `Encrypt(Compress(x))`); identity checks fail because the wrapper is not the original; do not use when you need to remove a middle decorator; decorator keeps the **same** interface (adapter changes it).

---

## P7. Adapter

**Intent:** convert the interface of an existing class into the one the client **expects**, so incompatible classes work together (legacy code, third-party SDKs).

```python
class PaymentProcessor(ABC):                           # the interface our app expects
    @abstractmethod
    def pay(self, amount_rupees: float) -> bool: ...
class LegacyGateway:                                   # third-party: different names, units
    def make_payment(self, paise: int, currency: str) -> dict: return {'status': 'OK'}
class LegacyGatewayAdapter(PaymentProcessor):
    def __init__(self, gw: LegacyGateway): self.gw = gw   # object adapter (composition)
    def pay(self, amount_rupees):
        res = self.gw.make_payment(int(round(amount_rupees * 100)), 'INR')
        return res['status'] == 'OK'
```

```cpp
class LegacyGateway { public: bool makePayment(int paise, const std::string& cur) { return true; } };
class PaymentProcessor { public: virtual ~PaymentProcessor() = default; virtual bool pay(double rupees) = 0; };
class LegacyAdapter : public PaymentProcessor {
    LegacyGateway& gw_;
public:
    explicit LegacyAdapter(LegacyGateway& g) : gw_(g) {}
    bool pay(double rupees) override { return gw_.makePayment(static_cast<int>(std::llround(rupees * 100)), "INR"); }
};
```

**Object adapter** (composition, shown) vs **class adapter** (multiple inheritance). **Real uses:** wrapping a vendor SDK so you can swap vendors, adapting data formats, `std::stack` over `deque`.

**Loopholes:** adapter vs decorator (changes the interface vs keeps it) vs facade (simplifies a whole subsystem vs adapts one class); unit conversion, error mapping and null handling belong inside the adapter; do not leak vendor types past the adapter boundary.

---

## P8. Facade

**Intent:** provide **one simple interface** to a complex subsystem of many classes; clients call the facade instead of orchestrating the details.

```python
class InventoryService:  reserve = lambda self, sku, qty: True
class PaymentService:    charge = lambda self, user, amt: 'txn123'
class ShippingService:   schedule = lambda self, order: 'AWB1'
class NotificationService: send = lambda self, user, msg: None

class OrderFacade:
    def __init__(self, inv, pay, ship, notify): self.inv, self.pay, self.ship, self.notify = inv, pay, ship, notify
    def place_order(self, user, sku, qty, amount):
        if not self.inv.reserve(sku, qty): raise RuntimeError('out of stock')
        txn = self.pay.charge(user, amount)
        awb = self.ship.schedule((user, sku, qty))
        self.notify.send(user, f'Order placed, tracking {awb}')
        return txn
```

```cpp
class OrderFacade {
    InventoryService& inv_; PaymentService& pay_; ShippingService& ship_;
public:
    OrderFacade(InventoryService& i, PaymentService& p, ShippingService& s) : inv_(i), pay_(p), ship_(s) {}
    std::string placeOrder(const std::string& user, const std::string& sku, int qty, double amt) {
        if (!inv_.reserve(sku, qty)) throw std::runtime_error("out of stock");
        auto txn = pay_.charge(user, amt);
        ship_.schedule(user, sku, qty);
        return txn;
    }
};
```

**Real uses:** an SDK's high-level client, a service layer, `requests.get()` hiding sockets and TLS, a compiler front end.

**Loopholes:** a facade can become a **god class** if it accumulates logic (keep it thin: orchestration only); it should not prevent advanced users from using the subsystem directly; it needs compensation/rollback when a step fails midway (the example above must release inventory if payment fails: that is a saga/transaction concern).

---

## P9. Command

**Intent:** encapsulate a request as an **object**, so you can parameterize, **queue, log, schedule and undo** operations.

```python
class Command(ABC):
    @abstractmethod
    def execute(self): ...
    @abstractmethod
    def undo(self): ...
class Editor:
    def __init__(self): self.text = ''
class InsertText(Command):
    def __init__(self, editor, s): self.ed, self.s = editor, s
    def execute(self): self.ed.text += self.s
    def undo(self): self.ed.text = self.ed.text[:-len(self.s)]
class History:
    def __init__(self): self.done, self.undone = [], []
    def run(self, cmd): cmd.execute(); self.done.append(cmd); self.undone.clear()    # new action clears redo
    def undo(self):
        if self.done: c = self.done.pop(); c.undo(); self.undone.append(c)
    def redo(self):
        if self.undone: c = self.undone.pop(); c.execute(); self.done.append(c)
ed, h = Editor(), History()
h.run(InsertText(ed, 'Hello')); h.run(InsertText(ed, ' World')); h.undo(); print(ed.text)   # Hello
```

```cpp
struct Command { virtual ~Command() = default; virtual void execute() = 0; virtual void undo() = 0; };
class InsertText : public Command {
    std::string& text_; std::string s_;
public:
    InsertText(std::string& t, std::string s) : text_(t), s_(std::move(s)) {}
    void execute() override { text_ += s_; }
    void undo() override { text_.erase(text_.size() - s_.size()); }
};
class History {
    std::vector<std::unique_ptr<Command>> done_;
public:
    void run(std::unique_ptr<Command> c) { c->execute(); done_.push_back(std::move(c)); }
    void undo() { if (!done_.empty()) { done_.back()->undo(); done_.pop_back(); } }
};
```

**Real uses:** undo/redo, macro recording, task/job queues, transaction logs, remote control buttons, scheduling (see the Task Scheduler problem), GUI actions, event sourcing.

**Loopholes:** every command needs enough state to **undo** (store old values); undo of non-reversible actions (sending an email) needs compensating actions; commands holding references to destroyed receivers dangle; memory growth of unbounded history (cap it); a new action after undo must clear the redo stack.

---

## P10. State

**Intent:** let an object **change its behaviour when its internal state changes**; each state is a class, and the context delegates to the current state. It replaces large `if/else` or `switch` blocks on a status field.

```
Order: NEW --pay--> PAID --ship--> SHIPPED --deliver--> DELIVERED      (invalid transitions are rejected by the state)
```

```python
class OrderState(ABC):
    def pay(self, o): raise InvalidTransition('pay', type(self).__name__)
    def ship(self, o): raise InvalidTransition('ship', type(self).__name__)
class InvalidTransition(Exception):
    def __init__(self, action, state): super().__init__(f'cannot {action} in {state}')
class New(OrderState):
    def pay(self, o): o.state = Paid()
class Paid(OrderState):
    def ship(self, o): o.state = Shipped()
class Shipped(OrderState): ...
class Order:
    def __init__(self): self.state = New()
    def pay(self): self.state.pay(self)
    def ship(self): self.state.ship(self)
o = Order(); o.pay(); o.ship()          # o.pay() again would raise InvalidTransition
```

```cpp
class Order;
struct OrderState {
    virtual ~OrderState() = default;
    virtual void pay(Order&) { throw std::logic_error("cannot pay in this state"); }
    virtual void ship(Order&) { throw std::logic_error("cannot ship in this state"); }
};
class Order {
    std::unique_ptr<OrderState> state_;
public:
    Order();
    void setState(std::unique_ptr<OrderState> s) { state_ = std::move(s); }
    void pay() { state_->pay(*this); }
    void ship() { state_->ship(*this); }
};
struct Paid : OrderState { void ship(Order&) override { /* setState(Shipped) */ } };
struct New  : OrderState { void pay(Order& o) override { o.setState(std::make_unique<Paid>()); } };
Order::Order() : state_(std::make_unique<New>()) {}
```

**Real uses:** vending machine, ATM, traffic light, order/ticket lifecycles, media player, TCP connection state, document workflow.

**Loopholes:** state vs strategy (state transitions are driven by the object itself; strategy is chosen from outside); for tiny machines an enum + table of allowed transitions is simpler; a state explosion needs a state table or hierarchical states; stateless state objects can be shared (flyweight); persist the state name, not the object, when saving.

---

## P11. Repository

**Intent:** an **abstraction over data storage** that gives the domain a collection-like interface (`add`, `get`, `find`, `remove`), hiding SQL/ORM/HTTP details. (Architectural/DDD pattern; close to DAO.)

```python
class UserRepository(ABC):
    @abstractmethod
    def add(self, user): ...
    @abstractmethod
    def get(self, user_id): ...
    @abstractmethod
    def find_by_email(self, email): ...
class InMemoryUserRepository(UserRepository):          # fake for tests and prototypes
    def __init__(self): self._d = {}
    def add(self, user): self._d[user.id] = user
    def get(self, user_id): return self._d.get(user_id)
    def find_by_email(self, email): return next((u for u in self._d.values() if u.email == email), None)
class SqlUserRepository(UserRepository):               # production implementation
    def __init__(self, conn): self.conn = conn
    def get(self, user_id):
        row = self.conn.execute('SELECT id, email FROM users WHERE id = ?', (user_id,)).fetchone()   # parameterised
        return User(*row) if row else None
    # add/find_by_email omitted for brevity
class SignupService:
    def __init__(self, repo: UserRepository): self.repo = repo      # depends on the abstraction (DIP)
```

```cpp
class UserRepository {
public:
    virtual ~UserRepository() = default;
    virtual void add(const User& u) = 0;
    virtual std::optional<User> get(int id) const = 0;          // optional instead of nullptr for "not found"
    virtual std::optional<User> findByEmail(const std::string& email) const = 0;
};
class InMemoryUserRepository : public UserRepository {
    std::unordered_map<int, User> d_;
public:
    void add(const User& u) override { d_[u.id] = u; }
    std::optional<User> get(int id) const override { auto it = d_.find(id); return it == d_.end() ? std::nullopt : std::optional<User>(it->second); }
    std::optional<User> findByEmail(const std::string& e) const override { for (auto& [id, u] : d_) if (u.email == e) return u; return std::nullopt; }
};
```

**Related:** **Unit of Work** (track changes and commit them as one transaction), **Specification** (composable query criteria), **DAO** (table-oriented) vs Repository (aggregate/domain-oriented), ORMs (SQLAlchemy, Hibernate) often already are repositories.

**Loopholes:** a **leaky abstraction** (exposing SQL, ORM entities or `Query` objects through the interface); returning `None` vs raising vs `optional` must be a consistent contract; a generic repository with `find_by_any_query` defeats the purpose; transactions and concurrency (optimistic version checks) must be considered at the repository/unit-of-work level; N+1 queries hidden behind a tidy interface; do not add a repository when plain ORM use is enough for a CRUD app.

**Pattern summary for interviews:** name the **problem**, the **pattern**, the **participants**, one **real example**, and one **trade-off**. If asked "where did you use it?", prefer a real example from your projects (middleware = decorator/chain, payment SDK wrapper = adapter, event listeners = observer).

---

# PART U: UML AND DESIGN PRACTICE

## U1. UML class diagrams and sequence diagrams

**Class diagram:** a box with three compartments: **name**, **attributes**, **methods**. Visibility: `+` public, `-` private, `#` protected, `~` package. Static members are underlined; abstract classes/methods are in *italics* (or `<<abstract>>`); interfaces are `<<interface>>`.

```
+--------------------------+
|      ParkingLot          |
+--------------------------+
| - floors: List<Floor>    |
| - feeStrategy: FeeStrategy|
+--------------------------+
| + park(v: Vehicle): Ticket|
| + exit(id: String): Money |
+--------------------------+
```

**Relationships (notation and meaning; compare O3)**

```
Inheritance (is-a)     Child ----|> Parent           Realization (implements)  Class - - -|> <<interface>>
Association (uses)     A -------> B                  Dependency (temporary)    A - - - -> B
Aggregation (has-a)    Whole <>------ Part           Composition (owns-a)      Whole <#>------ Part
Multiplicity           1    0..1    *    1..*        e.g.  ParkingLot 1 <#>---- 1..* Floor <#>---- * Slot
```

**Example class diagram: Parking Lot**

```
           <<interface>> FeeStrategy <|- - - HourlyFee
                         ^ uses
  ParkingLot 1 <#>---- 1..* Floor 1 <#>---- * Slot
      |                                       | 0..1
      | issues                                 v
      +----------> Ticket * ----------------> Vehicle
                     (id, entryTime)          (plate, type)
```

**Sequence diagram:** shows **objects over time**. Lifelines run downward, **solid arrows** are synchronous calls, **dashed arrows** are returns, activation bars show when an object is working. Fragments: `alt` (if/else), `opt`, `loop`, `par`.

```
Driver        Gate          ParkingLot       Slot         Ticket
  |--enter()-->|              |              |             |
  |            |--park(v)---->|              |             |
  |            |              |--findFree(type)-->|         |
  |            |              |<--slot ----------|         |
  |            |              |--occupy(v)------>|         |
  |            |              |--create-------------------->|
  |            |<--ticket-----|              |             |
  |<--ticket---|              |              |             |
  alt [lot full]: ParkingLot --throws LotFullError--> Gate --display 'FULL'--> Driver
```

**Interview tips:** draw only the classes that matter (6 to 10), show relationships with multiplicity, include interfaces/strategies, and walk through **one use case** with a sequence diagram. Boxes first, details second; say the responsibility of each class in one sentence. Tools: draw.io, Mermaid, PlantUML (text to diagram).

**Loopholes:** mixing up aggregation and composition arrows; putting every getter in the box; forgetting multiplicity; drawing inheritance where composition fits; diagrams that do not match the code you then write.

---

# PART L: LOW-LEVEL DESIGN (LLD) PROBLEMS

## L0. LLD interview framework (use for every problem)

**Time plan (about 45 minutes):** requirements 5 · entities and class diagram 10 · core code 20 · edge cases, concurrency and extension 7 · tests 3.

1. **Clarify requirements.** Actors, core use cases, what is **out of scope**, scale (single machine, one JVM/process?), **concurrency** (multiple users at once?), persistence, extensibility asks. Write functional vs non-functional lists.
2. **Identify entities:** nouns become classes, verbs become methods, adjectives/types become enums or strategies.
3. **Relationships:** has-a, uses, is-a; interfaces for things that vary (fees, allocation, notification channels).
4. **Apply patterns only where they pay off:** Strategy for pluggable policies, Factory for creation by type, State for lifecycles, Observer for notifications, Singleton sparingly (inject instead), Command for requests/undo.
5. **APIs first:** write the public method signatures the client would call; keep them small and typed.
6. **Code the core flow end to end**, then secondary pieces.
7. **Edge cases:** invalid input, empty/full, duplicates, double operations, boundaries, time, failure midway (rollback), idempotency.
8. **Concurrency:** what is shared, what is the critical section, lock granularity, deadlock avoidance, thread-safe collections.
9. **Extensibility:** say how you would add a new type/rule **without modifying** existing classes (OCP).
10. **Tests:** list 5 to 8 cases (happy path, boundary, error, concurrency).

**Habits that score:** money as integer minor units; inject the clock and ID generator; enums over magic strings; custom exceptions; no god classes; `final`/immutable value objects; small methods; talk through trade-offs aloud.

---

## L1. Parking Lot

**Requirements:** multiple floors; slots for motorcycle, car, truck; entry issues a **ticket**, exit computes the **fee** and frees the slot; hourly rates by vehicle type; find the **nearest** free slot; reject when full; no two vehicles in one slot even with concurrent gates. *Out of scope:* payments gateway, reservations (mention as extensions).

**Entities and patterns:** `ParkingLot` (orchestrator), `Floor`, `Slot` (id, floor, type), `Vehicle` (plate, type), `Ticket` (id, vehicle, slot, entry time), `FeeStrategy` (**Strategy**: hourly, flat, weekend pricing), slot-choice policy (**Strategy**), `Clock` (injected), custom exceptions. A **Factory** can create vehicles by type; an **Observer** can drive a display board.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
import heapq, itertools, math, threading, time

class VehicleType(Enum): MOTORCYCLE = 1; CAR = 2; TRUCK = 3
class LotFullError(Exception): ...
class InvalidTicketError(Exception): ...
class AlreadyParkedError(Exception): ...

@dataclass(frozen=True)
class Vehicle: plate: str; type: VehicleType
@dataclass(frozen=True)
class Slot: id: str; floor: int; type: VehicleType
@dataclass
class Ticket: id: str; vehicle: Vehicle; slot: Slot; entry_time: float

class FeeStrategy(ABC):
    @abstractmethod
    def fee(self, vtype: VehicleType, seconds: float) -> int: ...      # integer currency units
class HourlyFee(FeeStrategy):
    def __init__(self, rates): self.rates = rates                       # {VehicleType.CAR: 40, ...}
    def fee(self, vtype, seconds):
        hours = max(1, math.ceil(seconds / 3600))                       # minimum one hour, partial hour rounds up
        return hours * self.rates[vtype]

class ParkingLot:
    def __init__(self, slots, fee_strategy, clock=time.time):
        self._free = {t: [] for t in VehicleType}                       # min-heap per type: nearest = lowest (floor, id)
        for s in slots: heapq.heappush(self._free[s.type], (s.floor, s.id, s))
        self._tickets, self._active, self._fee, self._clock = {}, set(), fee_strategy, clock
        self._ids, self._lock = itertools.count(1), threading.Lock()
    def park(self, v: Vehicle) -> Ticket:
        with self._lock:                                                # one critical section: check + allocate + record
            if v.plate in self._active: raise AlreadyParkedError(v.plate)
            heap = self._free[v.type]
            if not heap: raise LotFullError(v.type.name)
            _, _, slot = heapq.heappop(heap)
            t = Ticket(f'T{next(self._ids)}', v, slot, self._clock())
            self._tickets[t.id] = t; self._active.add(v.plate)
            return t
    def exit(self, ticket_id: str) -> int:
        with self._lock:
            t = self._tickets.pop(ticket_id, None)                      # pop makes a second exit fail (no double exit)
            if t is None: raise InvalidTicketError(ticket_id)
            self._active.discard(t.vehicle.plate)
            heapq.heappush(self._free[t.slot.type], (t.slot.floor, t.slot.id, t.slot))
            return self._fee.fee(t.vehicle.type, self._clock() - t.entry_time)
```

```cpp
enum class VType { Motorcycle, Car, Truck };
struct Vehicle { std::string plate; VType type; };
struct Slot { int floor; int id; };
struct Ticket { int id; Vehicle vehicle; Slot slot; std::chrono::steady_clock::time_point entry; };
class FeeStrategy { public: virtual ~FeeStrategy() = default; virtual long fee(VType, long seconds) const = 0; };
class HourlyFee : public FeeStrategy {
    std::map<VType, long> rate_;
public:
    explicit HourlyFee(std::map<VType, long> r) : rate_(std::move(r)) {}
    long fee(VType t, long s) const override { long h = std::max(1L, (s + 3599) / 3600); return h * rate_.at(t); }
};
class ParkingLot {
    std::map<VType, std::set<std::pair<int,int>>> free_;      // ordered by (floor, id): begin() is the nearest slot
    std::unordered_map<int, Ticket> tickets_; std::unordered_set<std::string> active_;
    std::unique_ptr<FeeStrategy> fee_; std::mutex m_; int nextId_ = 1;
public:
    explicit ParkingLot(std::unique_ptr<FeeStrategy> f) : fee_(std::move(f)) {}
    void addSlot(VType t, int floor, int id) { free_[t].insert({floor, id}); }
    Ticket park(const Vehicle& v) {
        std::lock_guard<std::mutex> g(m_);
        if (active_.count(v.plate)) throw std::logic_error("already parked");
        auto& fs = free_[v.type];
        if (fs.empty()) throw std::runtime_error("lot full");
        auto [fl, id] = *fs.begin(); fs.erase(fs.begin());
        Ticket t{nextId_++, v, {fl, id}, std::chrono::steady_clock::now()};
        tickets_[t.id] = t; active_.insert(v.plate);
        return t;
    }
    long exit(int ticketId) {
        std::lock_guard<std::mutex> g(m_);
        auto it = tickets_.find(ticketId);
        if (it == tickets_.end()) throw std::invalid_argument("invalid ticket");
        Ticket t = it->second; tickets_.erase(it); active_.erase(t.vehicle.plate);
        free_[t.vehicle.type].insert({t.slot.floor, t.slot.id});
        long secs = std::chrono::duration_cast<std::chrono::seconds>(std::chrono::steady_clock::now() - t.entry).count();
        return fee_->fee(t.vehicle.type, secs);
    }
};
```

**Complexity:** `park` and `exit` are O(log n) with the free-slot heap/set (a linear scan would be O(n)).

**Edge cases and extensions:** lot full; same vehicle entering twice; invalid or reused ticket; **a car may be allowed in a larger slot** (fallback policy in the allocation strategy); lost ticket (max daily fee); reservation and EV-charging slots (new slot types, no change to `ParkingLot` if allocation is a strategy); multiple entry gates (the lock covers allocation); fee tiers (a new `FeeStrategy`); clock skew (inject clock, use monotonic time); persistence (a repository behind the lot).

**Tests:** park then exit returns the fee for 1 hour minimum; 61 minutes bills 2 hours; fill all car slots then the next car raises `LotFullError`; exit frees the slot for the next car; exiting twice raises `InvalidTicketError`; parking the same plate twice fails; two threads parking into the last slot: exactly one succeeds.

---

## L2. Splitwise (expense sharing)

**Requirements:** users and groups; add an expense paid by one user and split **equally**, by **exact amounts**, or by **percentages**; track who owes whom; show balances; **simplify debts** (minimize the number of settlements); settle up.

**Entities and patterns:** `User`, `Expense` (id, payer, total, splits), `Split` strategy (**Strategy/Factory**: `EqualSplit`, `ExactSplit`, `PercentSplit`), `ExpenseManager`/`BalanceSheet`. **Observer** for "you were added to an expense" notifications.

**Key design decisions:** keep **money as integer paise**; equal split must **distribute the remainder** (100 split three ways = 34 + 33 + 33, total stays exact); validate (exact sums to total, percentages sum to 100); maintain a **net balance per user** (positive = others owe you).

```python
from collections import defaultdict
import heapq

class SplitError(ValueError): ...

def equal_shares(total, users):
    base, rem = divmod(total, len(users))
    return {u: base + (1 if i < rem else 0) for i, u in enumerate(users)}       # first `rem` users pay one extra paisa
def exact_shares(total, amounts):
    if sum(amounts.values()) != total: raise SplitError('exact amounts must sum to total')
    return dict(amounts)
def percent_shares(total, pcts):
    if sum(pcts.values()) != 100: raise SplitError('percentages must sum to 100')
    shares = {u: total * p // 100 for u, p in pcts.items()}
    leftover = total - sum(shares.values())                                      # distribute rounding leftovers
    for u in list(shares)[:leftover]: shares[u] += 1
    return shares

class ExpenseManager:
    def __init__(self):
        self.owes = defaultdict(lambda: defaultdict(int))       # owes[debtor][creditor] = amount
        self.net = defaultdict(int)                              # net[user] > 0: others owe this user
    def add_expense(self, payer, total, shares):
        if sum(shares.values()) != total: raise SplitError('shares do not add up')
        for user, share in shares.items():
            if user == payer: continue
            self.owes[user][payer] += share
            self.net[payer] += share; self.net[user] -= share
    def balances(self):                                         # pairwise, netted both ways
        out = []
        for a in list(self.owes):
            for b, amt in self.owes[a].items():
                rev = self.owes[b].get(a, 0)
                if amt > rev: out.append((a, b, amt - rev))     # a owes b
        return out
    def settle(self, debtor, creditor, amount):
        self.owes[debtor][creditor] -= amount
        self.net[debtor] += amount; self.net[creditor] -= amount
    def simplify(self):
        # Min cash flow: repeatedly match the biggest debtor with the biggest creditor. At most (n - 1) transactions.
        cred = [(-v, u) for u, v in self.net.items() if v > 0]; debt = [(v, u) for u, v in self.net.items() if v < 0]
        heapq.heapify(cred); heapq.heapify(debt)
        txns = []
        while cred and debt:
            c, cu = heapq.heappop(cred); d, du = heapq.heappop(debt)
            c, d = -c, -d                                        # amounts as positives
            pay = min(c, d); txns.append((du, cu, pay))
            if c > pay: heapq.heappush(cred, (-(c - pay), cu))
            if d > pay: heapq.heappush(debt, (-(d - pay), du))
        return txns

m = ExpenseManager()
m.add_expense('A', 3000, equal_shares(3000, ['A', 'B', 'C']))   # A paid 30.00 for three people
m.add_expense('B', 1000, equal_shares(1000, ['A', 'B', 'C']))   # B paid 10.00
print(m.simplify())          # [('C', 'A', 1333), ('B', 'A', 667)]: A is owed 20.00 net; C owes 13.33, B owes 6.67
```

```cpp
// Debt simplification (net[u] > 0: others owe u). Greedy with two max-heaps.
std::vector<std::tuple<std::string, std::string, long>> simplify(const std::map<std::string, long>& net) {
    std::priority_queue<std::pair<long, std::string>> cred, debt;
    for (auto& [u, v] : net) { if (v > 0) cred.push({v, u}); else if (v < 0) debt.push({-v, u}); }
    std::vector<std::tuple<std::string, std::string, long>> txns;
    while (!cred.empty() && !debt.empty()) {
        auto [c, cu] = cred.top(); cred.pop();
        auto [d, du] = debt.top(); debt.pop();
        long pay = std::min(c, d);
        txns.emplace_back(du, cu, pay);                               // du pays cu
        if (c > pay) cred.push({c - pay, cu});
        if (d > pay) debt.push({d - pay, du});
    }
    return txns;
}
```

**Complexity:** adding an expense O(participants); simplification O(n log n) (finding the true minimum number of transactions is NP-hard; the greedy heap method is the standard interview answer and gives at most n - 1 payments).

**Edge cases:** payer not in the split (allowed: they paid for others); one participant only; zero or negative amounts; unknown users; rounding; percent not summing to 100; currency mismatch; deleting/editing an expense (store expenses as events and recompute, or apply the inverse); concurrent updates (lock per group); large groups.

**Tests:** 100 split among 3 gives 34/33/33; exact amounts not summing to total raises; two expenses net correctly; after `simplify`, applying the payments zeroes every net balance; payer-only expense creates no debt.

---

## L3. LRU Cache with fixed capacity

**Requirements:** `get(key)` and `put(key, value)` both in **O(1)**; capacity `N`; when full, evict the **least recently used** entry; both reads and writes count as use.

**Design:** a **hash map** (key → node) for O(1) lookup + a **doubly linked list** ordered by recency (head = most recent, tail = least recent) for O(1) move and evict. Sentinel head/tail nodes remove null checks.

```
head <-> [k3] <-> [k1] <-> [k2] <-> tail          map: {k1->node, k2->node, k3->node}
 most recent                least recent (evict this one on overflow)
```

```python
import threading
class _Node:
    __slots__ = ('key', 'val', 'prev', 'next')
    def __init__(self, key=None, val=None): self.key, self.val, self.prev, self.next = key, val, None, None

class LRUCache:
    def __init__(self, capacity):
        if capacity <= 0: raise ValueError('capacity must be positive')
        self.cap, self.map = capacity, {}
        self.head, self.tail = _Node(), _Node()
        self.head.next, self.tail.prev = self.tail, self.head
        self._lock = threading.RLock()
    def _unlink(self, n): n.prev.next, n.next.prev = n.next, n.prev
    def _push_front(self, n):
        n.next, n.prev = self.head.next, self.head
        self.head.next.prev = n; self.head.next = n
    def get(self, key, default=None):
        with self._lock:
            n = self.map.get(key)
            if n is None: return default
            self._unlink(n); self._push_front(n)             # mark most recently used
            return n.val
    def put(self, key, val):
        with self._lock:
            n = self.map.get(key)
            if n:                                             # update existing: change value AND refresh recency
                n.val = val; self._unlink(n); self._push_front(n); return
            if len(self.map) == self.cap:
                lru = self.tail.prev; self._unlink(lru); del self.map[lru.key]      # evict
            n = _Node(key, val); self.map[key] = n; self._push_front(n)

# Idiomatic Python shortcut (OrderedDict keeps insertion order, move_to_end is O(1)):
from collections import OrderedDict
class LRUCache2:
    def __init__(self, cap): self.cap, self.d = cap, OrderedDict()
    def get(self, k):
        if k not in self.d: return None
        self.d.move_to_end(k); return self.d[k]
    def put(self, k, v):
        self.d[k] = v; self.d.move_to_end(k)
        if len(self.d) > self.cap: self.d.popitem(last=False)
```

```cpp
template <class K, class V>
class LRUCache {
    size_t cap_; std::list<std::pair<K, V>> items_;                         // front = most recent
    std::unordered_map<K, typename std::list<std::pair<K, V>>::iterator> pos_;
    mutable std::mutex m_;
public:
    explicit LRUCache(size_t cap) : cap_(cap) { if (cap == 0) throw std::invalid_argument("capacity"); }
    std::optional<V> get(const K& k) {
        std::lock_guard<std::mutex> g(m_);
        auto it = pos_.find(k);
        if (it == pos_.end()) return std::nullopt;
        items_.splice(items_.begin(), items_, it->second);                  // O(1) move to front, iterator stays valid
        return it->second->second;
    }
    void put(const K& k, V v) {
        std::lock_guard<std::mutex> g(m_);
        auto it = pos_.find(k);
        if (it != pos_.end()) { it->second->second = std::move(v); items_.splice(items_.begin(), items_, it->second); return; }
        if (items_.size() == cap_) { pos_.erase(items_.back().first); items_.pop_back(); }
        items_.emplace_front(k, std::move(v)); pos_[k] = items_.begin();
    }
};
```

**Complexity:** O(1) time per operation; O(capacity) space. Note `get` **mutates** the structure, so a read lock is not enough (use one mutex, or shard the cache by key hash to reduce contention).

**Extensions:** generic key/value types; eviction callback; **LFU** (frequency buckets), **TTL** (L4); an `EvictionPolicy` strategy (LRU/FIFO/LFU) so the cache class is closed for modification; write-through vs write-back; metrics (hit ratio).

**Edge cases:** capacity 1; updating an existing key must not evict anything; capacity 0 or negative; `None`/null values vs a miss (return an `optional` or sentinel); very large values (size-based eviction).

**Tests:** put 1, 2, get 1, put 3 (cap 2) evicts 2; put existing key refreshes recency; capacity 1 behaviour; get miss returns none; concurrent get/put never corrupts the list or exceeds capacity.

---

## L4. Cache with TTL expiration

**Requirements:** each entry has a **time-to-live**; expired entries must not be returned; optionally combine with a capacity limit (LRU). Expired entries must eventually be **reclaimed** even if never read.

**Design choices**

| Strategy | How | Trade-off |
| --- | --- | --- |
| **Lazy expiration** | check expiry on `get`, delete if expired | O(1), but never-read keys stay in memory |
| **Active sweeping** | background thread or periodic scan removes expired keys | frees memory; scan cost, extra thread |
| **Expiry min-heap** | heap of `(expires_at, key)`; pop while the top is expired | efficient reclamation in O(log n) per expired key |
| **Combination (recommended)** | lazy check on read + heap purge on writes + optional sweeper | what Redis does (lazy + sampled active expiry) |

```python
import heapq, threading, time
class TTLCache:
    def __init__(self, default_ttl, clock=time.monotonic):          # monotonic: immune to wall-clock changes; injectable for tests
        self.default_ttl, self.clock = default_ttl, clock
        self.data = {}                                               # key -> (value, expires_at)
        self.heap = []                                               # (expires_at, key); may hold STALE entries after updates
        self.lock = threading.Lock()
    def put(self, key, value, ttl=None):
        with self.lock:
            exp = self.clock() + (self.default_ttl if ttl is None else ttl)
            self.data[key] = (value, exp)
            heapq.heappush(self.heap, (exp, key))
            self._purge()
    def get(self, key):
        with self.lock:
            item = self.data.get(key)
            if item is None: return None
            value, exp = item
            if exp <= self.clock():                                  # lazy expiry
                del self.data[key]; return None
            return value
    def _purge(self):
        now = self.clock()
        while self.heap and self.heap[0][0] <= now:
            exp, key = heapq.heappop(self.heap)
            item = self.data.get(key)
            if item and item[1] == exp: del self.data[key]           # skip stale heap entries (key was re-put with a new expiry)
    def size(self):
        with self.lock: self._purge(); return len(self.data)

# Test with a fake clock (no sleeping): c = [0]; cache = TTLCache(10, clock=lambda: c[0]); cache.put('a', 1); c[0] = 11; assert cache.get('a') is None
```

```cpp
class TTLCache {
    using Clock = std::chrono::steady_clock;
    struct Entry { std::string value; Clock::time_point expires; };
    std::unordered_map<std::string, Entry> data_; std::chrono::seconds defaultTtl_; mutable std::mutex m_;
public:
    explicit TTLCache(std::chrono::seconds ttl) : defaultTtl_(ttl) {}
    void put(const std::string& k, std::string v, std::optional<std::chrono::seconds> ttl = std::nullopt) {
        std::lock_guard<std::mutex> g(m_);
        data_[k] = {std::move(v), Clock::now() + ttl.value_or(defaultTtl_)};
    }
    std::optional<std::string> get(const std::string& k) {
        std::lock_guard<std::mutex> g(m_);
        auto it = data_.find(k);
        if (it == data_.end()) return std::nullopt;
        if (it->second.expires <= Clock::now()) { data_.erase(it); return std::nullopt; }      // lazy expiry
        return it->second.value;
    }
    size_t sweep() {                                                  // call periodically from a background thread
        std::lock_guard<std::mutex> g(m_); size_t removed = 0; auto now = Clock::now();
        for (auto it = data_.begin(); it != data_.end();) { if (it->second.expires <= now) { it = data_.erase(it); ++removed; } else ++it; }
        return removed;
    }
};
```

**Complexity:** `put`/`get` O(1) average (+ O(log n) per purged entry with the heap); the C++ `sweep` is O(n) (use a heap or sampling for big caches).

**Edge cases:** re-putting a key resets its TTL (stale heap entries must be ignored); TTL of 0 or negative; expiry exactly at `now` (decide `<=` vs `<`); **expired entries counting toward capacity** (purge before evicting by LRU); wall-clock changes (use monotonic time); **cache stampede** (many threads miss at once and reload; mitigate with per-key locks or a single-flight loader, jittered TTLs); memory growth without a sweeper; refresh-ahead vs expire-then-load.

**Tests:** value present before TTL, absent after (fake clock); per-entry TTL overrides default; re-put extends the life; expired entry does not count in `size`; concurrent puts and gets are safe; sweeper removes unread keys.

---

## L5. Elevator system

**Requirements:** N elevators serving M floors; **hall calls** (floor + direction button) and **car calls** (destination button inside); a **dispatcher** assigns hall calls to a car; each car moves, opens doors and serves stops efficiently; states idle/moving/door open; emergency stop and overload are extensions.

**Entities and patterns:** `ElevatorSystem` (controller/dispatcher), `Elevator` (floor, direction, pending stops, state), `Request`, `Direction`, `Door`, `Display`. **Strategy** for dispatch (nearest car, least load, zoning), **Strategy/algorithm** for movement (**SCAN/LOOK**: keep going in the current direction while there are stops ahead, then reverse), **State** for IDLE/MOVING\_UP/MOVING\_DOWN/DOOR\_OPEN, **Command** for requests, **Observer** for floor displays.

```
LOOK algorithm: car at 3 going UP with stops {5, 8, 1}:
  3 -> 4 -> 5 (serve) -> 6 -> 7 -> 8 (serve) -> no stops above, reverse -> 7 ... -> 1 (serve) -> IDLE
```

```python
from enum import Enum
class Direction(Enum): UP = 1; DOWN = -1; IDLE = 0

class Elevator:
    def __init__(self, eid, min_floor=0, max_floor=9, capacity=10):
        self.id, self.floor, self.dir = eid, min_floor, Direction.IDLE
        self.min_floor, self.max_floor, self.capacity = min_floor, max_floor, capacity
        self.stops, self.log = set(), []
    def add_stop(self, floor):
        if not (self.min_floor <= floor <= self.max_floor): raise ValueError('invalid floor')
        self.stops.add(floor)                                   # a set removes duplicate requests for free
    def _ahead(self, d):
        return any((f > self.floor) if d == Direction.UP else (f < self.floor) for f in self.stops)
    def step(self):                                              # advance the simulation by one tick
        if self.floor in self.stops:                             # arrived: open door and serve
            self.stops.discard(self.floor); self.log.append(f'open@{self.floor}'); return
        if not self.stops: self.dir = Direction.IDLE; return
        if self.dir == Direction.IDLE:                           # pick the direction of the nearest stop
            nearest = min(self.stops, key=lambda f: abs(f - self.floor))
            self.dir = Direction.UP if nearest > self.floor else Direction.DOWN
        elif not self._ahead(self.dir):                          # LOOK: nothing ahead, reverse
            self.dir = Direction.DOWN if self.dir == Direction.UP else Direction.UP
        self.floor += self.dir.value
    def cost(self, floor, direction):                            # lower is better
        d = abs(self.floor - floor)
        if self.dir == Direction.IDLE: return d
        toward = (self.dir == Direction.UP and floor >= self.floor) or (self.dir == Direction.DOWN and floor <= self.floor)
        return d if (toward and direction == self.dir) else d + 2 * (self.max_floor - self.min_floor)   # penalise reversing

class ElevatorSystem:
    def __init__(self, elevators): self.elevators = elevators
    def hall_call(self, floor, direction):
        best = min(self.elevators, key=lambda e: e.cost(floor, direction))   # dispatch strategy (swap for another)
        best.add_stop(floor); return best.id
    def car_call(self, elevator_id, floor): self.elevators[elevator_id].add_stop(floor)
    def tick(self):
        for e in self.elevators: e.step()
```

```cpp
enum class Dir { Up = 1, Down = -1, Idle = 0 };
class Elevator {
    int floor_ = 0; Dir dir_ = Dir::Idle; std::set<int> stops_; int minF_, maxF_;
    bool ahead(Dir d) const { for (int f : stops_) if ((d == Dir::Up && f > floor_) || (d == Dir::Down && f < floor_)) return true; return false; }
public:
    Elevator(int minF, int maxF) : floor_(minF), minF_(minF), maxF_(maxF) {}
    void addStop(int f) { if (f < minF_ || f > maxF_) throw std::out_of_range("floor"); stops_.insert(f); }
    void step() {
        if (stops_.erase(floor_)) { /* open doors */ return; }
        if (stops_.empty()) { dir_ = Dir::Idle; return; }
        if (dir_ == Dir::Idle) {
            int best = *stops_.begin();
            for (int f : stops_) if (std::abs(f - floor_) < std::abs(best - floor_)) best = f;
            dir_ = best > floor_ ? Dir::Up : Dir::Down;
        } else if (!ahead(dir_)) dir_ = (dir_ == Dir::Up) ? Dir::Down : Dir::Up;
        floor_ += static_cast<int>(dir_);
    }
    int floor() const { return floor_; }
};
```

**Concurrency:** requests arrive from many threads: put them on a thread-safe queue consumed by the controller loop, or guard each elevator's `stops` with a lock; the movement loop is single-threaded per elevator (a thread or an event loop).

**Edge cases:** request for the current floor (open the door immediately); duplicate requests (set); invalid floors; direction reversal at the top/bottom floor; hall call direction must match (a person going down should not board an up-moving car); overload (reject new car calls above `capacity`); door-open timeout and door-obstruction sensor; emergency stop and fire mode (ignore calls, go to ground floor); starvation of far requests (LOOK serves everything in a sweep, nearest-first greedy can starve).

**Tests:** car at 0 with stops {5, 2} serves 2 then 5; a call at the current floor opens immediately; duplicate calls produce one stop; the dispatcher chooses the idle nearby car over a distant moving one; after serving all stops the car becomes IDLE.

---

## L6. Library management system

**Requirements:** a catalogue of **books** with several physical **copies** each; **members** borrow up to N copies for a fixed loan period; return, **renew** (limited, not if someone is waiting), **fines** for late returns, **reserve** a book when no copy is free, **search** by title/author/subject, notify a member when a reserved book arrives.

**Entities and patterns:** `Book` (isbn, title, authors, subjects: the *title*), `BookCopy` (barcode, status: the *physical item*; one Book has many copies), `Member`, `Loan` (copy, member, issued, due, returned), `Reservation` (a FIFO queue per ISBN), `Catalog` (search indexes), `FinePolicy` (**Strategy**), `LibraryService` (orchestrator), **Observer** for availability notifications. Key idea: **separate Book from BookCopy**; most interview mistakes come from merging them.

```python
from abc import ABC, abstractmethod
from collections import defaultdict, deque
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
import math, threading

class CopyStatus(Enum): AVAILABLE = 1; LOANED = 2; HELD = 3; LOST = 4        # HELD: reserved for a waiting member
class LibraryError(Exception): ...
class NotAvailable(LibraryError): ...
class LimitExceeded(LibraryError): ...

@dataclass(frozen=True)
class Book: isbn: str; title: str; authors: tuple; subjects: tuple = ()
@dataclass
class BookCopy: barcode: str; book: Book; status: CopyStatus = CopyStatus.AVAILABLE; held_for: str | None = None
@dataclass
class Member: id: str; name: str; fines_due: int = 0
@dataclass
class Loan: copy: BookCopy; member: Member; issued: datetime; due: datetime; renewals: int = 0; returned: datetime | None = None

class FinePolicy(ABC):
    @abstractmethod
    def fine(self, due: datetime, returned: datetime) -> int: ...
class PerDayFine(FinePolicy):
    def __init__(self, per_day): self.per_day = per_day
    def fine(self, due, returned):
        late_days = max(0, math.ceil((returned - due) / timedelta(days=1)))
        return late_days * self.per_day

class Library:
    MAX_LOANS, LOAN_DAYS, MAX_RENEWALS, MAX_FINE = 5, 14, 2, 500
    def __init__(self, fine_policy, clock=datetime.now, notifier=print):
        self.copies, self.by_isbn = {}, defaultdict(list)                 # barcode -> copy ; isbn -> copies
        self.title_idx, self.author_idx = defaultdict(set), defaultdict(set)
        self.loans, self.member_loans = {}, defaultdict(list)             # barcode -> active Loan ; member id -> loans
        self.reservations = defaultdict(deque)                            # isbn -> queue of member ids
        self.members, self.fine_policy, self.clock, self.notify = {}, fine_policy, clock, notifier
        self._lock = threading.RLock()
    def add_book(self, book, n_copies):
        for i in range(n_copies):
            c = BookCopy(f'{book.isbn}-{len(self.by_isbn[book.isbn]) + 1}', book)
            self.copies[c.barcode] = c; self.by_isbn[book.isbn].append(c)
        for w in book.title.lower().split(): self.title_idx[w].add(book.isbn)
        for a in book.authors: self.author_idx[a.lower()].add(book.isbn)
    def search(self, title_word=None, author=None):
        ids = None
        for found in ([self.title_idx[title_word.lower()]] if title_word else []) + ([self.author_idx[author.lower()]] if author else []):
            ids = set(found) if ids is None else ids & found
        return [self.by_isbn[i][0].book for i in (ids or [])]
    def checkout(self, member_id, isbn):
        with self._lock:
            m = self.members[member_id]
            if len(self.member_loans[member_id]) >= self.MAX_LOANS: raise LimitExceeded('loan limit')
            if m.fines_due > self.MAX_FINE: raise LibraryError('clear fines first')
            copy = next((c for c in self.by_isbn[isbn]
                         if (c.status == CopyStatus.AVAILABLE and not self.reservations[isbn])
                         or (c.status == CopyStatus.HELD and c.held_for == member_id)), None)
            if copy is None: raise NotAvailable('no copy available; reserve it')
            if copy.status == CopyStatus.HELD: self.reservations[isbn].remove(member_id)
            now = self.clock()
            copy.status, copy.held_for = CopyStatus.LOANED, None
            loan = Loan(copy, m, now, now + timedelta(days=self.LOAN_DAYS))
            self.loans[copy.barcode] = loan; self.member_loans[member_id].append(loan)
            return loan
    def reserve(self, member_id, isbn):
        with self._lock:
            if member_id not in self.reservations[isbn]: self.reservations[isbn].append(member_id)
    def return_copy(self, barcode):
        with self._lock:
            loan = self.loans.pop(barcode, None)
            if loan is None: raise LibraryError('copy is not on loan')    # double return
            loan.returned = self.clock()
            loan.member.fines_due += self.fine_policy.fine(loan.due, loan.returned)
            self.member_loans[loan.member.id].remove(loan)
            q = self.reservations[loan.copy.book.isbn]
            if q:                                                           # hand the copy to the first waiting member
                loan.copy.status, loan.copy.held_for = CopyStatus.HELD, q[0]
                self.notify(f'notify {q[0]}: {loan.copy.book.title} is ready to borrow')
            else: loan.copy.status = CopyStatus.AVAILABLE
    def renew(self, barcode):
        with self._lock:
            loan = self.loans[barcode]
            if loan.renewals >= self.MAX_RENEWALS or self.reservations[loan.copy.book.isbn]:
                raise LibraryError('cannot renew')
            loan.due += timedelta(days=self.LOAN_DAYS); loan.renewals += 1
```

```cpp
// Core C++ shape (checkout and return); same entities as above
enum class CopyStatus { Available, Loaned, Held, Lost };
struct BookCopy { std::string barcode, isbn; CopyStatus status = CopyStatus::Available; std::string heldFor; };
struct Loan { BookCopy* copy; std::string memberId; std::chrono::system_clock::time_point issued, due; int renewals = 0; };
class Library {
    std::unordered_map<std::string, std::vector<BookCopy>> byIsbn_;
    std::unordered_map<std::string, Loan> loans_;                   // barcode -> active loan
    std::unordered_map<std::string, std::deque<std::string>> reservations_;
    std::unordered_map<std::string, int> loanCount_;
    std::mutex m_; static constexpr int kMaxLoans = 5;
public:
    Loan checkout(const std::string& member, const std::string& isbn) {
        std::lock_guard<std::mutex> g(m_);
        if (loanCount_[member] >= kMaxLoans) throw std::runtime_error("loan limit");
        for (auto& c : byIsbn_[isbn]) {
            bool free = c.status == CopyStatus::Available && reservations_[isbn].empty();
            bool mine = c.status == CopyStatus::Held && c.heldFor == member;
            if (!free && !mine) continue;
            if (mine) { auto& q = reservations_[isbn]; q.erase(std::find(q.begin(), q.end(), member)); }
            c.status = CopyStatus::Loaned; c.heldFor.clear(); loanCount_[member]++;
            auto now = std::chrono::system_clock::now();
            return loans_[c.barcode] = Loan{&c, member, now, now + std::chrono::hours(24 * 14), 0};
        }
        throw std::runtime_error("not available");
    }
};
```

**Complexity:** search O(result) via inverted indexes; checkout O(copies of that ISBN); all other operations O(1).

**Edge cases:** member at the loan limit or with large fines; returning a copy not on loan; renewing when someone reserved the title; **held copy expiry** (if the reserved member does not borrow within 48 hours, pass to the next: needs a scheduled job); lost/damaged copies (replacement-cost fine, status LOST); two members racing for the last copy (the lock makes one win); member borrowing a second copy of a title they already hold; clock injection for fine tests; persistence and an audit trail of loans.

**Tests:** borrow then return on time gives no fine; 3 days late with 10 per day gives 30; a sixth checkout raises `LimitExceeded`; reserve + return makes the copy HELD for the first member and a stranger cannot borrow it; renew is rejected when a reservation exists; double return raises.

---

## L7. Movie ticket booking (BookMyShow style)

**Requirements:** list shows by city/movie/date; show a **seat map**; user selects seats; seats are **held for a few minutes** while paying; confirm after payment; cancel with refund; **no two users can ever buy the same seat**.

**The core problem is concurrency**: two users click the same seat at once. The solution is a **seat lock with expiry** and an atomic **all-or-nothing** acquisition.

**Entities and patterns:** `Movie`, `Theater` → `Screen` → `Seat`; `Show` (movie, screen, start time) holding per-show seat state; `Booking` (user, show, seats, amount, status); `PricingStrategy` (**Strategy**: weekend, seat tier), `PaymentGateway` (adapter/interface), `BookingService` (**Facade**), seat lifecycle **State**: `AVAILABLE → LOCKED → BOOKED` (and back to AVAILABLE on expiry/cancel).

```python
import itertools, threading, time
from dataclasses import dataclass
from enum import Enum

class SeatState(Enum): AVAILABLE = 1; LOCKED = 2; BOOKED = 3
class SeatUnavailable(Exception): ...
class LockExpired(Exception): ...

@dataclass
class SeatInfo: price: int; state: SeatState = SeatState.AVAILABLE; owner: str | None = None; locked_until: float = 0

class Show:
    LOCK_TTL = 300                                                   # seconds to complete payment
    def __init__(self, show_id, movie, seat_prices, clock=time.monotonic):
        self.id, self.movie, self.clock = show_id, movie, clock
        self.seats = {sid: SeatInfo(p) for sid, p in seat_prices.items()}
        self._lock = threading.Lock()                                # per-show lock: different shows never contend
    def _free_expired(self, now):
        for s in self.seats.values():
            if s.state == SeatState.LOCKED and s.locked_until <= now: s.state, s.owner = SeatState.AVAILABLE, None
    def lock_seats(self, user, seat_ids):
        with self._lock:
            now = self.clock(); self._free_expired(now)
            for sid in seat_ids:                                     # phase 1: validate ALL (all-or-nothing)
                s = self.seats.get(sid)
                if s is None or s.state != SeatState.AVAILABLE: raise SeatUnavailable(sid)
            for sid in seat_ids:                                     # phase 2: lock ALL
                s = self.seats[sid]; s.state, s.owner, s.locked_until = SeatState.LOCKED, user, now + self.LOCK_TTL
    def release(self, user, seat_ids):
        with self._lock:
            for sid in seat_ids:
                s = self.seats[sid]
                if s.state == SeatState.LOCKED and s.owner == user: s.state, s.owner = SeatState.AVAILABLE, None
    def confirm(self, user, seat_ids):                               # call ONLY after payment succeeded
        with self._lock:
            now = self.clock(); total = 0
            for sid in seat_ids:
                s = self.seats[sid]
                if not (s.state == SeatState.LOCKED and s.owner == user and s.locked_until > now): raise LockExpired(sid)
                total += s.price
            for sid in seat_ids: self.seats[sid].state = SeatState.BOOKED
            return total

class BookingService:
    _ids = itertools.count(1)
    def __init__(self, gateway): self.gateway, self.bookings = gateway, {}
    def book(self, user, show, seat_ids):
        show.lock_seats(user, seat_ids)                              # 1. hold seats
        try:
            amount = sum(show.seats[s].price for s in seat_ids)
            if not self.gateway.charge(user, amount): raise RuntimeError('payment failed')   # 2. pay
            show.confirm(user, seat_ids)                             # 3. confirm (re-validates the lock)
        except Exception:
            show.release(user, seat_ids); raise                      # failure at any step frees the seats
        bid = next(self._ids); self.bookings[bid] = (user, show.id, tuple(seat_ids), amount); return bid
```

```cpp
class Show {
    enum class State { Available, Locked, Booked };
    struct Seat { int price; State state = State::Available; std::string owner; std::chrono::steady_clock::time_point until; };
    std::unordered_map<int, Seat> seats_; std::mutex m_;
    using Clock = std::chrono::steady_clock;
public:
    void addSeat(int id, int price) { seats_[id].price = price; }
    bool lockSeats(const std::string& user, const std::vector<int>& ids, std::chrono::seconds ttl) {
        std::lock_guard<std::mutex> g(m_); auto now = Clock::now();
        for (auto& [id, s] : seats_) if (s.state == State::Locked && s.until <= now) { s.state = State::Available; s.owner.clear(); }
        for (int id : ids) { auto it = seats_.find(id); if (it == seats_.end() || it->second.state != State::Available) return false; }
        for (int id : ids) { auto& s = seats_[id]; s.state = State::Locked; s.owner = user; s.until = now + ttl; }
        return true;                                              // all or nothing
    }
};
```

**Production version (distributed):** the lock must live in shared storage, not in process memory. With a SQL database, make the seat acquisition one atomic statement and check the affected row count:

```sql
CREATE TABLE show_seat (show_id INT, seat_id INT, status VARCHAR(10), locked_by BIGINT, locked_until TIMESTAMP,
                        PRIMARY KEY (show_id, seat_id));            -- unique per seat per show
BEGIN;
UPDATE show_seat SET status = 'LOCKED', locked_by = :user, locked_until = now() + interval '5 minutes'
WHERE show_id = :show AND seat_id IN (:s1, :s2)
  AND (status = 'AVAILABLE' OR (status = 'LOCKED' AND locked_until < now()));
-- if rows_affected = number of requested seats then COMMIT else ROLLBACK (someone else got one)
```

Or Redis `SET seat:{show}:{seat} {user} NX EX 300` per seat (release already-acquired seats if a later one fails); final confirmation writes a `booking` row with a **unique constraint** on (show\_id, seat\_id) as the last line of defence.

**Edge cases:** lock expires while the user is paying (decide: extend the lock when payment starts, or refund automatically); payment succeeded but confirm failed (**refund/compensate**, never leave the customer charged); duplicate confirm calls (**idempotency key** per booking); partial seat sets (all-or-nothing); a user locking dozens of seats (limit seats per booking and per user); cancellation windows and refunds; show started; price changes between lock and pay (price is fixed at lock time); thundering herd on popular shows.

**Tests:** two threads lock the same seat, exactly one succeeds; locking \[A1, A2\] when A2 is taken locks neither; expired locks can be re-locked by someone else; confirm after expiry raises `LockExpired`; payment failure releases the seats.

---

## L8. Food delivery system (Swiggy/Zomato style)

**Requirements:** customers browse restaurants and menus, build a **cart** (items from one restaurant), place an **order** and pay; the restaurant accepts and prepares; a **delivery partner** is assigned, picks up and delivers; the customer tracks status; cancellation rules; ratings (extension).

**Entities and patterns:** `Customer`, `Restaurant`, `MenuItem` (price, availability), `Cart`, `Order` (with `OrderItem`s, **State machine**), `DeliveryPartner` (location, availability), `Payment`, `Address`; services: `RestaurantService` (search), `OrderService` (**Facade/orchestrator**), `AssignmentStrategy` (**Strategy**: nearest available, least busy), `PricingStrategy`/discounts (**Strategy/Decorator**), `NotificationService` (**Observer**).

```
Order lifecycle (State):
 PLACED -> ACCEPTED -> PREPARING -> READY -> PICKED_UP -> DELIVERED
    |         |
    +---------+--> CANCELLED   (allowed only before PREPARING; refund triggered)      REJECTED (restaurant declines)
```

```python
from dataclasses import dataclass, field
from enum import Enum
import itertools, threading

class OrderState(Enum):
    PLACED = 1; ACCEPTED = 2; PREPARING = 3; READY = 4; PICKED_UP = 5; DELIVERED = 6; CANCELLED = 7; REJECTED = 8
ALLOWED = {OrderState.PLACED: {OrderState.ACCEPTED, OrderState.REJECTED, OrderState.CANCELLED},
           OrderState.ACCEPTED: {OrderState.PREPARING, OrderState.CANCELLED},
           OrderState.PREPARING: {OrderState.READY}, OrderState.READY: {OrderState.PICKED_UP},
           OrderState.PICKED_UP: {OrderState.DELIVERED}}                # absent = terminal
class InvalidTransition(Exception): ...

@dataclass
class MenuItem: id: str; name: str; price: int; in_stock: bool = True
@dataclass
class Restaurant: id: str; name: str; menu: dict; open: bool = True; loc: tuple = (0, 0)
@dataclass
class Partner: id: str; loc: tuple; available: bool = True
@dataclass
class Order:
    id: int; customer: str; restaurant: Restaurant; items: dict; amount: int
    state: OrderState = OrderState.PLACED; partner: Partner | None = None
    def move_to(self, new_state, observers=()):
        if new_state not in ALLOWED.get(self.state, set()): raise InvalidTransition(f'{self.state.name} -> {new_state.name}')
        self.state = new_state
        for o in observers: o(self, new_state)                          # Observer: customer app, restaurant app, partner app

class NearestPartner:                                                    # AssignmentStrategy
    def choose(self, partners, pickup_loc):
        free = [p for p in partners if p.available]
        return min(free, key=lambda p: abs(p.loc[0] - pickup_loc[0]) + abs(p.loc[1] - pickup_loc[1]), default=None)

class OrderService:
    def __init__(self, restaurants, partners, payment, assigner=NearestPartner(), delivery_fee=lambda amt: 30):
        self.restaurants, self.partners, self.payment, self.assigner, self.fee = restaurants, partners, payment, assigner, delivery_fee
        self.orders, self.observers, self._ids, self._lock = {}, [], itertools.count(1), threading.Lock()
    def place_order(self, customer, restaurant_id, cart):                # cart: {item_id: qty}
        r = self.restaurants[restaurant_id]
        if not r.open: raise ValueError('restaurant closed')
        if not cart or any(q <= 0 for q in cart.values()): raise ValueError('invalid cart')
        amount = 0
        for item_id, qty in cart.items():
            item = r.menu.get(item_id)
            if item is None or not item.in_stock: raise ValueError(f'unavailable: {item_id}')
            amount += item.price * qty
        amount += self.fee(amount)
        if not self.payment.charge(customer, amount): raise RuntimeError('payment failed')   # no order created if payment fails
        order = Order(next(self._ids), customer, r, dict(cart), amount); self.orders[order.id] = order
        return order
    def assign_partner(self, order):
        with self._lock:                                                  # two orders must not grab the same partner
            p = self.assigner.choose(self.partners, order.restaurant.loc)
            if p is None: return None                                     # none free: keep in a retry queue
            p.available, order.partner = False, p; return p
    def cancel(self, order):
        order.move_to(OrderState.CANCELLED, self.observers)
        self.payment.refund(order.customer, order.amount)
    def deliver(self, order):
        order.move_to(OrderState.DELIVERED, self.observers)
        with self._lock: order.partner.available = True
```

**Edge cases and production concerns:** items from two restaurants in one cart (reject or split); price changes between cart and checkout (re-validate at placement); restaurant rejects or times out (auto-cancel + refund); **no partner available** (queue and retry, widen search radius, surge fee); partner declines (reassign); payment succeeded but order creation failed (**saga/compensating refund**, idempotency key); double-click placing two orders; cancellation after PREPARING (partial refund policy); live location updates (a geospatial index such as a grid or geohash instead of scanning all partners); ETA estimation; coupons and taxes; rating after delivery.

**Tests:** valid cart creates an order and charges once; payment failure creates no order; `PLACED -> PREPARING` raises `InvalidTransition`; assigning two orders with one partner assigns exactly one; cancel before PREPARING refunds; observers receive each transition.

---

## L9. Vending machine

**Requirements:** products in numbered slots with price and quantity; user inserts coins/notes, selects a product; machine dispenses and returns **change**; cancel refunds; handle out-of-stock, insufficient money, cannot make change; admin can refill and collect money.

**Pattern: State.** The machine's behaviour depends on its state, so each state is a class: `Idle` (no money) → `HasMoney` → (dispense) → `Idle`; plus `OutOfService`. This removes nested `if (state == ...)` checks.

```
          insert(money)                select(code), price ok, stock ok
  IDLE ---------------> HAS_MONEY --------------------------------> dispense + change --> IDLE
    ^                       | cancel() -> refund -> IDLE
    +-----------------------+ select with insufficient money / sold out -> error, stay HAS_MONEY
```

```cpp
#include <bits/stdc++.h>
class VendingMachine;
struct State {                                    // default: every action is illegal unless a state overrides it
    virtual ~State() = default;
    virtual void insert(VendingMachine&, int) { throw std::logic_error("not allowed in this state"); }
    virtual void select(VendingMachine&, const std::string&) { throw std::logic_error("not allowed in this state"); }
    virtual int cancel(VendingMachine&) { throw std::logic_error("nothing to cancel"); }
};
struct Product { std::string name; int price; int qty; };
class VendingMachine {
    std::unique_ptr<State> state_;
public:
    std::map<std::string, Product> slots; int balance = 0;
    std::map<int, int> coins = {{1, 20}, {2, 20}, {5, 20}, {10, 20}};            // denomination -> count (for change)
    VendingMachine();                                                              // defined below, once the state classes are complete
    void setState(std::unique_ptr<State> s) { state_ = std::move(s); }
    void insert(int d) { state_->insert(*this, d); }
    void select(const std::string& code) { state_->select(*this, code); }
    int cancel() { return state_->cancel(*this); }
    std::optional<std::map<int, int>> makeChange(int amount) {                     // greedy: correct for canonical coin systems
        std::map<int, int> out; auto stock = coins;
        for (auto it = stock.rbegin(); it != stock.rend() && amount > 0; ++it) {
            int use = std::min(it->second, amount / it->first); if (use) { out[it->first] = use; amount -= use * it->first; }
        }
        if (amount != 0) return std::nullopt;                                      // cannot make exact change
        for (auto& [d, n] : out) coins[d] -= n;
        return out;
    }
};

// Declare BOTH states first (member functions only declared) so they can create each other...
struct IdleState : State {
    void insert(VendingMachine& m, int d) override;                                // switches to HasMoney
};
struct HasMoney : State {
    void insert(VendingMachine& m, int d) override { m.balance += d; m.coins[d]++; }
    void select(VendingMachine& m, const std::string& code) override;              // switches back to IdleState
    int cancel(VendingMachine& m) override;
};

// ...then define the bodies out of line: now every state is a complete type, so make_unique<...> compiles.
void IdleState::insert(VendingMachine& m, int d) {
    m.balance += d; m.coins[d]++;
    m.setState(std::make_unique<HasMoney>());
}
void HasMoney::select(VendingMachine& m, const std::string& code) {
    auto it = m.slots.find(code);
    if (it == m.slots.end()) throw std::invalid_argument("invalid code");
    if (it->second.qty == 0) throw std::runtime_error("sold out");
    if (m.balance < it->second.price) throw std::runtime_error("insufficient money");
    auto change = m.makeChange(m.balance - it->second.price);
    if (!change) throw std::runtime_error("cannot make change: cancel to get a refund");
    it->second.qty--; m.balance = 0; /* dispense it->second; return the change coins */
    m.setState(std::make_unique<IdleState>());
}
int HasMoney::cancel(VendingMachine& m) {
    int refund = m.balance; m.balance = 0;
    m.setState(std::make_unique<IdleState>());
    return refund;
}
VendingMachine::VendingMachine() : state_(std::make_unique<IdleState>()) {}
```

The C++ above is complete: `IdleState` moves to `HasMoney` on the first inserted coin, and `HasMoney` returns to `IdleState` after a sale or a cancel; every other action throws. A runnable Python version of the same design:

```python
class VendingError(Exception): ...
class State:
    def insert(self, m, d): raise VendingError('not allowed now')
    def select(self, m, code): raise VendingError('insert money first')
    def cancel(self, m): raise VendingError('nothing to cancel')
class Idle(State):
    def insert(self, m, d): m.balance += d; m.coins[d] += 1; m.state = HasMoney()
class HasMoney(State):
    def insert(self, m, d): m.balance += d; m.coins[d] += 1
    def select(self, m, code):
        p = m.slots.get(code)
        if p is None: raise VendingError('invalid code')
        if p['qty'] == 0: raise VendingError('sold out')
        if m.balance < p['price']: raise VendingError('insufficient money')
        change = m.make_change(m.balance - p['price'])
        if change is None: raise VendingError('cannot make change: cancel for refund')
        p['qty'] -= 1; m.balance = 0; m.state = Idle()
        return p['name'], change                                   # dispense item and give back the change
    def cancel(self, m):
        refund, m.balance, m.state = m.balance, 0, Idle(); return refund
class VendingMachine:
    def __init__(self, slots, coins):
        self.slots, self.coins, self.balance, self.state = slots, dict(coins), 0, Idle()
    def insert(self, d): self.state.insert(self, d)
    def select(self, code): return self.state.select(self, code)
    def cancel(self): return self.state.cancel(self)
    def make_change(self, amount):
        out = {}
        for d in sorted(self.coins, reverse=True):
            use = min(self.coins[d], amount // d)
            if use: out[d] = use; amount -= use * d
        if amount: return None
        for d, n in out.items(): self.coins[d] -= n
        return out
# Coins are stored when inserted (so they can be used as change). If a purchase fails the coins stay in the machine until cancel(): in a real
# machine you would reserve them separately; mention this as a refinement.
```

**Change-making:** greedy is optimal only for canonical coin systems (1, 2, 5, 10, 20, 50 ...); for arbitrary denominations use DP (Part 1, Coin Change) and a bounded-count variant when coin stock is limited.

**Edge cases:** insufficient money; sold out; invalid code; cannot make exact change (refuse sale or require exact money); inserting coins after a selection is in progress; timeout returning money; coin jam/hardware fault (`OutOfService` state); admin operations (refill, collect cash, change prices) in a separate `Admin` interface; invalid denominations; concurrency is usually single user (a lock is still cheap); persistence of inventory.

**Tests:** insert 20, buy a 15 item, receive 5 as change; insufficient money keeps the balance; cancel refunds all and returns to idle; selecting in idle raises; selling the last unit then selecting again says sold out; machine without small coins refuses a sale that needs unavailable change.

---

## L10. Logging framework

**Requirements:** levels (`DEBUG < INFO < WARN < ERROR < FATAL`); a configurable **minimum level**; several **destinations (appenders)**: console, file (with rotation), remote; pluggable **formatters** (plain text, JSON); named **loggers** in a hierarchy (`app.payments` inherits from `app`); **thread-safe**; optional **async** logging so slow I/O does not block callers; cheap when a level is disabled.

**Entities and patterns:** `Level` (enum), `LogRecord` (immutable: level, logger name, message, timestamp, thread), `Logger`, `Appender` (**Strategy**), `Formatter` (**Strategy**), `LogManager` (registry; a **Singleton**/factory for `get_logger(name)`), `AsyncAppender` (**Decorator** over another appender + producer-consumer queue), appender-level thresholds (**Chain of Responsibility**-like filtering), config via **Builder**. Observer view: appenders are subscribers to log events.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import IntEnum
import json, os, queue, sys, threading, time

class Level(IntEnum): DEBUG = 10; INFO = 20; WARN = 30; ERROR = 40; FATAL = 50

@dataclass(frozen=True)
class LogRecord: level: Level; logger: str; message: str; ts: float; thread: str

class Formatter(ABC):
    @abstractmethod
    def format(self, r: LogRecord) -> str: ...
class TextFormatter(Formatter):
    def format(self, r): return f'{time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(r.ts))} [{r.level.name}] {r.logger} ({r.thread}): {r.message}'
class JsonFormatter(Formatter):
    def format(self, r): return json.dumps({'ts': r.ts, 'level': r.level.name, 'logger': r.logger, 'thread': r.thread, 'msg': r.message})

class Appender(ABC):
    def __init__(self, formatter=None, min_level=Level.DEBUG): self.formatter, self.min_level = formatter or TextFormatter(), min_level
    def append(self, r):
        if r.level >= self.min_level: self._write(self.formatter.format(r))
    @abstractmethod
    def _write(self, line: str): ...
    def close(self): pass
class ConsoleAppender(Appender):
    def __init__(self, **kw): super().__init__(**kw); self._lock = threading.Lock()
    def _write(self, line):
        with self._lock: print(line, file=sys.stderr)                 # lock so concurrent lines never interleave
class FileAppender(Appender):
    def __init__(self, path, max_bytes=5_000_000, **kw):
        super().__init__(**kw); self.path, self.max_bytes, self._lock = path, max_bytes, threading.Lock(); self._f = open(path, 'a')
    def _write(self, line):
        with self._lock:
            if self._f.tell() >= self.max_bytes:                       # size-based rotation
                self._f.close(); os.replace(self.path, self.path + '.1'); self._f = open(self.path, 'a')
            self._f.write(line + '\n'); self._f.flush()
    def close(self): self._f.close()

class AsyncAppender(Appender):                                         # Decorator: wraps any appender
    _STOP = object()
    def __init__(self, delegate, capacity=10000):
        super().__init__(min_level=Level.DEBUG); self.delegate, self.q = delegate, queue.Queue(capacity)
        self.dropped = 0; self.t = threading.Thread(target=self._run, daemon=True); self.t.start()
    def append(self, r):
        try: self.q.put_nowait(r)                                      # never block the caller: drop (and count) when full
        except queue.Full: self.dropped += 1
    def _write(self, line): pass
    def _run(self):
        while (r := self.q.get()) is not self._STOP: self.delegate.append(r)
    def close(self): self.q.put(self._STOP); self.t.join(); self.delegate.close()     # flush everything on shutdown

class Logger:
    def __init__(self, name, parent=None): self.name, self.parent, self.level, self.appenders = name, parent, None, []
    def effective_level(self):
        l = self
        while l is not None and l.level is None: l = l.parent        # inherit from the nearest configured ancestor
        return l.level if l else Level.INFO
    def _appenders(self):
        l = self
        while l is not None: yield from l.appenders; l = l.parent    # additivity: events also go to ancestors' appenders
    def log(self, level, msg, *args):
        if level < self.effective_level(): return                     # cheap exit BEFORE any formatting work
        r = LogRecord(level, self.name, msg % args if args else msg, time.time(), threading.current_thread().name)
        for a in self._appenders():
            try: a.append(r)
            except Exception as e: print(f'logging failure: {e}', file=sys.stderr)   # logging must never crash the app
    def debug(self, m, *a): self.log(Level.DEBUG, m, *a)
    def info(self, m, *a): self.log(Level.INFO, m, *a)
    def error(self, m, *a): self.log(Level.ERROR, m, *a)

class LogManager:
    _loggers, _lock = {'': Logger('')}, threading.Lock()
    @classmethod
    def get_logger(cls, name=''):
        with cls._lock:
            if name not in cls._loggers:
                parent = cls.get_logger(name.rpartition('.')[0]) if name else None      # 'a.b' -> parent 'a' -> root ''
                cls._loggers[name] = Logger(name, parent)
            return cls._loggers[name]

root = LogManager.get_logger(); root.level = Level.INFO; root.appenders.append(ConsoleAppender())
log = LogManager.get_logger('app.payments'); log.info('charged %s for order %d', 'asha', 42)
```

```cpp
enum class Level { Debug = 10, Info = 20, Warn = 30, Error = 40, Fatal = 50 };
struct Appender { virtual ~Appender() = default; virtual void write(const std::string& line) = 0; };
class ConsoleAppender : public Appender {
    std::mutex m_;
public:
    void write(const std::string& line) override { std::lock_guard<std::mutex> g(m_); std::cerr << line << '\n'; }
};
class Logger {
    std::string name_; std::atomic<int> level_{static_cast<int>(Level::Info)};
    std::vector<std::shared_ptr<Appender>> appenders_; std::mutex m_;
public:
    explicit Logger(std::string n) : name_(std::move(n)) {}
    void setLevel(Level l) { level_ = static_cast<int>(l); }
    void addAppender(std::shared_ptr<Appender> a) { std::lock_guard<std::mutex> g(m_); appenders_.push_back(std::move(a)); }
    void log(Level l, const std::string& msg) {
        if (static_cast<int>(l) < level_.load()) return;                      // fast path without locking
        std::vector<std::shared_ptr<Appender>> snapshot; { std::lock_guard<std::mutex> g(m_); snapshot = appenders_; }
        std::ostringstream os; os << '[' << static_cast<int>(l) << "] " << name_ << ": " << msg;
        for (auto& a : snapshot) a->write(os.str());
    }
};
#define LOG_INFO(logger, msg) (logger).log(Level::Info, std::string(__FILE__) + ":" + std::to_string(__LINE__) + " " + (msg))
```

**Complexity:** a disabled level costs one integer comparison; an enabled call is O(appenders); async mode makes the call O(1) (enqueue).

**Edge cases and production concerns:** **never throw** from logging; avoid building expensive messages when the level is off (lazy `%` args or lambdas); interleaved output from threads (lock per appender or one writer thread); **queue overflow policy** for async (block, drop oldest, drop newest, count drops); flush on shutdown and on crash; rotation by size or time and retention; **context** (request id, user id via thread-local / MDC) added to every record; sensitive data (**never log passwords, tokens, card numbers**; mask PII); log injection (escape newlines); clock/timezone (UTC); the logging framework must not depend on itself (no recursive logging); changing levels at runtime.

**Tests:** a DEBUG message is dropped when the level is INFO; a child logger inherits its parent's level and appenders; two appenders both receive an event; an appender that raises does not stop the others; async `close()` flushes all queued records; 1000 concurrent threads produce 1000 intact lines.

---

## L11. Notification framework

**Requirements:** send notifications through **multiple channels** (email, SMS, push, in-app); event-driven ("order shipped") using **templates**; respect **user preferences** (opted-in channels, quiet hours); **priority** (OTP before marketing); **retry with backoff** on failure and a **fallback channel**; **asynchronous** delivery; **deduplication/idempotency**; delivery **status tracking**; easy to add a new channel.

**Entities and patterns:** `Channel` interface (**Strategy/Factory**: `EmailChannel`, `SmsChannel`, `PushChannel`), `Template` (**Template Method**/formatter), `PreferenceService`, `Notification`/`Delivery` (status), `NotificationService` (**Facade**), a queue + workers (producer-consumer, **Command**), `RetryPolicy`, event subscription (**Observer**: services publish events, the notifier subscribes), optional **Decorator** wrappers (rate limiting, logging) around a channel.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import IntEnum
import itertools, queue, threading, time

class Priority(IntEnum): LOW = 1; NORMAL = 2; HIGH = 3                 # HIGH examples: OTP, security alerts
class Status: PENDING, SENT, FAILED, SKIPPED = 'PENDING', 'SENT', 'FAILED', 'SKIPPED'

class Channel(ABC):
    name: str
    @abstractmethod
    def send(self, user, subject, body) -> bool: ...                   # returns False/raises on failure
class EmailChannel(Channel):
    name = 'email'
    def send(self, user, subject, body): print(f'EMAIL to {user["email"]}: {subject}'); return True
class SmsChannel(Channel):
    name = 'sms'
    def send(self, user, subject, body): print(f'SMS to {user["phone"]}: {body}'); return True
class PushChannel(Channel):
    name = 'push'
    def send(self, user, subject, body): print(f'PUSH to {user["id"]}: {subject}'); return True

class TemplateStore:
    def __init__(self): self.t = {}
    def register(self, event, channel, subject, body): self.t[(event, channel)] = (subject, body)
    def render(self, event, channel, data):
        subject, body = self.t[(event, channel)]
        return subject.format_map(data), body.format_map(data)         # KeyError for a missing variable: fail loudly

class Preferences:
    def __init__(self): self.prefs = {}                                  # user_id -> {event: set(channels)}, quiet hours
    def channels(self, user_id, event, priority, hour):
        p = self.prefs.get(user_id, {'channels': {event: {'email'}}, 'quiet': (22, 7)})
        chans = set(p['channels'].get(event, ()))
        start, end = p['quiet']; quiet = hour >= start or hour < end
        if quiet and priority < Priority.HIGH: chans.discard('sms'); chans.discard('push')   # only urgent alerts disturb quiet hours
        return chans

@dataclass
class Delivery: id: int; user: dict; event: str; channel: str; subject: str; body: str; priority: Priority; attempts: int = 0; status: str = Status.PENDING

class NotificationService:
    def __init__(self, channels, templates, prefs, max_attempts=3, base_delay=1.0, workers=2, clock_hour=lambda: time.localtime().tm_hour):
        self.channels = {c.name: c for c in channels}; self.templates, self.prefs = templates, prefs
        self.max_attempts, self.base_delay, self.hour = max_attempts, base_delay, clock_hour
        self.q, self._seq, self.seen, self.status, self.dead = queue.PriorityQueue(), itertools.count(), set(), {}, []
        self._lock = threading.Lock()
        for _ in range(workers): threading.Thread(target=self._worker, daemon=True).start()
    def notify(self, user, event, data, priority=Priority.NORMAL, key=None):
        if key is not None:                                              # idempotency: the same business event is sent only once
            with self._lock:
                if (user['id'], key) in self.seen: return []
                self.seen.add((user['id'], key))
        ids = []
        for ch in self.prefs.channels(user['id'], event, priority, self.hour()):
            subject, body = self.templates.render(event, ch, data)
            d = Delivery(next(self._seq), user, event, ch, subject, body, priority)
            self.status[d.id] = Status.PENDING; self.q.put((-priority, d.id, d)); ids.append(d.id)    # higher priority first
        return ids
    def _worker(self):
        while True:
            _, _, d = self.q.get()
            try:
                ok = self.channels[d.channel].send(d.user, d.subject, d.body)
            except Exception:
                ok = False
            d.attempts += 1
            if ok: self.status[d.id] = Status.SENT
            elif d.attempts < self.max_attempts:                         # exponential backoff, then re-enqueue
                threading.Timer(self.base_delay * 2 ** (d.attempts - 1), lambda d=d: self.q.put((-d.priority, d.id, d))).start()
            else: self.status[d.id] = Status.FAILED; self.dead.append(d)   # dead-letter list for inspection/replay
```

```cpp
struct Channel { virtual ~Channel() = default; virtual bool send(const std::string& user, const std::string& subject, const std::string& body) = 0; };
class NotificationService {
    std::map<std::string, std::unique_ptr<Channel>> channels_; int maxAttempts_ = 3;
public:
    void registerChannel(const std::string& name, std::unique_ptr<Channel> c) { channels_[name] = std::move(c); }
    bool deliver(const std::string& channel, const std::string& user, const std::string& subject, const std::string& body) {
        auto it = channels_.find(channel);
        if (it == channels_.end()) throw std::invalid_argument("unknown channel");
        for (int attempt = 0; attempt < maxAttempts_; ++attempt) {
            try { if (it->second->send(user, subject, body)) return true; } catch (const std::exception&) {}
            std::this_thread::sleep_for(std::chrono::milliseconds(100 << attempt));      // exponential backoff
        }
        return false;                                                                   // caller moves it to a dead-letter queue
    }
};
```

**Design notes:** the service is **decoupled from the producers**: order code just publishes `order_shipped`; subscribing to events (Observer / message broker) means no `if` per feature. Add a new channel by writing one class and registering it (OCP). In production the in-memory queue becomes a durable broker (Kafka/SQS/RabbitMQ) so notifications survive restarts, with retry topics and a **dead-letter queue**.

**Edge cases:** channel outage (retry, then **fallback** to another channel such as SMS if push fails); duplicate events (idempotency key); retries must not duplicate a message already delivered (provider idempotency); **rate limiting** per user and per provider; quiet hours and time zones; unsubscribed/opt-out and legal rules (marketing vs transactional); template variable missing or HTML injection (escape); localisation; attachments and size limits; ordering (a later "cancelled" must not arrive before "confirmed"); batching/digests; delivery receipts and webhooks updating the status; sensitive data in logs.

**Tests:** the event with preferences \[email\] sends exactly one email; an OTP (HIGH) goes to SMS during quiet hours while a promo (LOW) does not; the same idempotency key twice sends once; a channel failing twice then succeeding ends as SENT after 3 attempts; permanent failure lands in the dead-letter list; a missing template variable raises.

---

## L12. Task scheduler

**Requirements:** run a task **after a delay**, at a **time**, or **repeatedly** (fixed rate or fixed delay); support **priority** among tasks due at the same time; **cancel** tasks; execute with a **thread pool** so one slow task does not delay others; thread-safe; a failing task must not kill the scheduler; **graceful shutdown**.

**Core design:** a **min-heap (priority queue) ordered by next run time** (tie-break by priority, then insertion sequence for FIFO fairness), a single **scheduler thread** that sleeps until the earliest task is due and a **worker pool** that runs tasks. The scheduler waits on a **condition variable with a timeout**; adding an earlier task or shutting down **notifies** it so it recomputes its wait.

```
schedule(fn, delay) --> [heap by (run_at, -priority, seq)] --> scheduler thread: wait until top is due
                                                              pop -> submit to worker pool -> (if periodic) re-insert
```

```python
import heapq, itertools, threading, time, traceback
from concurrent.futures import ThreadPoolExecutor

class ScheduledTask:
    def __init__(self, fn, run_at, seq, priority=0, period=None, fixed_rate=True):
        self.fn, self.run_at, self.seq, self.priority, self.period, self.fixed_rate = fn, run_at, seq, priority, period, fixed_rate
        self.cancelled = False
    def __lt__(self, o): return (self.run_at, -self.priority, self.seq) < (o.run_at, -o.priority, o.seq)
    def cancel(self): self.cancelled = True                              # lazy removal: skipped when it reaches the top

class Scheduler:
    def __init__(self, workers=4, clock=time.monotonic):                 # monotonic clock: immune to system time changes
        self.clock, self._heap, self._seq = clock, [], itertools.count()
        self._cv, self._stopped = threading.Condition(), False
        self._pool = ThreadPoolExecutor(max_workers=workers)
        self._thread = threading.Thread(target=self._loop, name='scheduler', daemon=True); self._thread.start()
    def _add(self, task):
        with self._cv:
            if self._stopped: raise RuntimeError('scheduler is shut down')
            heapq.heappush(self._heap, task)
            self._cv.notify()                                            # wake the loop: this task may be due sooner than what it waits for
        return task
    def schedule(self, fn, delay=0.0, priority=0):
        return self._add(ScheduledTask(fn, self.clock() + delay, next(self._seq), priority))
    def schedule_at_fixed_rate(self, fn, initial_delay, period, priority=0):          # run times: t0, t0+p, t0+2p ... (no drift)
        return self._add(ScheduledTask(fn, self.clock() + initial_delay, next(self._seq), priority, period, True))
    def schedule_with_fixed_delay(self, fn, initial_delay, delay, priority=0):        # next run = completion time + delay
        return self._add(ScheduledTask(fn, self.clock() + initial_delay, next(self._seq), priority, delay, False))
    def _loop(self):
        with self._cv:
            while not self._stopped:
                if not self._heap: self._cv.wait(); continue
                task = self._heap[0]
                if task.cancelled: heapq.heappop(self._heap); continue
                wait = task.run_at - self.clock()
                if wait > 0: self._cv.wait(timeout=wait); continue        # re-check: a new earlier task or shutdown may have arrived
                heapq.heappop(self._heap)
                if task.period and task.fixed_rate:                       # re-arm BEFORE running so the period is exact
                    task.run_at += task.period; task.seq = next(self._seq); heapq.heappush(self._heap, task)
                self._pool.submit(self._run, task)
    def _run(self, task):
        try:
            if not task.cancelled: task.fn()
        except Exception:
            traceback.print_exc()                                         # a failing task never kills the scheduler
        if task.period and not task.fixed_rate and not task.cancelled and not self._stopped:
            task.run_at = self.clock() + task.period; task.seq = next(self._seq); self._add(task)
    def shutdown(self, wait=True):
        with self._cv: self._stopped = True; self._cv.notify_all()
        self._pool.shutdown(wait=wait)
```

```cpp
class Scheduler {
    using Clock = std::chrono::steady_clock;
    struct Task {
        Clock::time_point at; int priority; long seq; std::function<void()> fn;
        bool operator<(const Task& o) const { return std::tie(at, o.priority, seq) > std::tie(o.at, priority, o.seq); }   // reversed: min-heap
    };
    std::priority_queue<Task> heap_; std::mutex m_; std::condition_variable cv_; bool stop_ = false; long seq_ = 0;
    std::thread timer_; std::vector<std::thread> workers_; std::queue<std::function<void()>> ready_; std::condition_variable wcv_;
    void timerLoop() {
        std::unique_lock<std::mutex> lk(m_);
        while (!stop_) {
            if (heap_.empty()) { cv_.wait(lk); continue; }
            auto next = heap_.top().at;
            if (Clock::now() < next) { cv_.wait_until(lk, next); continue; }                 // wakes early on notify; loop re-checks
            Task t = heap_.top(); heap_.pop();
            ready_.push(std::move(t.fn)); wcv_.notify_one();                                 // hand over to a worker
        }
    }
    void workerLoop() {
        for (;;) {
            std::function<void()> job;
            { std::unique_lock<std::mutex> lk(m_); wcv_.wait(lk, [&] { return stop_ || !ready_.empty(); });
              if (ready_.empty()) return; job = std::move(ready_.front()); ready_.pop(); }
            try { job(); } catch (...) { /* log and continue */ }
        }
    }
public:
    explicit Scheduler(int n = 4) { timer_ = std::thread([this] { timerLoop(); }); for (int i = 0; i < n; i++) workers_.emplace_back([this] { workerLoop(); }); }
    void schedule(std::function<void()> fn, std::chrono::milliseconds delay, int priority = 0) {
        { std::lock_guard<std::mutex> g(m_); heap_.push({Clock::now() + delay, priority, seq_++, std::move(fn)}); }
        cv_.notify_one();
    }
    ~Scheduler() { { std::lock_guard<std::mutex> g(m_); stop_ = true; } cv_.notify_all(); wcv_.notify_all(); timer_.join(); for (auto& w : workers_) w.join(); }
};
```

**Complexity:** `schedule` O(log n); the scheduler pops the next task in O(log n); cancel is O(1) (lazy deletion); cancelled tasks leave heap entries until they surface (purge periodically for very large heaps).

**Edge cases and concurrency pitfalls:** **spurious wakeups** and the wait/notify race (always re-check the condition in a loop with the lock held); a new earlier task must wake the scheduler; a **long task** would block timing if run on the scheduler thread (hence the pool); **fixed rate vs fixed delay** (a slow task makes fixed-rate tasks pile up: decide to skip, coalesce or run catch-up); task exceptions must be caught (and periodic tasks should keep going); cancellation of a task that is **already running** (cooperative flag, cannot kill a thread); shutdown semantics (`wait` for running tasks, drop or run pending ones); clock changes (use a monotonic clock); delay of zero or negative; many tasks at the same instant (priority then FIFO); memory growth from unbounded periodic tasks; persistence/recovery for durable schedules; cron expressions (compute the next fire time with a parser and re-insert); distributed schedulers need leader election or a distributed lock to avoid duplicate runs.

**Tests (inject a fake clock or use short real delays):** a task with delay 50 ms runs after about 50 ms and not before; two tasks run in due-time order; same time and different priority runs the higher priority first; a periodic task runs N times in N x period; a cancelled task never runs; a task that raises does not stop later tasks; a task added with an earlier time while the loop is sleeping runs on time; after `shutdown` new schedules raise.

---

## L13. LLD revision sheet

| Problem | Patterns to name | The one tricky part to talk about |
| --- | --- | --- |
| Parking Lot | Strategy (fee, allocation), Factory | O(log n) nearest-slot allocation and one critical section for park/exit |
| Splitwise | Strategy/Factory (split types) | integer money, remainder distribution, **debt simplification** with two heaps |
| LRU Cache | (data-structure design) | hash map + doubly linked list; `get` mutates, so locking; update refreshes recency |
| TTL Cache | Strategy (eviction), heap | lazy + active expiry, stale heap entries, monotonic/injected clock |
| Elevator | State, Strategy (dispatch), Command | LOOK scheduling, direction rules, dispatcher cost function |
| Library | Strategy (fine), Observer | **Book vs BookCopy**, reservation queue and held copies |
| Movie Booking | State (seat), Facade | **seat lock with expiry**, all-or-nothing, DB unique constraint, payment failure rollback |
| Food Delivery | State (order), Strategy (assignment), Observer | order state machine, partner assignment race, saga/refund |
| Vending Machine | **State** | state transitions, change-making, cannot-make-change case |
| Logging Framework | Strategy, Decorator (async), Singleton/Factory | level check before formatting, non-blocking async queue, logger hierarchy |
| Notification Framework | Strategy/Factory, Observer, Decorator | preferences, retry with backoff, idempotency, dead-letter queue |
| Task Scheduler | Command, producer-consumer | min-heap + condition-variable timed wait, notify on earlier task, thread pool |

**Follow-up questions interviewers ask, and the shape of a good answer**

1. **"How would you make it thread-safe?"** Identify the shared state and the invariant; one lock per aggregate (per show, per lot, per cache shard); make check-then-act atomic; avoid holding locks during I/O; consistent lock order.
2. **"How does it scale to many servers?"** Move shared state to a database/Redis; use unique constraints, row locks, conditional updates, or distributed locks with TTL; make operations idempotent; partition by key (show, city, user).
3. **"How do you add a new X without changing existing code?"** Show the interface/strategy/registry and add a new class (OCP).
4. **"What if a step fails halfway?"** Roll back or compensate (release locks, refund, retry, dead-letter); idempotency keys for retries.
5. **"How would you test it?"** Inject clock/ID generator/gateway; fakes over mocks where possible; boundary and concurrency tests.
6. **"What would you change with 10x load?"** Cache, shard, async queues, batching, read replicas; say what you would measure first.
7. **"Why this pattern?"** State the problem it solves and its cost; do not force a pattern.

**Final LLD checklist:** requirements and out-of-scope stated · entities with one-line responsibilities · relationships and multiplicities · interfaces where behaviour varies · money as integers, injected clock · validation and custom exceptions · core flow coded end to end · concurrency and failure handling discussed · 5 to 8 test cases named · extensions listed.

---

*End of Part 4. Part 5 covers system design (HLD): scalability, caching, load balancing, databases and sharding, messaging, CAP, and worked designs such as URL shortener, rate limiter, chat, news feed and notification system.*
