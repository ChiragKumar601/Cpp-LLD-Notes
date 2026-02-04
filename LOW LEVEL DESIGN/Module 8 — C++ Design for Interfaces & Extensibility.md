# 8.1 Interface design in C++ (pure abstract classes, minimal APIs)
## Intuition

In LLD, your “interface” is the **contract boundary**. Good interfaces:
- expose **minimum** needed behavior
- hide representation + heavy headers
- are hard to misuse
- are stable even as implementations grow

## Concepts (rules you must know)

**Rule 1 — Prefer small interfaces**
- If an interface has 10+ methods, you likely mixed responsibilities (violates ISP).

**Rule 2 — Pure abstract base = “protocol”
**
```cpp
struct IClock {
    virtual ~IClock() = default;     // must be virtual
    virtual long long nowMs() const = 0;
};
```

**Rule 3 — Never forget virtual destructor**  
If you delete via base pointer without it → UB.

**Rule 4 — Avoid returning owning raw pointers**  
Return `std::unique_ptr<T>` or value or `std::shared_ptr<T>` (rare).

**Rule 5 — Don’t leak concrete types in interface headers**  
Keep interfaces “light”: forward-declare, use PImpl, avoid including heavy vendor headers.

**Rule 6 — Const-correctness is part of the contract**
- `foo() const` means thread-safety/read-only intent (not guaranteed, but signal).

**Rule 7 — Prefer non-throwing destructor**  
Destructors should not throw.

## Architecture view

```
App/Domain  -->  IPort (interface)  -->  Adapter/Impl
```

This is Ports & Adapters at LLD level.

#### Practical example

**Payment gateway boundary**
```cpp
struct IPaymentGateway {
    virtual ~IPaymentGateway() = default;
    virtual bool charge(double amount, std::string_view currency) = 0;
};
```

**Minimal API design:** only what the domain needs now; extend later via versioning strategy (8.5).

## Real-world use cases
- Swap vendor SDKs
- Test with fakes
- Multi-environment (prod/stage/dev) implementations

## Interview questions

**Q. What makes an interface “too fat”?**
An interface is “too fat” when it violates **ISP**—it packs too many unrelated responsibilities (often seen as “10+ methods” and low cohesion), forcing implementers/clients to depend on methods they don’t need, leading to no-op/“not supported” implementations and frequent ripple changes.

**Q. Why must base destructors be virtual?**
If an object can be deleted via a base pointer, the base destructor must be **virtual** so `delete Base*` dispatches to the derived destructor; otherwise you get **undefined behavior** (partial destruction/resource leaks).
 
**Q. How do you keep interface headers stable/fast to compile?**
Keep interface headers **minimal and dependency-light**: forward-declare types, avoid heavy/vendor includes, hide details behind **PImpl/adapters**, keep templates out of public headers when possible, and evolve via **extension interfaces/capabilities** instead of editing the base contract repeatedly.


# 8.2 Ownership at boundaries (who deletes what?)

## Intuition

C++ boundaries fail when ownership is ambiguous. You must be able to answer:

> “Who owns this object and who deletes it?”

## The 4 ownership patterns (LLD staples)

**A) Value ownership (best if small/concrete)**
```cpp
class Engine { ... };
class Car { Engine engine; };   // Car owns engine
```

**B) Unique ownership**
- one owner, clear lifetime

```cpp
std::unique_ptr<IParser> p = makeParser();
```

Factory returns `unique_ptr`, caller owns.

**C) Shared ownership (use sparingly)**
- multiple owners, shared lifetime
```cpp
std::shared_ptr<Config> cfg;
```

Use only when “who owns” is genuinely plural.

**D) Non-owning reference**
- observes, doesn’t delete

```cpp
class Service {   
	ILogger& logger; // must outlive Service 
}
```

Or `T*` with clear “non-owning” doc.

#### Boundary rulebook (interview-ready)
* **Factories return owning** `unique_ptr<Base>`
* **Services store non-owning refs** to long-lived dependencies
* **Collections own children with** `vector<unique_ptr<T>>`
* Use **IDs** instead of pointers for cross-aggregate references (safer)

**1-line SDE3 rule:**  
Default to **A or B**; use **C only when lifetime is truly shared**, and use **D for dependencies** where ownership lives elsewhere.

## Practical example (plugin creation)

```cpp
std::unique_ptr<IFormatter> createFormatter(); // caller owns
```

#### Real-world use cases
- Resource lifetime in servers
- Avoid leaks/cycles in observer patterns
- Avoid dangling pointers in caches

#### Interview questions

**Q. When do you choose `unique_ptr` vs `shared_ptr`?**
Use **`std::unique_ptr`** by default to express **single, clear ownership** (transfer via move; ideal for factories returning owned objects). Use **`std::shared_ptr`** only when **multiple independent owners must extend lifetime** and there’s no natural single owner—otherwise it introduces hidden coupling, ref-count overhead, and cycle risks (often pair with `weak_ptr`).

**Q. How do you represent “optional ownership”? (`unique_ptr` or `optional<T>`)**
Use `std::optional<T>` when you want _optional value semantics_ (a T may or may not exist, stored inline), and use `std::unique_ptr<T>` when you want _optional heap ownership_—especially for polymorphism, large/non-movable objects, or stable lifetime/identity.

**Q. Why are IDs safer than pointers across modules?**
Pointers depend on address + lifetime + allocator/ABI, so across modules/plugins they can dangle, break on unload, or be freed by the wrong runtime; IDs/handles are stable, checkable, serializable, and let the owning module safely resolve access and lifetime.


# 8.3 Plugin architecture basics (factory registry + dynamic dispatch)

## Intuition

Plugins mean: “Add new behavior without modifying core code.”  
Core knows only:
- a **stable interface**
- a **registry** (map from key → creator)
- optional **dynamic loading** (DLL/SO) (often out of scope, but mentionable)

## Architecture view

```
Core -> IPlugin
Core -> Registry(key -> creator)
PluginX registers itself
PluginY registers itself
```

## Concepts

- **Dynamic dispatch**: call through interface
- **Registration**: plugin announces itself to registry
- **Creation**: registry returns `unique_ptr<IPlugin>` 

## Practical example (registry)

```cpp
struct IPlugin {
    virtual ~IPlugin() = default;
    virtual std::string_view name() const = 0;
    virtual void run() = 0;
};

class PluginRegistry {
public:
    using Creator = std::function<std::unique_ptr<IPlugin>()>;

    void registerPlugin(std::string key, Creator c) {
        creators.emplace(std::move(key), std::move(c));
    }

    std::unique_ptr<IPlugin> create(const std::string& key) const {
        auto it = creators.find(key);
        if (it == creators.end()) throw std::invalid_argument("Unknown plugin");
        return (it->second)();
    }

private:
    std::unordered_map<std::string, Creator> creators;
};
```

Plugin registers:

```cpp
class PdfPlugin : public IPlugin {
public:
    std::string_view name() const override { return "pdf"; }
    void run() override { /*...*/ }
};

// wiring
registry.registerPlugin("pdf", []{ return std::make_unique<PdfPlugin>(); });
```

## Real-world use cases

- Notification channels (email/sms/whatsapp)
- Pricing rules
- Parsers/exporters (csv/json/xml)
- Command handlers

## Interview questions

Q. Factory vs Registry?

Q. How do you prevent registry from becoming a global service locator?
Make the registry **non-global and injected**, keep it **narrow (plugin catalog only)**, build it at startup and **freeze/read-only**, so dependencies remain explicit and testable.

**Bad (global service locator):**
```cpp
// anywhere in code:
auto db = Registry::instance().get<DbClient>();
auto logger = Registry::instance().get<Logger>();
```

Problems: hidden dependencies, hard to test, global mutable state.

**Good (inject a narrow plugin registry):**

```cpp
struct IPluginRegistry {
  virtual std::unique_ptr<IPlugin> create(const std::string& type) = 0;
  virtual ~IPluginRegistry() = default;
};

class Engine {
  IPluginRegistry& reg;     // explicit dependency
public:
  explicit Engine(IPluginRegistry& r) : reg(r) {}
  void run(const std::string& type) {
    auto p = reg.create(type);
    p->execute();
  }
};
```

Now Engine only depends on “can create plugins”, not “give me anything”.

Q. How do you test plugins? (inject registry, fake plugin)
Inject a registry (or loader interface) and register **fake plugins** for unit tests, plus a small integration test that loads a real plugin and asserts `register → create → execute` works end-to-end.

---

# 8.4 Type erasure patterns for extensible APIs (`std::function`, `any`, custom)

## Intuition

Sometimes you want extensibility without forcing inheritance everywhere.  
Type erasure lets you store “anything that behaves like X” behind a uniform type.

Use when:
- you want plugin-like extensibility
- you want to avoid deep base-class hierarchies
- you want performance + stable ABI boundaries


## A) `std::function` (erase callable type)

```cpp
using Handler = std::function<void(const Event&)>;
```

Pros: easy, composable  
Cons: allocation possible, overhead, cannot introspect type

## B) `std::any` (erase value type)

```cpp
std::any payload = std::string("hello");
auto& s = std::any_cast<std::string&>(payload);
```

Pros: flexible payloads
Cons: runtime type errors if not disciplined

## C) Custom type erasure (classic LLD answer)

You define a wrapper with:
- concept interface
- `model<T> storing concrete`
- forwarding calls

Example: “Formatter” without inheritance requirement:
```cpp
class AnyFormatter {
    struct Concept {
        virtual ~Concept() = default;
        virtual std::string format(std::string_view) const = 0;
    };
    template <typename T>
    struct Model : Concept {
        T obj;
        explicit Model(T o) : obj(std::move(o)) {}
        std::string format(std::string_view s) const override { return obj.format(s); }
    };

    std::unique_ptr<Concept> self;

public:
    template <typename T>
    AnyFormatter(T x) : self(std::make_unique<Model<T>>(std::move(x))) {}

    std::string format(std::string_view s) const { return self->format(s); }
};
```



Now any type with `format()` works:

```cpp
struct JsonFmt { std::string format(std::string_view s) const { return "{...}"; } };
AnyFormatter f = JsonFmt{};
f.format("x");
```

## Real-world use cases

- Event handlers / middleware pipelines (`std::function`)
- Generic payload buses (`std::any` with discipline)
- Stable ABI plugin boundaries (custom type erasure + PImpl)

## Interview questions

**Q. When do you prefer type erasure over inheritance?**
Prefer type erasure when you need plugin-like extensibility over unrelated types without forcing a base class, and want a small stable API across module/ABI boundaries.
 
**Q. What are the costs of `std::function`?**
`std::function` trades flexibility for overhead: type-erased call indirection (often no inlining), possible heap allocation for large callables (despite SBO), and extra copy/move cost from storing the erased callable.
 
**Q. How do you keep `std::any` safe? (tagged protocols / `enums` / schema)**
Treat `std::any` as ‘payload + protocol’: always pair it with an explicit tag/schema (enum/type id), centralize the `any_cast` based on that tag, and prefer `std::variant` when the payload types are a known finite set.

### 8.5 Versioning & backward compatibility (practical rules)

#### Intuition

Interfaces evolve. Your job is to evolve them without breaking:

- existing implementations
- existing binaries (if you ship libs)
- existing callers

#### Practical rule set (LLD + C++ reality)

**Rule 1 — Never break an interface method signature casually**  
Changing method parameters/return type breaks all implementers.

**Rule 2 — Prefer adding new methods via “capabilities”**  
Instead of growing one fat interface:

- keep base stable
- add optional extension interfaces


Example:
```cpp
struct IStorageV1 { virtual void put(...) = 0; };
struct IStorageV2 : IStorageV1 { virtual void putWithTTL(...) = 0; };
```

Caller does:
- use V1 always
- only use V2 feature if supported (capability check)

Example idea:
```cpp
if (auto* v2 = dynamic_cast<IStorageV2*>(storage)) v2->putWithTTL(...);
else storage->put(...); // fallback
```

This keeps old implementations working (they still implement V1).

**Rule 3 — Prefer free functions / non-virtual extension points**  
If you control callers, adding a free function avoids vtable changes.

**Rule 4 — ABI warning (important if asked)**  
Changing:

- virtual methods order
- adding data members
- changing layout  
    can break ABI for precompiled clients.

**Rule 5 — Version at the boundary**
- Include version in plugin keys: `"formatter:v1"`, `"formatter:v2"`
- Or expose `int apiVersion()` on plugins

```cpp
struct IPlugin {
  virtual int apiVersion() const = 0;
};
```

**Rule 6 — Deprecation path**
- keep old method working
- introduce new API
- migrate callers
- remove only in major version

## Practical example (plugin compatibility)
```cpp
Core supports IPluginV1 and IPluginV2
Registry stores creators for both
Core picks best supported version at runtime
```

## Real-world use cases
- long-lived SDKs
- plugin ecosystems
- backwards-compatible services

## Interview questions

**Q. How do you evolve an interface without breaking implementers?**
Keep old signatures stable, add features via capability interfaces/non-virtual extensions, version at the boundary to avoid ABI breaks (don’t change vtable/layout; use PImpl if needed), and keep old behavior with a deprecation path for callers.
 
**Q. How do you handle optional new features?**
Core knows only a stable interface + registry, optional features come via capability/versioned interfaces (V2 extends V1) with fallback, and plugins can be dynamically loaded with an API version check.
 
**Q. What’s the difference between API break and ABI break?**
**API break (source-level):** a change that breaks _compilation_ for callers/implementers (e.g., change a method signature, rename/remove a function).

**ABI break (binary-level):** a change that breaks _already-compiled binaries_ at link/run time (e.g., change class layout/vtable/virtual order, calling convention), even if the source still compiles.