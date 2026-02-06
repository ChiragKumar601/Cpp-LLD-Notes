### What creational patterns solve (Intuition)
Creation is where designs often break because:
- constructors get huge (too many params)
- object choice depends on runtime config/type
- ownership/lifetime becomes unclear (especially in C++)

Creational patterns give you:
- **controlled construction**
- **clean ownership**
- **decoupled callers from concrete types (DIP + OCP)**

# 5.1 Singleton (Use rarely — but you must master it)

## Intuition
Singleton enforces “exactly one instance globally”.
It’s famous, but in real engineering it’s often a **global variable in disguise**.

**Use-case category where it can be justified:**
- process-wide immutable config snapshot
- logger sink (sometimes)
- metrics registry (sometimes)
- thread pool (sometimes)

But even there, prefer DI if possible.

---

## Concepts (Rules + pitfalls you must know)

### ✅ The only safe-ish modern C++ form (Meyers Singleton)

```cpp
class Logger {
public:
    static Logger& instance() {
        static Logger inst;  // thread-safe since C++11
        return inst;
    }

    void log(const std::string& msg) { /*...*/ }

private:
    Logger() = default;                    // hide construction
    ~Logger() = default;
    Logger(const Logger&) = delete;        // prevent copy
    Logger& operator=(const Logger&) = delete;
};
```
### Key rules (interview-critical)
#### Rule 1 : Thread safety: local static init is thread-safe in C++11+
If two threads call `Logger::instance()` at the same time, the language guarantees:
- only **one** `Logger` is constructed
- other threads wait until initialization finishes
**Why interviewers care:** pre-C++11 this could race; now it’s guaranteed.

#### Rule 2 : Copy/move must be deleted to avoid duplicates
**Meaning:** Even if construction is private, you could still accidentally create “another logger” by copying or moving, e.g.:```
```cpp
Logger a = Logger::instance(); // would copy if allowed
```
Logger a = Logger::instance(); // would copy if allowed

#### Rule 3: Destructor order problem: global/static destruction order across translation units is tricky

**Meaning:** If you have multiple global/static objects in different `.cpp` files, the order they are destroyed at program shutdown is **not reliably controllable**.

So this can happen:
- Some global object’s destructor runs
- It tries to log
- But `Logger` has already been destroyed  
    → crash/UB or lost logs.

This is the classic **“static initialization/destruction order fiasco.”**

**Mitigations:**
- avoid logging in global destructors
- keep singleton alive until process exit intentionally (leak-on-purpose)
- or use proper lifecycle management in `main()`

#### Rule 4: Hidden dependencies: any code can call `Logger::instance()` → hard to test

**Meaning:** With a singleton, dependencies are **not visible in constructor signatures**.

So a class looks like it has no dependencies:    
```cpp
class OrderService {
public:
    void placeOrder() {
        Logger::instance().log("placed");
    }
};
```
But it secretly depends on Logger.

**Testing pain:**
- can’t easily replace logger with a fake/mock
- tests influence each other (global state)
- you need hacks to redirect output

DIP-friendly alternative:
- inject `ILogger&` into `OrderService`

#### Rule 5: Singleton is global mutable state unless immutable

**Meaning:** If your singleton has mutable state (log level, output file, buffers), then any part of the program can change it:
```cpp
Logger::instance().setLevel(DEBUG);
```

That creates:
- unpredictable behavior (“who changed log level?”    
- race conditions if not synchronized
- order-dependent bugs and flaky tests

**Rule of thumb:**
- global mutable state makes reasoning hard
- if you must use a singleton, keep it **mostly immutable** or carefully synchronized


---

## Architecture View
```cpp
AnyClass ---> Logger::instance()  (global access)
```
This is why coupling increases: dependencies are invisible in constructors.

---

## Practical Example

### Scenario: basic logging
```cpp
Logger::instance().log("payment started");
```

---

## Real-world use cases (and safer alternatives)

### Acceptable-ish

- read-only config:
    - `Config::instance()` returns immutable config
    - **Idea:** You load configuration once at startup, then expose it globally as immutable data.

- registry-like singletons:
    - metrics registry (but often injected)

### Prefer instead (most interviewers like this)
- **DI + composition root**
```cpp
class Service {
    ILogger& logger;
public:
    Service(ILogger& lg) : logger(lg) {}
};
```

This gives testability + visible dependencies.

---

## Interview-level Questions (Singleton)

**Q. Why is Singleton considered an anti-pattern in many codebases?**
Because it introduces hidden global mutable state, increasing coupling and making testing, reasoning, and lifecycle management (init/shutdown) hard.

**Q. How is Meyers Singleton thread-safe?**
 Meyers Singleton is thread-safe in **C++11+** because **function-local static initialization is guaranteed to be thread-safe**: the runtime ensures the static object is constructed **exactly once**, even if multiple threads call `instance()` concurrently. You still typically **delete copy/move** to prevent duplication, but the thread-safety comes from the language guarantee around initialization of the local static.
 
**Q. What is “static initialization order fiasco”?**
The static initialization order fiasco is the problem where **non-local static objects across different translation units** (different `.cpp` files) have an **unspecified initialization order**. If one static depends on another static in a different file, it may run before the dependency is constructed, leading to undefined behavior. A related issue happens at program shutdown: **destruction order is the reverse of construction**, and that order is also problematic across translation units.

**Q. When would you still accept Singleton? What constraints would you enforce?**
Only for things that are effectively “global and read-only” (like config) or a simple shared registry (like metrics), and I’d keep it immutable, thread-safe, initialized once at startup, and avoid using it for business state.

**Q. How would you refactor Singleton out of a design to improve testability?**
Stop calling `X::instance()` directly—pass the dependency into classes (constructor injection) via an interface, wire it in `main()` (composition root), and in tests inject a fake/mock instead.


---

# 5.2 Factory Method (Create without coupling to concrete types)

## Intuition

Callers want an object, but shouldn’t know **which concrete class** to create.

Factory Method is used when:

- type depends on runtime input
- you want OCP for adding new types
- you want construction logic centralized

---

## Concepts

- Expose an interface (or base class)
- Factory method returns base pointer (`unique_ptr` usually)

### C++ ownership rule

- Factories usually return `std::unique_ptr<Base>` (caller owns it)

---

## Architecture View
```
Client --> Factory --> Base*
                \--> DerivedA
                \--> DerivedB

```

---

## Practical Example (Payment methods)
```cpp
class IPaymentMethod {
public:
    virtual ~IPaymentMethod() = default;
    virtual void pay(double amount) = 0;
};

class CardPayment : public IPaymentMethod { /*...*/ };
class UpiPayment  : public IPaymentMethod { /*...*/ };

class PaymentFactory {
public:
    static std::unique_ptr<IPaymentMethod> create(const std::string& type) {
        if (type == "CARD") return std::make_unique<CardPayment>();
        if (type == "UPI")  return std::make_unique<UpiPayment>();
        throw std::invalid_argument("Unknown payment type");
    }
};
```

### Growth path (interview-smart)

- Start with `if` inside factory (only one place changes)
- When types explode → replace with registry map (Module 5.7 style)

---

## Real-world use cases

- Parser factory: CSV/JSON/XML
- Notification sender: Email/SMS/WhatsApp
- DB connector: MySQL/Postgres/InMemory

---

## Interview Questions (Factory Method)

**Q. Why does Factory help OCP?**
A factory lets you **add new concrete types by extending the factory/wiring** without changing the business code that uses the interface—so consumers stay closed to modification, open to extension.

**Q. Why should factory return `unique_ptr` vs raw pointer?**
`unique_ptr` makes **ownership explicit** and guarantees **automatic cleanup** (no leaks, no ambiguous `delete` responsibility), while still allowing polymorphism (`unique_ptr<Base>`).
Raw pointers don’t encode ownership, so returning them from factories leads to leaks and lifetime bugs; `unique_ptr` makes ownership explicit and safe by construction.

**Q. Where do you keep the factory? (composition root vs scattered)**
Keep factories in the **composition root/bootstrapping layer** so object creation and wiring are **centralized**, dependencies stay **explicit**, and business logic doesn’t get polluted with construction decisions; avoid scattering factories across the codebase because it increases coupling and makes changes/config harder.

**Q. How do you avoid the factory itself becoming a giant switch?**
Start with `switch` for a few types; as variants grow, use **registration (key → creator)** or **split into smaller factories**, so new types are added by **registering**, not editing a giant conditional.

## 5.3 Abstract Factory (Create _families_ of related objects)

## Intuition

Factory Method picks **one product** type.  
Abstract Factory picks an entire **compatible family** of products.

Use it when:
- you must create multiple related objects that must “match”
- you want to switch the whole family by config/environment

Example families:
- **AWS vs GCP** cloud clients
- **Dark vs Light** UI components
- **Prod vs Test** infrastructure implementations

---

### Concepts
- Define interfaces for products: `IQueue`, `IBlobStore`, `IMetrics`
- Define a factory interface that creates each product
- Concrete factories produce a consistent set

---

### Architecture View

```
             +----------------------+
Client ----> |  IInfraFactory       |
             |  createQueue()       |
             |  createBlobStore()   |
             +----------^-----------+
                        |
        +---------------+----------------+
        |                                |
+-------------------+          +-------------------+
| AwsInfraFactory   |          | GcpInfraFactory   |
+-------------------+          +-------------------+
  | createQueue()->AwsQueue       | createQueue()->GcpQueue
  | createBlob()->S3BlobStore     | createBlob()->GcsBlobStore

```


---

### Practical Example (C++)

```cpp
#include <memory>
#include <string>

// ---------- Products ----------
class IQueue {
public:
    virtual ~IQueue() = default;
    virtual void push(const std::string& msg) = 0;
};

class IBlobStore {
public:
    virtual ~IBlobStore() = default;
    virtual void put(const std::string& key) = 0;
};

// ---------- Abstract Factory ----------
class IInfraFactory {
public:
    virtual ~IInfraFactory() = default;
    virtual std::unique_ptr<IQueue> createQueue() = 0;
    virtual std::unique_ptr<IBlobStore> createBlobStore() = 0;
};

// ---------- AWS family ----------
class AwsQueue : public IQueue {
public:
    void push(const std::string& msg) override {
        // ...
    }
};

class S3BlobStore : public IBlobStore {
public:
    void put(const std::string& key) override {
        // ...
    }
};

class AwsInfraFactory : public IInfraFactory {
public:
    std::unique_ptr<IQueue> createQueue() override {
        return std::make_unique<AwsQueue>();
    }
    std::unique_ptr<IBlobStore> createBlobStore() override {
        return std::make_unique<S3BlobStore>();
    }
};

// ---------- GCP family ----------
class GcpQueue : public IQueue {
public:
    void push(const std::string& msg) override {
        // ...
    }
};

class GcsBlobStore : public IBlobStore {
public:
    void put(const std::string& key) override {
        // ...
    }
};

class GcpInfraFactory : public IInfraFactory {
public:
    std::unique_ptr<IQueue> createQueue() override {
        return std::make_unique<GcpQueue>();
    }
    std::unique_ptr<IBlobStore> createBlobStore() override {
        return std::make_unique<GcsBlobStore>();
    }
};

// ---------- Client (class-based usage) ----------
class UploaderService {
    std::unique_ptr<IInfraFactory> factory_;  // store the factory as a dependency
    std::unique_ptr<IQueue> queue_;
    std::unique_ptr<IBlobStore> store_;

public:
    explicit UploaderService(std::unique_ptr<IInfraFactory> factory)
        : factory_(std::move(factory)) {
        // create a consistent "family" of related objects
        queue_ = factory_->createQueue();
        store_ = factory_->createBlobStore();
    }

    void upload(const std::string& key) {
        store_->put(key);
        queue_->push(key);
    }
};

// ---------- Example wiring (composition root idea) ----------
// int main() {
//     std::unique_ptr<IInfraFactory> factory = std::make_unique<AwsInfraFactory>(); // or GcpInfraFactory
//     UploaderService svc(std::move(factory));
//     svc.upload("file-123");
// }
```

---

### Real-world use cases

- Switching infra providers without touching business logic
- Test vs prod infra:
    - `TestInfraFactory` returns in-memory fakes
- Multi-tenant customization (per-tenant family selection)

---

### Interview Questions (Abstract Factory)

**Q. When is Factory Method not enough?**
When you need to create **multiple related objects that must stay compatible as a set** (a “family”), Factory Method becomes insufficient—**Abstract Factory** lets you switch the entire family (e.g., Queue + BlobStore) together.

**Q. What’s the trade-off? (more interfaces/classes)**
Abstract Factory adds **more interfaces/classes and indirection**, which can increase boilerplate and onboarding complexity—but it buys you **clean provider switching**, consistency across related objects, and better decoupling.

**Q. How do you keep factories from becoming god objects?**
Keep factories **small and cohesive**: one factory per product-family/bounded context, avoid putting business logic inside, and split into sub-factories/modules when the product set grows.

**Q. How to test services using Abstract Factory? (inject test factory)**
Inject a `TestInfraFactory` that returns **in-memory fakes/mocks**, so the service runs unchanged while tests control behavior and assertions.


---


## 5.4 Builder (Construct complex objects safely)

### Intuition

When constructors become unreadable or unsafe:

- too many parameters
- optional fields
- invariants must be enforced
- you want fluent, readable construction

Builder helps create objects **step-by-step**, while ensuring validity at `build()`.


---

### Concepts

Two common builder styles:

#### A) Fluent builder (most common in interviews)
- chain methods
- build final object

#### B) Step builder (compile-time enforcement of required fields)
- stronger but more complex; mention as advanced

---

### Architecture View

```rust
Builder collects pieces -> validates -> builds final object
```

---

### Practical Example (Fluent Builder)
```cpp
struct HttpRequest {
    std::string url;
    std::string method;
    std::unordered_map<std::string, std::string> headers;
    std::string body;
};

class HttpRequestBuilder {
    HttpRequest req;
public:
    HttpRequestBuilder& url(std::string u) { req.url = std::move(u); return *this; }
    HttpRequestBuilder& method(std::string m) { req.method = std::move(m); return *this; }
    HttpRequestBuilder& header(std::string k, std::string v) {
        req.headers.emplace(std::move(k), std::move(v)); return *this;
    }
    HttpRequestBuilder& body(std::string b) { req.body = std::move(b); return *this; }

    HttpRequest build() {
        if (req.url.empty()) throw std::invalid_argument("url required");
        if (req.method.empty()) req.method = "GET";
        return req;
    }
};
```

Usage:
```cpp
auto req = HttpRequestBuilder()
            .url("https://api.x.com")
            .method("POST")
            .header("Auth", "token")
            .body("{...}")
            .build();
```

- **Use `struct`** for _plain data / value objects_ where members are naturally public and there’s little/no invariant logic.
- **Use `class`** when you need _encapsulation + invariants_ (private state, controlled mutation, validation).

---

### Real-world use cases

- Creating complex domain objects (Order, Invoice, Config)
- Creating DTOs for APIs
- Test data builders (very common in real codebases)

---

### Interview Questions (Builder)

**Q. When do you prefer Builder over telescoping constructors?**
_Prefer Builder when there are many optional parameters, readable named setup matters, or you want to avoid constructor overload explosion and parameter-order bugs.

**Q. How does Builder enforce invariants?**
 Builder collects inputs step-by-step and performs validation in `build()` (and/or per-step), so the final object is created only in a valid state.
 
**Q. What’s the downside? (more code, extra type)**
More boilerplate and an extra type/allocations; overkill for small objects, and it can slow onboarding if used everywhere.
 
**Q. What’s a step-builder? Why might you use it?**
A step-builder is a typed builder that **forces required fields in a fixed order at compile time**, preventing `build()` until mandatory steps are completed—use it for complex invariants or “must-set” fields.



---

## 5.5 Prototype (Clone existing objects efficiently)

### Intuition

Prototype is used when:
- object creation is expensive/complex
- you want to create new objects by **cloning a template**
- you need copies with slight modifications

Think:
- default configuration templates
- pre-built rule sets
- game entities (enemy templates)
- UI component style templates


---

### Concepts (C++ specifics)

- Provide a `clone()` method returning a new object
- Prefer returning `std::unique_ptr<Base>`
- Must define deep copy semantics correctly

---

### Architecture View
```scss
Prototype (template) --clone--> New instance
```


---

### Practical Example (C++)
```cpp
class Document {
public:
    virtual ~Document() = default;
    virtual std::unique_ptr<Document> clone() const = 0;
    virtual void setTitle(std::string) = 0;
};

class PdfDocument : public Document {
    std::string title;
public:
    PdfDocument(std::string t) : title(std::move(t)) {}

    std::unique_ptr<Document> clone() const override {
        return std::make_unique<PdfDocument>(*this); // uses copy ctor
    }

    void setTitle(std::string t) override { title = std::move(t); }
};
```

Usage:

```cpp
PdfDocument templateDoc("Default Title");
auto d1 = templateDoc.clone();
d1->setTitle("Invoice #123");
```

---

### Real-world use cases

- Cloning preconfigured objects from config
- Object creation where constructor needs many dependencies
- Game engines: clone archetypes
- UI: duplicate widgets with style

---

### Interview Questions (Prototype)

**Q. When is Prototype better than Builder?**
**Prototype** is better when creating an object is **expensive/complex** and you need **many similar instances**—clone a preconfigured template and tweak fields.

**Q. What must be true for clone() to be safe? 
Deep copy is important, but “safe clone” also requires no shared mutable state, preserved invariants, and no slicing / correct concrete type with clear ownership.

**Q. What’s the ownership model? (unique_ptr)**

**Q. How do you clone polymorphic types correctly in C++?**
Put a virtual `clone` in the base, override it in every derived to return a new instance of that derived copying its full state, so cloning through a base reference preserves the dynamic type, avoids slicing, and gives independent ownership.


---

## Quick Comparison (when to use what)

- **Factory Method**: choose _which class_ to instantiate
- **Abstract Factory**: choose _a whole family_ of related classes
- **Builder**: construct _one complex object_ stepwise
- **Prototype**: create _new objects by cloning templates_

## 5.6 Object Pool (Reuse expensive objects safely)

### Intuition
Creating/destroying some objects is costly:
- DB connections
- threads
- big buffers
- sockets
- expensive-to-initialize parsers

An **Object Pool** keeps a set of reusable objects and hands them out on demand.

Use it when:
- object creation is expensive **and**
- objects are reusable **and**
- you can bound concurrency (pool size)

---

### Concepts (Rules + pitfalls)

#### Core rule

> Borrow → use → return (always, even on exceptions)

So in C++, you typically wrap the borrowed object in an RAII handle that returns it automatically.

#### Pitfalls
- Pool becomes a global singleton (hidden dependency)
- Returning the same object twice
- Leaking borrowed objects (not returned)
- Concurrency issues (needs locking/queue)

---

### Architecture View

```cpp
Client -> Pool.acquire() -> [Obj]
Client uses Obj
Client -> returns Obj to Pool (RAII preferred)
```


---

### Practical Example (C++ RAII handle)

Minimal pool idea (conceptual, interview-friendly):
```cpp
template <typename T>
class ObjectPool {
    std::mutex m;
    std::vector<std::unique_ptr<T>> freeList;

public:
    explicit ObjectPool(size_t n) {
        freeList.reserve(n);
        for (size_t i = 0; i < n; ++i) freeList.push_back(std::make_unique<T>());
    }

    class Handle {
        ObjectPool* pool;
        std::unique_ptr<T> obj;
    public:
        Handle(ObjectPool* p, std::unique_ptr<T> o) : pool(p), obj(std::move(o)) {}
        Handle(Handle&&) = default;
        Handle& operator=(Handle&&) = default;
        T* operator->() { return obj.get(); }
        T& operator*() { return *obj; }
        ~Handle() { if (pool && obj) pool->release(std::move(obj)); }
    };

    Handle acquire() {
        std::lock_guard<std::mutex> lock(m);
        if (freeList.empty()) throw std::runtime_error("Pool exhausted");
        auto obj = std::move(freeList.back());
        freeList.pop_back();
        return Handle(this, std::move(obj));
    }

private:
    void release(std::unique_ptr<T> obj) {
        std::lock_guard<std::mutex> lock(m);
        freeList.push_back(std::move(obj));
    }
};
```

Usage:
```cpp
ObjectPool<DBConnection> pool(10);
{
    auto conn = pool.acquire();  // borrowed
    conn->query("SELECT ...");
} // auto returned even if exception
```

---

### Real-world use cases

- DB connection pools (classic)
- Thread pools (conceptually similar)
- Buffer pools for high-throughput networking
- Reusing parsers/encoders in media systems


---

### Interview Questions (Object Pool)

**Q. When is Object Pool a bad idea? (when object creation is cheap / over-complexity)**
Object Pool is a bad idea when object creation is cheap, objects aren’t safely resettable, pooling causes contention/latency, or it complicates lifetime management more than it helps.

**Q. How do you ensure objects are returned even on exceptions? (RAII handle)**
Use an RAII “lease/handle” whose destructor automatically returns the object to the pool, so exceptions can’t leak checked-out objects.

**Minimal idea (no fluff):**
- `auto obj = pool.acquire();` returns `PooledHandle<T>`
- `~PooledHandle()` calls `pool.release(obj)` automatically

**Q. What happens when pool is exhausted? (block vs throw vs expand)**
When exhausted, you either block (often with timeout), fail fast (throw/return error), or expand up to a hard max—choice depends on latency vs availability goals.

**Q. How do you make it thread-safe? (mutex/condvar)**
Use a **mutex** to protect the pool’s shared state (free list/queue) and a **condition_variable** so threads **wait when empty** and **wake on release** (`cv.wait(lock, pred)`), ensuring only one thread mutates the pool at a time.


---

## 5.7 Dependency Injection patterns in C++ (DIP in practice)

### Intuition

DIP says: depend on abstractions.  
DI is how you “wire” those abstractions in real code.

In C++, there’s no framework by default, so you must know the patterns.

---

### Concepts (Most useful DI styles)

#### A) Constructor Injection (default, best)
- dependencies visible
- objects valid immediately
- easy to test

```cpp
class Service {
    ILogger& logger;
public:
    explicit Service(ILogger& lg) : logger(lg) {}
};
```

#### B) Method Injection (when dependency used rarely)
```cpp
void process(ILogger& logger);
```

#### C) Setter Injection (avoid unless truly optional)
- risks partially-initialized objects


---

### Ownership rules (interview-critical)
- If service does **not** own dependency: store `T&` or `T*` (non-owning)
- If service **owns** dependency: store `std::unique_ptr<T>`
- If ownership is shared: `std::shared_ptr<T>` (use sparingly)

---

### Composition Root (where wiring happens)

> All `new` should ideally live in **one place**.

```
main() / AppBuilder / Bootstrapper
   |
   +--> build concrete objects
   +--> inject into services
```

Example:
```cpp
int main() {
    ConsoleLogger logger;
    MySqlOrderRepo repo;
    CheckoutService svc(logger, repo);
}
```

---

### Real-world use cases

- Making business logic unit-testable with fakes
- Swapping infrastructure: MySQL → Postgres → InMemory
- Plugin-like services created by factories

---

### Interview Questions (DI in C++)

**Q. Constructor vs setter injection: which and why?**
Use **constructor injection by default** because dependencies are **explicit + required**, the object is **valid immediately**, and tests can easily inject fakes. Use **setter injection only for truly optional/late-bound dependencies** (often framework-driven), because it risks **partially-initialized objects** and forces null-checks/ordering discipline.

**Q. Where do you place object creation in LLD?**
Put object creation in a **composition root / bootstrap layer** (e.g., `main()`, module initializer, DI container wiring), not inside domain/business classes, so logic stays decoupled from construction.
 
**Q. How do you manage ownership of injected dependencies?**
Define lifetimes at the **composition root**: services typically **hold references or `shared_ptr` to longer-lived dependencies**, use `unique_ptr` only when the service truly owns it exclusively, and ensure the dependency outlives the consumer to avoid dangling references.
“The container/composition root owns; services just reference.”



---

## 5.8 Registry / Service Locator (with warnings)

### Intuition

A **Registry** maps keys → creators/instances.  
Used for:

- plugin systems
- runtime extension (add new handlers without modifying core)
- selecting strategies by config

A **Service Locator** is a registry used globally to fetch dependencies.  
It works, but it’s an **anti-pattern in interviews unless justified**, because it hides dependencies.

---

### A) Registry (good when used explicitly)

#### Use-case: runtime selection of payment method without big switch
```cpp
class PaymentRegistry {
    using Creator = std::function<std::unique_ptr<IPaymentMethod>()>;
    std::unordered_map<std::string, Creator> creators;

public:
    void registerType(std::string key, Creator c) {
        creators.emplace(std::move(key), std::move(c));
    }

    std::unique_ptr<IPaymentMethod> create(const std::string& key) const {
        auto it = creators.find(key);
        if (it == creators.end()) throw std::invalid_argument("Unknown type");
        return (it->second)();
    }
};
```

Usage:
```cpp
PaymentRegistry reg;
reg.registerType("CARD", []{ return std::make_unique<CardPayment>(); });
reg.registerType("UPI",  []{ return std::make_unique<UpiPayment>(); });

auto method = reg.create("CARD");
method->pay(100);
```

✅ This is OCP-friendly and keeps selection centralized.

---

### B) Service Locator (warning)
```cpp
auto& repo = ServiceLocator::get<IOrderRepo>(); // hidden dependency
```

#### Why interviewers dislike it
- dependencies aren’t visible in constructors
- tests become hard (global state)
- encourages uncontrolled coupling

#### When it can be acceptable (rare)
- legacy systems
- frameworks where locator is unavoidable
- extreme plugin systems (still prefer explicit injection)

**Interview-safe line:**

> “I’ll avoid Service Locator and prefer explicit DI. If plugin registration is needed, I’ll use a registry passed into the composition root.”

---
### Real-world use cases

- Registries: handler maps, plugin loaders, command routing
- Service locator: usually legacy or framework-driven (avoid in fresh design)

---

### Interview Questions (Registry / Service Locator)

**Q. Registry vs Factory: how are they different?**
A **Factory creates new objects** (construction), while a **Registry stores and returns existing instances/handlers** by key (lookup/lifecycle).

**Q. Why is Service Locator an anti-pattern?**
Because it **hides dependencies** behind runtime lookup, which increases **global coupling**, makes code harder to reason about, and makes **tests brittle**

**Q. How do you keep registry testable? (inject it, don’t globalize)**
Make the registry an **interface** and **inject it** (constructor injection) so tests can pass a fake/in-memory registry; avoid global static registries.

**Q. How would you handle unknown keys? (fail fast vs default handler)**
Decide by domain: **fail fast** (exception/error) when missing key is a programming/config bug; use a **default handler** only when “unknown” is a valid runtime case—still log/metric it and consider returning `optional`/`expected` to force callers to handle it.