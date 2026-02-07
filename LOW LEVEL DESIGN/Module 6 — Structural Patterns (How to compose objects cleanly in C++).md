### Intuition

Structural patterns answer:
- “How do I **connect** objects so the system stays clean?”
- “How do I **wrap/compose** behavior without modifying core code?”
- “How do I integrate **incompatible interfaces** safely?”

In C++, structural patterns are also about:
- ownership/lifetime (unique_ptr vs reference)
- avoiding inheritance misuse
- keeping APIs stable

# 6.1 Adapter (Make incompatible interfaces work together)
## Intuition

Adapter is used when:
- you have an existing class (or 3rd-party library)
- its interface doesn’t match what your system expects
- you want to plug it in **without changing either side**

**Mental model:** “Convert interface, not behavior.

## Concepts
- **Target interface**: what your code expects
- **Adaptee**: the incompatible class you have
- **Adapter**: wraps adaptee and exposes target API

Two forms:

1. **Object Adapter** (preferred): composition (wrap instance)
2. **Class Adapter** (rare in C++): inheritance-based adaptation (often not needed)

## Architecture View

```nginx
Client --> IStorage (Target)
              ^
              |
        StorageAdapter
              |
              v
         AwsS3Client (Adaptee)
```

## Practical Example (C++)

### Target interface (your system)

```cpp
class IStorage {
public:
    virtual ~IStorage() = default;
    virtual void put(std::string key, std::string data) = 0;
    virtual std::string get(std::string key) = 0;
};
```

### Adaptee (3rd-party / legacy)

```cpp
class LegacyBlobClient {
public:
    void upload(const char* name, const char* bytes) { /*...*/ }
    const char* download(const char* name) { /*...*/ return ""; }
};
```

### Adapter

```cpp
class LegacyBlobAdapter : public IStorage {
    LegacyBlobClient& client; // non-owning
public:
    explicit LegacyBlobAdapter(LegacyBlobClient& c) : client(c) {}

    void put(std::string key, std::string data) override {
        client.upload(key.c_str(), data.c_str());
    }

    std::string get(std::string key) override {
        return client.download(key.c_str());
    }
};
```

Now your service stays clean:

```cpp
class Uploader {
    IStorage& storage;
public:
    explicit Uploader(IStorage& s) : storage(s) {}
    void uploadFile(std::string k, std::string data) { storage.put(k, data); }
};
```

---

## Real-World Use Cases

- Integrating vendor SDKs (payment gateway, cloud storage)
- Wrapping legacy code behind modern interfaces 
- Converting data formats (DTO ↔ domain)
- Migrating systems gradually (old → new) without rewriting callers

---

## Interview Questions (Adapter)

**Q. Adapter vs Facade: difference?**
**Facade** simplifies a complex subsystem behind a unified API; **Adapter** converts one interface into another so incompatible code can work together.

**Q. Object adapter vs class adapter: which is preferred and why?**
Object adapters are preferred because they use **composition over inheritance**, avoid tight coupling, and work with existing/adapted classes without subclassing.

**Q. Where does Adapter live in architecture? (at boundaries)**
At **system boundaries** (SDK, DB, network, legacy APIs) to isolate external changes from core domain logic.

**Q. How do you test adapters? (fake adaptee / integration tests)**
Unit-test with a **fake/mock adaptee** for translation logic, and add **integration tests** against the real dependency to catch contract drift.



---

# 6.2 Decorator (Add behavior without modifying a class)

## Intuition

Decorator is used when you want to add behavior like:
- logging
- retries
- metrics
- caching
- authorization

…without touching the original implementation, and without inheritance explosion.

**Mental model:** “Wrap to extend.”

---

## Concepts

- Decorator implements the same interface as the wrapped object
- It forwards calls, adding behavior before/after

---

## Architecture View

```yaml
Client -> IGateway
            |
            v
       RetryDecorator
            |
            v
      LoggingDecorator
            |
            v
        RealGateway
```

Order matters (important interview note).

---

## Practical Example (C++)

### Interface

```cpp
class IPaymentGateway {
public:
    virtual ~IPaymentGateway() = default;
    virtual bool charge(double amount) = 0;
};
```

### Concrete implementation

```cpp
class StripeGateway : public IPaymentGateway {
public:
    bool charge(double amount) override {
        // real API call
        return true;
    }
};
```

### Decorator base (optional but clean)

```cpp
class GatewayDecorator : public IPaymentGateway {
protected:
    std::unique_ptr<IPaymentGateway> wrap;
public:
    explicit GatewayDecorator(std::unique_ptr<IPaymentGateway> g)
        : wrap(std::move(g)) {}
};
```

### Logging decorator

```cpp
class LoggingGateway : public GatewayDecorator {
public:
    using GatewayDecorator::GatewayDecorator;

    bool charge(double amount) override {
        // log before
        bool ok = wrap->charge(amount);
        // log after
        return ok;
    }
};
```

### Retry decorator

```cpp
class RetryGateway : public GatewayDecorator {
    int retries;
public:
    RetryGateway(std::unique_ptr<IPaymentGateway> g, int r)
        : GatewayDecorator(std::move(g)), retries(r) {}

    bool charge(double amount) override {
        for (int i = 0; i <= retries; ++i) {
            if (wrap->charge(amount)) return true;
        }
        return false;
    }
};
```

Composition (single place):

```cpp
auto gw =
  std::make_unique<RetryGateway>(
    std::make_unique<LoggingGateway>(
      std::make_unique<StripeGateway>()
    ),
    3
  );
```

---

## Real-World Use Cases

- API client wrappers (retry, circuit breaker, logging)
- Adding caching around repositories
- Security checks around service methods
- Metrics around any interface

---

## Interview Questions (Decorator)

**Q. Decorator vs Inheritance: when decorator is better?**
**Decorator is better when you want to add/remove behaviors at runtime and combine them without creating many subclasses** (e.g., logging, retry, caching, metrics), while inheritance is better when you’re defining a true “is-a” specialization with a stable type hierarchy.


**Q. Decorator vs Proxy: difference?**
- **Decorator** adds features/responsibilities to an object (enhancement).
- **Proxy** controls access to an object (lazy-load, remote call, auth, caching, rate-limit) while trying to keep behavior “equivalent” from the caller’s view.


**Q. Why does decorator ordering matter? Give example.**
Ordering changes _what gets wrapped_, so side-effects and guarantees change.
**Example:**
- `Retry(Logging(Stripe))` → logs **every attempt** (good for debugging retries).
- `Logging(Retry(Stripe))` → logs **once per request** (cleaner logs, less noise).

(Another quick one) `Auth(RateLimit(Service))` vs `RateLimit(Auth(Service))` changes whether unauthenticated traffic consumes rate-limit.


**Q. How do you avoid “wrapper explosion”? (compose in one place, use pipelines)**
Wrapper explosion is when decorator-based designs create too many wrapper layers/combinations, causing complexity and inconsistent behavior.
Keep decorators **small and orthogonal**, and **compose them in one place** (composition root) using a pipeline/builder/config, instead of hardcoding combinations across the codebase.

# 6.3 Composite (Treat individual objects and groups uniformly)

## Intuition

Use Composite when you have a **tree structure** and you want to apply the same operation on:
- a **leaf** (single item)
- a **composite** (collection of items)

Classic examples:
- File system (File / Folder)
- UI components (Button / Panel)
- Org chart (Employee / Manager)

**Mental model:** “Leaf and Container share the same interface.”

---

## Concepts

- `Component` interface defines operations (`size()`, `render()`, `cost()`)
- `Leaf` implements operations directly
- `Composite` stores children and delegates operations over them

---

## Architecture View
```cpp
          Component
          /      \
       Leaf     Composite
                  |
               children: Component*
```

---

## Practical Example (C++: File System)

```cpp
class Node {
public:
    virtual ~Node() = default;
    virtual size_t size() const = 0;
};

class File : public Node {
    size_t bytes;
public:
    explicit File(size_t b) : bytes(b) {}
    size_t size() const override { return bytes; }
};

class Folder : public Node {
    std::vector<std::unique_ptr<Node>> children; // owns children (composition)
public:
    void add(std::unique_ptr<Node> n) { children.push_back(std::move(n)); }

    size_t size() const override {
        size_t sum = 0;
        for (auto& c : children) sum += c->size();
        return sum;
    }
};
```

Usage:

```cpp
auto root = std::make_unique<Folder>();
root->add(std::make_unique<File>(100));
auto sub = std::make_unique<Folder>();
sub->add(std::make_unique<File>(50));
root->add(std::move(sub));
auto total = root->size(); // works uniformly
```

---

## Real-World Use Cases

- Hierarchical permissions / policies
- Menu systems
- Expression trees (parsers)
- Nested configuration trees

---

## Interview Questions (Composite)

Q. Why is Composite better than having separate APIs for leaf and folder?
Composite is better because it gives a **single uniform interface** for both leaf and container, so clients don’t need type-checks or separate code paths—operations work **recursively** over the whole tree.

Q. How do you represent ownership of children in C++? (unique_ptr)
Use `std::vector<std::unique_ptr<Node>>` to express **exclusive ownership** and automatic lifetime cleanup (folder destroys children).

_(If shared child nodes are needed: `shared_ptr`, but avoid unless truly required.)_

Q. What’s the downside? (sometimes leaf doesn’t support add/remove; keep interface minimal)
The interface can get awkward: leaves often can’t meaningfully support `add/remove`, so keep the base interface minimal (or separate “Composite” operations from the common “Node” interface).

Q. Composite vs Iterator: relationship?
Composite models the **tree structure**; Iterator provides a **traversal mechanism** (DFS/BFS) over that tree without exposing internal representation—iterators are often built on top of composites.


# 6.4 Facade (Provide a simple API over a complex subsystem)

## Intuition
When a subsystem has many moving parts, clients shouldn’t coordinate them.
Facade creates a **single high-level interface** that hides complexity.
**Mental model:** “One door to many rooms.”

---

## Concepts
- Facade does **orchestration**, not deep business logic
- Internals stay decoupled; facade simplifies usage

---

### Architecture View

```
Client --> VideoEncodingFacade
             |
             +--> Decoder
             +--> FilterChain
             +--> Encoder
             +--> Storage

```

---

## Practical Example (C++)

Subsystem:

```cpp
class Inventory { public: void reserve(int itemId) {/*...*/} };
class Payment { public: bool charge(double amt) {/*...*/ return true;} };
class Invoice { public: void generate(int orderId) {/*...*/} };

```

Facade:

```cpp
class CheckoutFacade {
    Inventory& inv;
    Payment& pay;
    Invoice& invoice;
public:
    CheckoutFacade(Inventory& i, Payment& p, Invoice& in)
        : inv(i), pay(p), invoice(in) {}

    bool checkout(int orderId, int itemId, double amount) {
        inv.reserve(itemId);
        if (!pay.charge(amount)) return false;
        invoice.generate(orderId);
        return true;
    }
};
```

Client now calls 1 method instead of coordinating 3 subsystems.

---

## Real-World Use Cases
- “PlaceOrder” API that coordinates inventory + payment + shipping
- SDK wrappers (simple interface over complex vendor APIs)
- Startup/bootstrap: initialize many subsystems behind one entry point

---

## Interview Questions (Facade)

Q. Facade vs Adapter: difference?
**Facade** simplifies a complex subsystem behind a unified API; **Adapter** converts one interface into another so incompatible code can work together.

Q. Facade vs Service (application service): how are they similar/different?
 Both can orchestrate calls, but:
- **Facade** is a **UI/integration convenience layer** over a subsystem (simplifies usage; often thin).
- **Application Service** is a **use-case layer** that coordinates domain operations and enforces business workflow/transactions.

**One-liner:** Facade simplifies a subsystem; Application Service implements a business use case.

Q. How to prevent facade from becoming a God Object?
Keep the facade thin: only orchestrate and delegate; keep business rules in domain/services, split into multiple facades by use-case/subsystem boundary, and avoid accumulating unrelated methods.

Q. When should you not introduce a facade?
Don’t add a facade when the subsystem is already small/clear, when clients genuinely need fine-grained control over components, or when a facade would become a bloated “do-everything” class that hides important capabilities and changes too often.


---

# 6.5 Proxy (Control access to an object)

## Intuition

Proxy is a stand-in object that controls access to the real object.

Use it when you need:
- lazy initialization (create heavy object only when needed)
- access control (auth checks)
- remote proxy (RPC calls)
- caching proxy
- rate-limiting proxy

**Mental model:** “Same interface, but with gatekeeping.”


---

### Concepts

Proxy implements same interface as real object:
- forwards calls, but adds control behavior

---

### Architecture View
```
Client --> IService
            |
            +--> ProxyService --> RealService
```

---

### Practical Example (Lazy Proxy in C++)

```cpp
class IImage {
public:
    virtual ~IImage() = default;
    virtual void display() = 0;
};

class RealImage : public IImage {
    std::string path;
public:
    explicit RealImage(std::string p) : path(std::move(p)) {
        // expensive load
    }
    void display() override { /* render */ }
};

class ImageProxy : public IImage {
    std::string path;
    std::unique_ptr<RealImage> real; // created lazily
public:
    explicit ImageProxy(std::string p) : path(std::move(p)) {}

    void display() override {
        if (!real) real = std::make_unique<RealImage>(path);
        real->display();
    }
};
```

---

### Real-World Use Cases

- Database connection lazy initialization
- Authorization proxy around service methods
- Caching proxy for read-heavy APIs
- Remote proxies for microservice clients (client stub)


---

### Proxy vs Decorator (interview trap)

- **Proxy**: controls access (lazy/auth/remote). Client thinks it’s the real thing.
- **Decorator**: adds features/behavior (logging/metrics/retry) usually as layered wrappers.  
    In practice they look similar; difference is intent.    

---

## Interview Questions (Proxy)

Q. Proxy vs Decorator: differentiate by intent.
- **Proxy:** _controls access_ to a real object (lazy-load, remote call, auth, caching).
- **Decorator:** _adds responsibilities/features_ around the same interface (logging, metrics, retry), and wrappers can be stacked.

Q. How would you implement a caching proxy?
I’d place a caching proxy in front of the real cache/data source. On a request, the proxy first checks its local key-value cache and returns immediately on a hit. On a miss, it delegates to the real cache (DB/Redis/service), stores the result in the proxy cache with TTL/eviction, and returns it.

Q. What are common proxy types? (virtual/lazy, remote, protection, caching)
- **Virtual / Lazy proxy:** delays expensive creation until first use
- **Remote proxy:** represents an object in another process/network
- **Protection proxy:** enforces access control/permissions
- **Caching proxy:** caches results to reduce expensive calls

Q. Where does proxy live? (boundary layer)
Proxy typically lives at the **boundary/integration layer** (DB/network/filesystem/third-party APIs), so domain logic stays clean and unaware of access/caching/security concerns

# 6.6 Bridge (Separate abstraction from implementation so both can vary independently)

## Intuition

Bridge is used when you have **two dimensions of change** and you don’t want an inheritance explosion.

Classic pain:
- You model one dimension using inheritance…
- then you need another dimension…
- you end up with `AxByCz` subclasses.

Bridge splits:

- **Abstraction** (what client sees)
- **Implementor** (how it’s done)

So you can mix and match at runtime.

## When to use (fast rule)

If you see:  
**“X can be combined with Y”**  
(e.g., `Shape` with different `Renderer`s, `Remote` with different `Device`s) → Bridge.


## Architecture View

```
      Abstraction  ----has-a---->  Implementor
         |                            |
   RefinedAbstraction            ConcreteImplementor

```


## Practical Example (Shape × Renderer)

**Implementor**
```cpp
class IRenderer {
public:
    virtual ~IRenderer() = default;
    virtual void drawCircle(double r) = 0;
};

class RasterRenderer : public IRenderer {
public:
    void drawCircle(double r) override { /* raster draw */ }
};

class VectorRenderer : public IRenderer {
public:
    void drawCircle(double r) override { /* vector draw */ }
};
```

**Abstraction**
```cpp
class Shape {
protected:
    IRenderer& renderer; // non-owning, injected
public:
    explicit Shape(IRenderer& r) : renderer(r) {}
    virtual ~Shape() = default;
    virtual void draw() = 0;
};

class Circle : public Shape {
    double radius;
public:
    Circle(IRenderer& r, double rad) : Shape(r), radius(rad) {}
    void draw() override { renderer.drawCircle(radius); }
};
```

Now you can combine independently:
```cpp
RasterRenderer rr;
VectorRenderer vr;

Circle c1(rr, 10);
Circle c2(vr, 10);
```

✅ No `RasterCircle`, `VectorCircle`, etc.



## Real-World Use Cases

- UI abstraction vs OS backend (Windows/Mac/Linux)
- Storage abstraction vs backend (S3/GCS/local)
- Device + remote control
- Rendering engines


## Interview Questions (Bridge)

**Q. Bridge vs Strategy: difference?**
**Strategy** swaps a behavior/algorithm for a client, while **Bridge** splits a type into **abstraction + implementor** so both can evolve independently and you avoid subclass explosion across two dimensions.

**Q. What inheritance explosion does Bridge prevent?**
It prevents the cross-product explosion like `CircleRaster`, `CircleVector`, `SquareRaster`, `SquareVector`, … where every new shape × every new renderer requires a new subclass.

**Q. How does DIP relate to Bridge?**
Bridge applies DIP by making the abstraction depend on an **interface (`IRenderer`)**, not concrete renderers, so high-level shapes don’t depend on low-level drawing implementations.



# 6.7 Flyweight (Share common data to save memory)

## Intuition
If you have **huge numbers** of similar objects, memory blows up.
Flyweight splits object state into:
- **Intrinsic (shared)**: common, immutable data
- **Extrinsic (per-instance)**: small, supplied at use time

Use it when:
- millions of objects
- many repeat the same internal data
- memory dominates performance


## Architecture View

```sql
FlyweightFactory -> returns shared Flyweight
Client supplies extrinsic state when using it
```


## Practical Example (Text rendering / glyphs)

Intrinsic shared data: glyph shape for each character.  
Extrinsic per-use data: position, color, size.

```cpp
struct Glyph {
    char ch; // placeholder for heavy glyph data
    explicit Glyph(char c) : ch(c) {}
    void draw(int x, int y) const { /* draw glyph at x,y */ }
};

class GlyphFactory {
    std::unordered_map<char, std::shared_ptr<Glyph>> cache;
public:
    std::shared_ptr<Glyph> get(char c) {
        auto it = cache.find(c);
        if (it != cache.end()) return it->second;
        auto g = std::make_shared<Glyph>(c);
        cache[c] = g;
        return g;
    }
};
```

Usage:
```cpp
GlyphFactory f;
auto gA = f.get('A'); // shared
gA->draw(10, 20);     // extrinsic x,y passed per call
```


## Real-World Use Cases
- Text editors (glyphs)
- Game engines (trees/rocks with same mesh, different positions)
- Particle systems
- Caching immutable configs/templates


## Interview Questions (Flyweight)

**Q. Intrinsic vs extrinsic state—define with example.**
- **Intrinsic state:** shared, usually immutable data stored inside the flyweight (e.g., glyph shape/bitmap for `'A'`).
- **Extrinsic state:** per-use/per-instance data passed in at runtime (e.g., `(x,y)`, color, font size when drawing).

Q. Flyweight vs Object Pool?
 Flyweight shares immutable data; Pool reuses mutable objects over time.

**Q. Why does Flyweight often use `shared_ptr`?**
Because flyweights are **shared by many clients simultaneously**, and `shared_ptr` cleanly expresses shared ownership/lifetime of cached objects returned by the factory.

**Q. What’s the trade-off? (complexity + factory lookup overhead)**
You trade memory for **added complexity** (managing intrinsic vs extrinsic), **factory/cache lookup overhead**, and potential **thread-safety** concerns in the shared cache.



---

## 6.8 PImpl (Pointer to Implementation) — C++ encapsulation + compile-time + ABI stability

## Intuition
PImpl hides implementation details from headers.
It solves 3 real C++ pains:
1. Reduce compile-time dependencies (fewer includes in headers)
2. Keep ABI stable (implementation can change without recompiling users)
3. Hide private members (true encapsulation)

Use it when:
- large headers included everywhere
- you want stable library ABI
- you want to hide third-party headers from users

---

### Architecture View
```
User includes .h
  |
  v
PublicClass (small, stable)
  |
  v
Impl (defined in .cpp, can change freely)

```

---

### Practical Example (C++)

**Header (Widget.h)**
```cpp
#pragma once
#include <memory>
#include <string>

class Widget {
public:
    Widget();
    ~Widget();                    // defined in .cpp
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;

    Widget(const Widget&) = delete;
    Widget& operator=(const Widget&) = delete;

    void setName(std::string name);
    std::string name() const;

private:
    struct Impl;                   // forward-declare
    std::unique_ptr<Impl> pImpl;   // owns implementation
};
```

**Source (Widget.cpp)**
```cpp
#pragma once
#include <memory>
#include <string>

class Widget {
public:
    Widget();
    ~Widget();                    // defined in .cpp
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;

    Widget(const Widget&) = delete;
    Widget& operator=(const Widget&) = delete;

    void setName(std::string name);
    std::string name() const;

private:
    struct Impl;                   // forward-declare
    std::unique_ptr<Impl> pImpl;   // owns implementation
};
```


### Key C++ rules (interview-critical)

- Use `unique_ptr<Impl>` for ownership
- Destructor must be out-of-line (`~Widget()` in .cpp), because `Impl` is incomplete in header
- Copy requires deep copy logic; many PImpl types are **move-only** (common)
- Great for library code and big projects


## Real-World Use Cases
- Public SDKs/libraries
- Large codebases where build times matter
- Hiding third-party types (OpenSSL, AWS SDK) from public headers


## Interview Questions (PImpl)

**Q. Why does PImpl reduce compile time?**

**Q. Why must destructor be in .cpp?**

**Q. How do you implement copy semantics with PImpl?**

**Q. What’s the runtime cost? (one heap alloc + indirection)**



## Quick Comparison (Bridge vs Strategy vs PImpl)

- **Bridge**: 2 varying hierarchies, avoid subclass explosion
- **Strategy**: swap one behavior/algorithm
- **PImpl**: hide implementation + reduce compile-time coupling (not about runtime flexibility