#### SRP (Single Responsibility Principle) — “One Reason To Change”
SRP does not mean "one class does one thing".

It means:
A class/module should have exactly one business reason to change.

If a class changes for two unrelated reasons, it becomes a "blast radius" class:
* a small feature change breaks unrelated behavior
* tests become hard
* refactoring becomes risky

❌ One class doing everything (business + DB + email + formatting)

```cpp
class OrderService {
public:
    void placeOrder(int userId, int amount) {
        int total = amount + calculateTax(amount);      // business logic
        saveToDatabase(userId, total);                  // persistence
        sendEmail(userId, "Order placed. Total=" + std::to_string(total)); // communication
        log("Placed order");                            // logging
    }

private:
    int calculateTax(int amount) { return amount * 18 / 100; }

    void saveToDatabase(int userId, int total) {
        // DB code...
    }

    void sendEmail(int userId, const std::string& msg) {
        // SMTP / API call...
    }

    void log(const std::string& msg) {
        // logging...
    }
};
```
This class has **multiple responsibilities**, so it has **multiple reasons to change**.

**Small feature change breaks unrelated behavior**
- You change email formatting, and accidentally break order saving or tax logic because everything is tangled in one method.

**Tests become hard**
* To test tax calculation, you’re forced to also deal with DB + email + logging, so unit tests need heavy mocking or become slow/flaky.

**Refactoring becomes risky**
* Changing DB or email provider touches the same class, so a refactor in one area can silently break other responsibilities.



**Core Concepts**

1. **Responsibility = “Reason to change”, not “method count”**  
A class can have 20 methods and still follow SRP if all serve one reason (one axis).

2. **SRP is about "who is the client of this code"**
  If two different stakeholders want changes:
  * product/business logic changes
  * infra/format/ persistence changes

3. **SRP boundary is a seam for change**
A “seam” is a **place you can modify/replace without touching other code**.

You split SRP where **change patterns differ**:
- Business rules change often (pricing, discounts)
- Infrastructure changes differently (DB, API format, storage)

So you put them in separate classes so each can evolve independently.

4. A good SRP split makes dependencies point inward.
This means:
	Business logic should not depend on external details like DB, printing, file, I/O, JSON.

Because business rules are the core; everything else is just a delivery mechanism.

**Bad dependency direction**
`BusinessLogic → DB/JSON/Logging`

**Good dependency direction**
`DB/JSON/Logging → BusinessLogic`

So business code stays stable, testable, and reusable.



**One-line memory hook**
(1) SRP = one reason to change
(2) Different stakeholders = different responsibilities
(3) Split where change frequency differs
(4) Core logic shouldn’t depend on IO/infra details


**Architecture View**

Bad (mixed reasons):

```
OrderService
  - calculateTotal()      (business)
  - saveToDB()            (persistence)
  - printInvoice()        (presentation)
```

Good (each has one reason to change):
```
        +------------------+
        |  OrderService    |  (business rules)
        +------------------+
           |          |
           v          v
+----------------+  +------------------+
| OrderRepo      |  | InvoiceRenderer  |
| (persistence)  |  | (presentation)   |
+----------------+  +------------------+
```

**Practical Example (C++ — SRP refactor)**

❌ Non-SRP (changes for pricing + DB + printing)
```cpp
class OrderService {
public:
    double total(const std::vector<double>& items) {
        double sum = 0;
        for (double x : items) sum += x;
        return sum * 1.18; // tax rule (business)
    }

    void saveToDb(int orderId, double amount) {
        // SQL / DB driver calls (infra)
    }

    void printInvoice(int orderId, double amount) {
        // formatting / output rules (presentation)
    }
};
```

✅ SRP (each class has one reason to change)
```cpp
class Pricing {
public:
    double totalWithTax(const std::vector<double>& items) const {
        double sum = 0;
        for (double x : items) sum += x;
        return sum * 1.18; // business rule lives here
    }
};

class OrderRepository {
public:
    void save(int orderId, double amount) {
        // persistence reason to change
    }
};

class InvoiceRenderer {
public:
    std::string render(int orderId, double amount) const {
        // presentation reason to change
        return "Order#" + std::to_string(orderId) + " Amount=" + std::to_string(amount);
    }
};

class OrderService {
    Pricing pricing;
    OrderRepository repo;
    InvoiceRenderer renderer;

public:
    void checkout(int orderId, const std::vector<double>& items) {
        double amount = pricing.totalWithTax(items);
        repo.save(orderId, amount);
        auto invoice = renderer.render(orderId, amount);
        // send/print invoice somewhere
    }
};
```

**Key SRP win:** if tax changes → only `Pricing` changes.  
If DB changes → only `OrderRepository` changes.  
If invoice format changes → only `InvoiceRenderer` changes.


**Real-World Use Cases (where SRP matters most)**

* Payments: risk rules vs gateway calls vs receipt rendering
* Notification system: template rendering vs delivery vs retry policy
* Logging/Auditing: domain events vs storage vs formatting
* File processing: parsing vs validation vs persistence vs reporting
* Schedulers: schedule calculation vs job execution vs persistence vs observability


**Q. Define SRP in one line and give a concrete example.**
SRP: A class should have only one reason to change (one responsibility).

Example:
TaxCalculator only computes tax. If tax rules change → only this class changes. It should not also send emails or write to DB.

```cpp
#include <iostream>
#include <string>
#include <stdexcept>

using namespace std;

// ✅ SRP: Only tax rules live here.
// If tax rules change, ONLY this class changes.
class TaxCalculator {
public:
    double computeTax(double amount) const {
        if (amount < 0) throw invalid_argument("amount cannot be negative");
        return amount * 0.18; // 18% GST (example)
    }
};

// Separate responsibilities (NOT in TaxCalculator)
class EmailService {
public:
    void sendEmail(const string& to, const string& msg) {
        cout << "[Email] To: " << to << " | " << msg << "\n";
    }
};

class OrderRepository {
public:
    void saveOrder(const string& user, double amount, double tax, double total) {
        cout << "[DB] Saved order for " << user
             << " amount=" << amount
             << " tax=" << tax
             << " total=" << total << "\n";
    }
};

// Orchestrator: coordinates multiple responsibilities, but does not implement them.
class OrderService {
    TaxCalculator taxCalc;
    EmailService email;
    OrderRepository repo;

public:
    void placeOrder(const string& userEmail, double amount) {
        double tax = taxCalc.computeTax(amount);
        double total = amount + tax;

        repo.saveOrder(userEmail, amount, tax, total);
        email.sendEmail(userEmail, "Order placed! Total = " + to_string(total));
    }
};

int main() {
    OrderService service;
    service.placeOrder("ashok@example.com", 1000.0);
    return 0;
}
```


**Q. In a “UserService” class, what are 3 common responsibilities people mistakenly mix?**
* **Business rules** (signup rules, password policy, eligibility, user state transitions)
* **Infrastructure/IO** (DB persistence, external API calls, file/queue operations)
* **Presentation/transport concerns** (DTO mapping, JSON formatting, HTTP status/error translation)

**Q. How would you refactor a class that both:**
- validates input
- calls external API
- persists results
We do that by Split by responsibility:
- **Validator**: pure validation (no IO)
- **ExternalClient**: only API communication
- **Repository**: only persistence
- **Orchestrator service**: coordinates them (thin)

**Sketch:**
- `InputValidator.validate(request)`
- `ExternalKycClient.verify(request)`
- `UserRepository.save(result)`
- `UserOnboardingService.onboard()` = calls the above in order

This keeps business flow in one place but isolates change-prone details.


**Q. SRP vs “too many classes”: how do you decide the right boundary?**
Use these rules:
- **Different stakeholders / reasons to change?** Split.
- **Different change frequency?** Split (business rules change often, DB/client changes differently).
- **Can you unit test core logic without mocks?** If not, you likely need a split.
- **If two parts always change together**, keep them together.

A good boundary is where you can swap/modify one concern without touching the others.

Q. Where to place logging? service, repo, or separate decorator? why?

<u><b>Service layer</b></u>
Put logs here when they describe **business outcomes** and decisions.
✅ Good: high-signal, low-volume logs  
❌ Avoid: logging every tiny step (“entered method”, “exiting method”) everywhere

<u><b>Repo layer</b></u>
Put logs here for IO-level failures and latency:
✅ Good: log **errors**, **timeouts**, **retries**, **slow operations**  
⚠️ Be careful with PII and large payloads

<u><b>Separate decorator</b></u>
This is the best place for consistent logs like:
- requestId / correlationId
- latency
- status (success/failure)
- exception mapping
- standard fields (service, endpoint, tenant, traceId)
**Why:** It avoids duplication and guarantees uniform logging across endpoints.
✅ This is what mature systems rely on for “every request has a log line” consistency.

**Quick rule of thumb**
- If the log answers **“what did the business operation do?”** → **Service**
- If it answers **“why did IO fail / how slow was it?”** → **Repo/Client**
- If it answers **“how did this request behave end-to-end?”** → **Decorator/Middleware**

#### OCP (Open/Closed Principle) — “Add features without editing stable code

**Intuition**
OCP means:

> Your existing, tested code should be **closed for modification** but **open for extension**.

In interviews, OCP is the difference between:
- a growing **if/else ladder** (fragile, hard to test, risky)
- a design where adding a new behavior is just **adding a new class** (safe, scalable)

**Mental model:**  
If adding a new “type” (payment method, discount rule, notification channel) forces you to edit core logic → you violated OCP.

****

**Core Concepts**

-> **OCP is about stabilizing the “policy” layer**

**Policy layer** = the _core decision-making / orchestration_ of the system (the “what” and “when”).

OCP says:
- Keep this core flow **stable**
- Put the parts that vary (the “how”) into **pluggable extensions**

**Example:** `CheckoutService.placeOrder()` should not change every time you add a new payment type. The payment-specific logic should live outside it.


-> **OCP is usually implemented using polymorphism, composition, registration**

**Polymorphism (interfaces + virtual)**
Core calls an interface:
- `PaymentMethod::pay()`  
    New payment = new class implementing it.

**Composition (Strategy/Factory)**
Core _contains_ a strategy:
- `PricingStrategy`  
    Swap strategy object → behavior changes without touching core logic.

**Configuration/registration (map of handlers)**
Core looks up handler by key:
`handlers["UPI"]->pay()`  
New handler registered → core unchanged.


-> **OCP is not “never modify code”**  
You _will_ modify code sometimes. The point is:
- stable core changes rarely
- new features come via extensions

-> **Identify the axis of change**
- “Payment type changes”
- “Pricing rules change”
- “Notification channel changes”  
    Those become extension points.

**Architecture View**

❌ Non-OCP (if/else ladder grows)
```cpp
CheckoutService
  if (type == CARD) ...
  else if (type == UPI) ...
  else if (type == WALLET) ...
  else if ...
```

✅ OCP (stable caller + pluggable strategies)
```
CheckoutService  --->  PaymentMethod (interface)
                         /      |       \
                     Card     UPI     Wallet
```

**Practical Example (C++)**

❌ Non-OCP
```cpp
enum class PaymentType { CARD, UPI };

class CheckoutService {
public:
    void pay(PaymentType type, double amount) {
        if (type == PaymentType::CARD) {
            // card flow
        } else if (type == PaymentType::UPI) {
            // upi flow
        }
        // adding NETBANKING => modify this method
    }
};
```
Problem: every new payment type edits `CheckoutService` (core logic).

✅ OCP with Strategy + Polymorphism
```cpp
class PaymentMethod {
public:
    virtual ~PaymentMethod() = default;
    virtual void pay(double amount) = 0;
};

class CardPayment : public PaymentMethod {
public:
    void pay(double amount) override {
        // card payment logic
    }
};

class UpiPayment : public PaymentMethod {
public:
    void pay(double amount) override {
        // upi payment logic
    }
};

class CheckoutService {
public:
    void checkout(PaymentMethod& method, double amount) {
        // stable orchestration stays unchanged forever
        method.pay(amount);
    }
};
```

Now adding NetBanking:
```cpp
class NetBankingPayment : public PaymentMethod {
public:
    void pay(double amount) override {
        // netbanking logic
    }
};
```
✅ `CheckoutService` stays untouched.


**OCP “Interview-Grade” Upgrade: Factory + Registry (no if/else anywhere)**
```cpp
#include <unordered_map>
#include <memory>
#include <functional>
#include <string>

class PaymentMethod {
public:
    virtual ~PaymentMethod() = default;
    virtual void pay(double amount) = 0;
};

class CardPayment : public PaymentMethod {
public:
    void pay(double amount) override { /*...*/ }
};

class UpiPayment : public PaymentMethod {
public:
    void pay(double amount) override { /*...*/ }
};

class PaymentFactory {
    using Creator = std::function<std::unique_ptr<PaymentMethod>()>;
    std::unordered_map<std::string, Creator> registry;

public:
    void registerMethod(const std::string& key, Creator creator) {
        registry[key] = std::move(creator);
    }

    std::unique_ptr<PaymentMethod> create(const std::string& key) const {
        auto it = registry.find(key);
        if (it == registry.end()) throw std::invalid_argument("Unknown payment type");
        return (it->second)();
    }
};
```

Usage:
```cpp
PaymentFactory factory;
factory.registerMethod("CARD", [] { return std::make_unique<CardPayment>(); });
factory.registerMethod("UPI",  [] { return std::make_unique<UpiPayment>(); });

auto method = factory.create("CARD");
method->pay(100.0);
```
✅ Adding a new type = register it (no edits to stable code).

**Real-World Use Cases**
* Discount engine: add new discount rules without editing pricing core
* Notification system: add WhatsApp/Email/SMS without touching dispatcher
* Fraud checks: add new checks via chain/strategy
* File parsers: CSV/JSON/XML via plugin parsers
* Games: new weapons/skills without rewriting combat loop

##### Interview Questions

**Q. Define OCP in one line. What does “closed” mean practically?**

A module should allow **new behavior by adding new code**, without **editing existing stable code**.
What “closed” means practically:
For new variants (new delivery types / payment types), you should **not have to modify the core orchestration logic** (no growing `if/else/switch`); you extend via new implementations/registrations instead.

**Q. You have if/else on DeliveryType. Refactor to OCP. Which pattern?**

✅ Best typical answer: **Strategy + Factory** (or **Strategy + Registry**).
- **Strategy** = one handler per delivery type
- **Factory/Registry** = choose the right strategy based on the type

**Q. Strategy vs Factory: when do you need each?**
**Strategy** = swap an algorithm/policy at runtime behind the same interface (remove `if/else`, enable per-tenant/A-B behavior).  
**Factory** = choose and construct the right concrete implementation at creation time (hide `new` + wiring based on config/input).


**Q. How would you make OCP work when the “type” comes from DB/config?**
Use a **Registry (map of handlers)** keyed by the DB/config value:
- DB returns `"EXPRESS"`
- Registry maps `"EXPRESS" -> ExpressDeliveryHandler`
- Core does: `registry[type]->deliver(order)`

This keeps core stable and lets you add new types by **registering** new handlers.

**Q. What’s the downside of OCP overuse?**
- Too many interfaces/classes (**pattern soup**)
- Excessive indirection (harder to read/debug)
- Over-engineering for changes that may never happen
- Harder navigation and onboarding
- More boilerplate and wiring

**Rule of thumb:** apply OCP only on **real axes of change**, not hypothetical ones.

##### Common Traps (they ask these in interviews)
- **“Enum + switch” is not OCP** unless you isolate switch in one place that rarely changes.
- OCP doesn’t mean “make everything interface-based”.
- Over-abstracting early makes code unreadable and slower to change.

#### LSP (Liskov Substitution Principle) — “Derived must be usable anywhere Base is expected”

**Intuition**
LSP means:

> If code works with a **Base**, it must also work with any **Derived** _without surprises_.

So inheritance is not “code reuse”.  
Inheritance is a **behavior contract**.

If `Derived` breaks the assumptions clients rely on → inheritance is wrong (use composition or a different abstraction).

##### Core Concepts

###### Rule A — Don’t strengthen preconditions

Base allows `f(x)` for a set of inputs.  
Derived must accept **at least that same set** (can accept more, not less).

**Meaning:** Derived must not reject inputs that Base would accept.

**Example**
Base contract:
- `withdraw(amount)` is valid if `amount <= balance`
Bad Derived:
- adds a new stricter rule: `amount <= 1000`

Now code that worked with Base breaks with Derived:
```cpp
void payRent(Account& a) { a.withdraw(5000); } // valid for base (if balance allows)
```
If you pass `DerivedAccount` that caps at 1000 → it breaks substitutability.

✅ Derived can accept **more** (looser), not **less** (stricter).

###### Rule B — Don’t weaken postconditions

Base guarantees something after the call.  
Derived must guarantee **at least that much** (can guarantee more, not less).

**Meaning:** Derived must not provide fewer guarantees than Base.

Base guarantees:
- `pay()` either completes payment **or throws** if it fails.
Bad Derived:
- silently does nothing and returns success.

Now caller assumes payment happened, but it didn’t → logic breaks.

✅ Derived can guarantee **more** (extra confirmations), but not **less** (weaker outcomes).

######  Rule C — Preserve invariants

Base invariants must remain true in Derived.

**Meaning:** Derived must keep Base’s “always true” rules always true.

Example Base invariant:
- `balance >= 0` always

Bad Derived:
- overrides `withdraw()` and allows balance to go negative.

Now any code relying on `balance >= 0` breaks.
✅ Derived may add extra invariants, but must not violate base ones.

```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

class Account {
protected:
    int balance;

public:
    Account(int b) : balance(b) {
        if (balance < 0) throw invalid_argument("Negative balance not allowed");
    }

    virtual void withdraw(int amount) {
        if (amount < 0) throw invalid_argument("Negative withdraw");
        if (amount > balance) throw runtime_error("Insufficient funds");
        balance -= amount;                  // ✅ invariant preserved
    }

    int getBalance() const { return balance; }

    virtual ~Account() = default;
};

class BadOverdraftAccount : public Account {
public:
    BadOverdraftAccount(int b) : Account(b) {}

    void withdraw(int amount) override {
        // ❌ violates base invariant: balance can become negative
        balance -= amount;
    }
};

int main() {
    BadOverdraftAccount a(100);
    a.withdraw(150);
    cout << "Balance = " << a.getBalance() << "\n"; // Balance = -50 (❌ base invariant broken)
}
```

**Base invariant: balance must never be negative (`balance >= 0`)**

❌ Bad example (Derived breaks Base invariant)
```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

class Account {
protected:
    int balance;

public:
    Account(int b) : balance(b) {
        if (balance < 0) throw invalid_argument("Negative balance not allowed");
    }

    virtual void withdraw(int amount) {
        if (amount < 0) throw invalid_argument("Negative withdraw");
        if (amount > balance) throw runtime_error("Insufficient funds");
        balance -= amount;                  // ✅ invariant preserved
    }

    int getBalance() const { return balance; }

    virtual ~Account() = default;
};

class BadOverdraftAccount : public Account {
public:
    BadOverdraftAccount(int b) : Account(b) {}

    void withdraw(int amount) override {
        // ❌ violates base invariant: balance can become negative
        balance -= amount;
    }
};

int main() {
    BadOverdraftAccount a(100);
    a.withdraw(150);
    cout << "Balance = " << a.getBalance() << "\n"; // Balance = -50 (❌ base invariant broken)
}
```
**Why it’s bad:** Any code relying on `Account`’s guarantee “balance is never negative” is now broken when given `BadOverdraftAccount`.

✅ Good example (Derived preserves Base invariant)
If you want overdraft behavior, change the invariant properly:  
Instead of breaking `balance >= 0`, introduce a **new invariant**: `availableFunds = balance + overdraftLimit >= 0`.

```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

class Account {
protected:
    int balance;

public:
    Account(int b) : balance(b) {
        if (balance < 0) throw invalid_argument("Negative balance not allowed");
    }

    virtual void withdraw(int amount) {
        if (amount < 0) throw invalid_argument("Negative withdraw");
        if (amount > balance) throw runtime_error("Insufficient funds");
        balance -= amount;                  // ✅ balance stays >= 0
    }

    int getBalance() const { return balance; }

    virtual ~Account() = default;
};

class GoodOverdraftAccount : public Account {
    int overdraftLimit;

public:
    GoodOverdraftAccount(int b, int limit)
        : Account(b), overdraftLimit(limit) {}

    void withdraw(int amount) override {
        if (amount < 0) throw invalid_argument("Negative withdraw");

        // ✅ preserves base invariant: balance never goes below 0
        // Uses overdraftLimit separately, does NOT make balance negative
        if (amount <= balance) {
            balance -= amount;
        } else {
            int remaining = amount - balance;
            if (remaining > overdraftLimit) throw runtime_error("Overdraft limit exceeded");
            balance = 0;
            overdraftLimit -= remaining;
        }
    }

    int getOverdraftRemaining() const { return overdraftLimit; }
};

int main() {
    GoodOverdraftAccount a(100, 200);
    a.withdraw(150);
    cout << "Balance = " << a.getBalance() << "\n";              // Balance = 0 ✅
    cout << "Overdraft left = " << a.getOverdraftRemaining() << "\n"; // 150 ✅
}
```
**Why this is good**
- `Account` promised: **balance never negative** ✅ still true
- Derived adds extra capability **without violating base invariants**

###### Rule D — No surprise side-effects
If Base promises “doesn’t mutate X”, Derived must not mutate X.
**Meaning:** Derived must not introduce new important side-effects that Base promised not to do.

##### Architecture View

Client should depend only on Base contract:

```
Client ---> Base (contract)
              ^
              |
           Derived (must honor Base contract)
```
If Derived can’t honor it → hierarchy is wrong.

##### Classic LSP break

❌ Bad design (breaks LSP)
```cpp
#include <iostream>
#include <cassert>
using namespace std;

class Rectangle {
protected:
    int w, h;

public:
    Rectangle(int w, int h) : w(w), h(h) {}

    virtual void setWidth(int newW)  { w = newW; }
    virtual void setHeight(int newH) { h = newH; }

    int width()  const { return w; }
    int height() const { return h; }
    int area()   const { return w * h; }

    virtual ~Rectangle() = default;
};

class Square : public Rectangle {
public:
    Square(int side) : Rectangle(side, side) {}

    // To remain a square, we must keep w == h
    void setWidth(int newW) override  { w = h = newW; }
    void setHeight(int newH) override { w = h = newH; }
};

// Code written for Rectangle's contract: width/height independent
void makeIt5x4(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);
    // Expected by the Rectangle contract:
    assert(r.width() == 5);
    assert(r.height() == 4);
    assert(r.area() == 20);
}

int main() {
    Rectangle r(2, 3);
    makeIt5x4(r); // ✅ OK

    Square s(2);
    makeIt5x4(s); // ❌ Assertion fails (area becomes 16, width/height both become 4)
}
```

**Why this breaks LSP**

Because `Square` **changes the meaning** of `setWidth` / `setHeight`:
- `Rectangle::setWidth(x)` promises: **width becomes x, height unchanged**
- `Square::setWidth(x)` actually changes **both** width and height

That’s a **contract violation / surprise side-effect**, so callers that rely on the base contract break.


**Ways to fix it (realistic options)**


**Fix 1: Don’t use inheritance here (preferred)**
Make both be “Shapes”, not “Square is-a Rectangle”.
```cpp
#include <iostream>
#include <memory>
#include <vector>
using namespace std;

class Shape {
public:
    virtual int area() const = 0;
    virtual ~Shape() = default;
};

class Rectangle : public Shape {
    int w, h;
public:
    Rectangle(int w, int h) : w(w), h(h) {}
    int area() const override { return w * h; }
};

class Square : public Shape {
    int side;
public:
    Square(int s) : side(s) {}
    int area() const override { return side * side; }
};

int main() {
    vector<unique_ptr<Shape>> shapes;
    shapes.push_back(make_unique<Rectangle>(5, 4));
    shapes.push_back(make_unique<Square>(4));

    for (auto& s : shapes) cout << s->area() << "\n";
}
```
✅ LSP holds: any `Shape*` works uniformly.  
✅ No fake “Rectangle” contract imposed on square.


**Fix 2: Use composition (if you want reuse)**

Square has a rectangle internally, but does not pretend to be one.
```cpp
class RectangleData {
    int w, h;
public:
    RectangleData(int w, int h) : w(w), h(h) {}
    int area() const { return w * h; }
};

class Square {
    RectangleData rect; // reuse implementation
public:
    Square(int s) : rect(s, s) {}
    int area() const { return rect.area(); }
};
```
✅ Reuse code  
✅ No LSP promise violated because Square isn’t substitutable for Rectangle anymore


**Fix 3: Redesign the base contract (if you must keep hierarchy)**

If you truly want a hierarchy, don’t expose an API that assumes independent resizing.

Example: make `Rectangle` immutable or only allow resizing as a _single operation_ with a clear contract.

```cpp
class Rectangle {
    int w, h;
public:
    Rectangle(int w, int h) : w(w), h(h) {}
    int area() const { return w * h; }
    // No setWidth/setHeight => no independent-resize contract to violate
};
```
Or split interfaces:
- `Shape` (area)
- `IndependentlyResizableRectangle` (setWidth/setHeight)  
	Square implements `Shape`, not the resizable-rectangle interface.

✅ This preserves LSP by making contracts honest.

##### C++-Specific LSP Pitfalls (Interview favorites)
```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <stdexcept>
#include <string>

using namespace std;

// ============================================================
// 1) Object slicing (stores Base by value -> Derived sliced)
// ============================================================
class Animal {
public:
    virtual void speak() const { cout << "Animal\n"; }
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void speak() const override { cout << "Dog\n"; }
};

void demoObjectSlicing() {
    cout << "\n--- 1) Object slicing ---\n";

    vector<Animal> v;      // stores BY VALUE
    v.push_back(Dog{});    // Dog part sliced off

    v[0].speak();          // prints "Animal" (polymorphism lost)

    // ✅ Fix: store pointers (smart pointers)
    vector<unique_ptr<Animal>> p;
    p.push_back(make_unique<Dog>());
    p[0]->speak();         // prints "Dog"
}

// ============================================================
// 2) Wrong override due to signature mismatch (silent overload)
// ============================================================
class Base {
public:
    virtual void f(int) const { cout << "Base::f(int)\n"; }
    virtual ~Base() = default;
};

class DerivedWrong : public Base {
public:
    // ❌ Not overriding! This is an overload (different signature)
    void f(double) const { cout << "DerivedWrong::f(double)\n"; }
};

class DerivedCorrect : public Base {
public:
    // ✅ Correct override; compiler checks signature
    void f(int) const override { cout << "DerivedCorrect::f(int)\n"; }
};

void demoWrongOverride() {
    cout << "\n--- 2) Wrong override (signature mismatch) ---\n";

    Base* b1 = new DerivedWrong();
    b1->f(10);   // calls Base::f(int), NOT DerivedWrong
    delete b1;

    Base* b2 = new DerivedCorrect();
    b2->f(10);   // calls DerivedCorrect::f(int)
    delete b2;

    // Interview rule: ALWAYS write override.
    // If you wrote: void f(double) override {} -> compiler error (good!)
}

// ============================================================
// 3) Strengthening preconditions in Derived (reject valid inputs)
// ============================================================
class Account {
protected:
    int balance;

public:
    explicit Account(int b) : balance(b) {}

    // Contract: withdraw allowed for any amount <= balance
    virtual void withdraw(int amount) {
        if (amount <= 0 || amount > balance) throw runtime_error("Invalid withdraw");
        balance -= amount;
        cout << "Account withdraw OK. Balance=" << balance << "\n";
    }

    virtual ~Account() = default;
};

class LimitedAccount : public Account {
public:
    LimitedAccount(int b) : Account(b) {}

    // ❌ Strengthens precondition: now amount must also be <= 1000
    void withdraw(int amount) override {
        if (amount > 1000) throw runtime_error("Limit exceeded (<=1000)");
        Account::withdraw(amount);
    }
};

void payRent(Account& a) {
    a.withdraw(3000);  // valid for Account if balance allows
}

void demoStrengthenedPrecondition() {
    cout << "\n--- 3) Strengthening preconditions ---\n";

    Account a(5000);
    payRent(a); // ✅ OK

    LimitedAccount la(5000);
    try {
        payRent(la);   // ❌ breaks substitutability (throws for a “valid” base-case)
    } catch (const exception& e) {
        cout << "LimitedAccount broke LSP: " << e.what() << "\n";
    }
}

// ============================================================
// 4) Throwing new kinds of failures (surprise failure modes)
// ============================================================
// Base contract: "returns false on failure" (caller expects no exceptions)
class Parser {
public:
    virtual bool parse(const string& s) const {
        return !s.empty(); // false means "couldn't parse"
    }
    virtual ~Parser() = default;
};

class ThrowingParser : public Parser {
public:
    // ❌ Changes contract: throws on normal failure instead of returning false
    bool parse(const string& s) const override {
        if (s.empty()) throw runtime_error("Empty input"); // surprise failure mode
        return true;
    }
};

void demoSurpriseFailures() {
    cout << "\n--- 4) Surprise failure modes ---\n";

    Parser base;
    cout << "Base parse empty -> " << base.parse("") << " (no throw)\n";

    ThrowingParser tp;
    Parser& p = tp; // use polymorphically

    try {
        cout << "Derived parse empty -> ";
        cout << p.parse("") << "\n"; // ❌ throws for a case base handled normally
    } catch (const exception& e) {
        cout << "threw: " << e.what() << " (LSP break)\n";
    }
}

int main() {
    demoObjectSlicing();
    demoWrongOverride();
    demoStrengthenedPrecondition();
    demoSurpriseFailures();
    return 0;
}
```

```cpp
//Fix for 3
// =========================================================
#include <iostream>
#include <memory>
#include <stdexcept>
using namespace std;

class WithdrawalPolicy {
public:
    virtual void check(int amount, int balance) const = 0;
    virtual ~WithdrawalPolicy() = default;
};

class NoLimitPolicy : public WithdrawalPolicy {
public:
    void check(int amount, int balance) const override {
        if (amount <= 0 || amount > balance) throw runtime_error("Invalid withdraw");
    }
};

class DailyLimitPolicy : public WithdrawalPolicy {
    int limit;
public:
    explicit DailyLimitPolicy(int limit) : limit(limit) {}

    void check(int amount, int balance) const override {
        if (amount <= 0 || amount > balance) throw runtime_error("Invalid withdraw");
        if (amount > limit) throw runtime_error("Daily limit exceeded");
    }
};

class Account {
    int balance;
    unique_ptr<WithdrawalPolicy> policy;

public:
    Account(int bal, unique_ptr<WithdrawalPolicy> p)
        : balance(bal), policy(std::move(p)) {}

    void withdraw(int amount) {
        policy->check(amount, balance);   // policy decides constraints
        balance -= amount;
    }

    int getBalance() const { return balance; }
};

void payRent(Account& a) {
    a.withdraw(3000);
}

int main() {
    Account normal(5000, make_unique<NoLimitPolicy>());
    payRent(normal); // ✅ OK

    Account limited(5000, make_unique<DailyLimitPolicy>(1000));
    try { payRent(limited); } catch (const exception& e) {
        cout << "Policy blocked rent: " << e.what() << "\n";
    }
}

 // Fix for 4. Throwing “new kinds” of failures (surprise exceptions)
 
#include <iostream>
#include <stdexcept>
#include <string>
using namespace std;

class Parser {
public:
    // Contract: returns false on failure, never throws
    virtual bool parse(const string& s) const noexcept {
        return !s.empty();
    }
    virtual ~Parser() = default;
};

class StrictParser : public Parser {
public:
    bool parse(const string& s) const noexcept override {
        try {
            if (s.empty()) return false;

            // internal logic might throw:
            if (s == "boom") throw runtime_error("bad format");

            return true;
        } catch (...) {
            return false; // ✅ preserve base behavior: no exceptions leak out
        }
    }
};

int main() {
    Parser* p = new StrictParser();

    cout << p->parse("") << "\n";      // false
    cout << p->parse("ok") << "\n";    // true
    cout << p->parse("boom") << "\n";  // false (no throw)

    delete p;
}

```
######  What to remember (interview one-liners)
- **Slicing:** don’t store polymorphic types by value → use pointers/smart pointers.
- **Override mismatch:** always write `override` so compiler catches signature bugs.
- **Preconditions:** derived must not reject inputs base accepts.
- **Failures:** derived must not introduce surprise failure modes vs base contract.

##### Real-World Use Cases (where LSP matters a lot)
- **Payment methods**: every `PaymentMethod::pay()` must honor the same guarantees (execute/failed clearly).
- **Storage backends**: `KVStore::get()` must behave consistently across Redis/InMemory/DB.
- **Notification channels**: `send()` must respect reliability semantics expected by caller (sync/async, retries, failure reporting).
- **Caching layers**: `Cache` derived types must preserve TTL/invalidation assumptions.
- **State machines**: each state class must honor transition rules expected by context.

##### Interview-Level Practice Questions

**Q. Define LSP in one line.**
If a function works with `Base`, it should work the same way with any `Derived`—no surprises, no special cases.

**Q. Why Square : Rectangle is an LSP violation?**
Because the common `Rectangle` API implies **width and height can be set independently**:
- Base expectation: `setWidth(5)` changes only width, keeps height.
- Square must keep `width == height`, so `setWidth(5)` also changes height.

So code written for `Rectangle` breaks:

```cpp
void make5x4(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);
    // expected area = 20
}
```
Pass a `Square`, you get `4x4` (area 16).  
That’s a **contract/behavior change**, so substitution fails.


**Q. What is object slicing? How do you prevent it in C++?**
**Object slicing:** when a `Derived` object is copied into a `Base` **by value**, the derived part is cut off, and polymorphism is lost.

Example:
```cpp
std::vector<Base> v;
v.push_back(Derived{}); // slicing
```

**Prevent it:**
- Store **pointers/references**, not values:
```cpp
std::vector<std::unique_ptr<Base>> v;
v.push_back(std::make_unique<Derived>());
```
(or `std::vector<std::shared_ptr<Base>>`, or references via wrapper types)

**Q. Give an example of “strengthened precondition” in Derived.**
```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

class Account {
protected:
    int balance;

public:
    explicit Account(int b) : balance(b) {}

    // Base contract: valid for any amount <= balance
    virtual void withdraw(int amount) {
        if (amount <= 0 || amount > balance) throw runtime_error("Invalid withdraw");
        balance -= amount;
        cout << "Account withdrew " << amount << ", balance=" << balance << "\n";
    }

    virtual ~Account() = default;
};

class LimitedAccount : public Account {
public:
    explicit LimitedAccount(int b) : Account(b) {}

    // ❌ Strengthened precondition: additionally requires amount <= 1000
    void withdraw(int amount) override {
        if (amount > 1000) throw runtime_error("Limit exceeded (<= 1000)");
        Account::withdraw(amount);
    }
};

// Function written to the Base contract:
void payRent(Account& a) {
    a.withdraw(3000); // valid for Account if balance >= 3000
}

int main() {
    Account normal(5000);
    payRent(normal); // ✅ OK

    LimitedAccount limited(5000);
    try {
        payRent(limited); // ❌ LSP break: throws for a case Base allows
    } catch (const exception& e) {
        cout << "LSP violated: " << e.what() << "\n";
    }
}
```
**Why this is “strengthened precondition”**
- Base accepts: `amount <= balance` → (3000 is valid when balance=5000)
- Derived accepts: `amount <= balance AND amount <= 1000` → (3000 rejected)

So **Derived narrows the allowed input set**, which breaks substitutability.


**Q. When should you replace inheritance with composition?**
Replace inheritance with composition when:
- Derived **can’t honor the base contract** (preconditions/postconditions/invariants/side-effects differ).
- You’re inheriting mainly for **code reuse**, not true “is-a”.
- You need **swappable behavior** (policies/strategies) rather than a strict subtype.
- You want to avoid fragile base-class coupling and deep hierarchies.

**Rule of thumb:**

> If `Derived` is not safely usable everywhere `Base` is expected, don’t inherit—compose.

#### ISP (Interface Segregation Principle) — “Don’t force clients to depend on what they don’t use”

**Intuition**

ISP means:
Clients should not be forced to implement or depend on methods they don’t need. So instead of one fat interface, create **small role-based interfaces**.

##### Why ISP matters in LLD interviews

###### Avoid huge “God interfaces”

A God interface tries to cover every possible capability in one place.

Bad:
```cpp
class UserStore {
public:
    virtual void saveUser(...) = 0;
    virtual void deleteUser(...) = 0;
    virtual void sendEmail(...) = 0;
    virtual void exportCsv(...) = 0;
    virtual void auditLog(...) = 0;
};
```
Now any implementation of UserStore must implement everything — even if it’s irrelevant.

✅ ISP fix: split into roles
- `UserRepository` (save/delete)
- `UserExporter` (export)
- `Notifier` (send)
- `AuditLogger` (audit)


###### Avoid dummy / no-op implementations

When an interface is too big, some implementations don’t support many methods, so they do:
- `throw runtime_error("not supported")`
- `return nullptr;`
- empty function `{}`

That’s a code smell: **the interface is lying**.

Bad:
```cpp
class FileNotifier : public Notifier {
public:
    void sendEmail(...) override { throw runtime_error("not supported"); }
    void sendSMS(...) override { /* ok */ }
};
```
✅ ISP fix: split:
- `EmailSender`
- `SmsSender`

Then `FileNotifier` wouldn’t exist; you’d have `SmsSender` only.

###### Avoid brittle inheritance trees
A large interface often pushes you toward inheritance hacks:
- create a base class with default implementations
- derived classes override only some
- others remain no-op / throw

This creates a fragile hierarchy where behavior depends on what got overridden.

✅ ISP prevents this by keeping each capability separate so you don’t need inheritance just to “fill in the blanks”.

**Brittle inheritance tree (ISP broken)**

**Problem: One “God interface” forces a deep hierarchy with default + throw/no-op**

```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

class DataStore {
public:
    virtual void save(const string& key, const string& value) = 0;
    virtual string load(const string& key) = 0;
    virtual void remove(const string& key) = 0;

    // "extra features" not all stores support
    virtual void beginTx() = 0;
    virtual void commitTx() = 0;

    virtual ~DataStore() = default;
};

// Base class tries to "help" by giving defaults
class DataStoreBase : public DataStore {
public:
    void beginTx() override { throw runtime_error("Transactions not supported"); }
    void commitTx() override { throw runtime_error("Transactions not supported"); }
};

class InMemoryStore : public DataStoreBase {
public:
    void save(const string&, const string&) override { cout << "InMemory save\n"; }
    string load(const string&) override { return "value"; }
    void remove(const string&) override { cout << "InMemory remove\n"; }
};

class SqlStore : public DataStoreBase {
public:
    void save(const string&, const string&) override { cout << "SQL save\n"; }
    string load(const string&) override { return "value"; }
    void remove(const string&) override { cout << "SQL remove\n"; }

    void beginTx() override { cout << "SQL beginTx\n"; }
    void commitTx() override { cout << "SQL commitTx\n"; }
};

void doWork(DataStore& store) {
    store.save("k", "v");
    store.beginTx();   // Caller assumes tx exists (because interface has it!)
    store.commitTx();
}

int main() {
    SqlStore sql;
    doWork(sql); // ✅ ok

    InMemoryStore mem;
    try {
        doWork(mem); // ❌ runtime crash because tx throws
    } catch (const exception& e) {
        cout << "Brittle behavior: " << e.what() << "\n";
    }
}
```
**Why this is “brittle”**
- The interface promises “transactions exist”
- But some implementations can’t support it
- So we hide it behind inheritance + default throwing methods
- Code compiles, then fails at runtime depending on which derived type you pass
    ✅ **That’s brittle substitutability**.


✅ ISP fix (split interfaces, no brittle tree)
```cpp
#include <iostream>
#include <string>
using namespace std;

class KeyValueStore {
public:
    virtual void save(const string& key, const string& value) = 0;
    virtual string load(const string& key) = 0;
    virtual void remove(const string& key) = 0;
    virtual ~KeyValueStore() = default;
};

class Transactional {
public:
    virtual void beginTx() = 0;
    virtual void commitTx() = 0;
    virtual ~Transactional() = default;
};

class InMemoryStore : public KeyValueStore {
public:
    void save(const string&, const string&) override { cout << "InMemory save\n"; }
    string load(const string&) override { return "value"; }
    void remove(const string&) override { cout << "InMemory remove\n"; }
};

class SqlStore : public KeyValueStore, public Transactional {
public:
    void save(const string&, const string&) override { cout << "SQL save\n"; }
    string load(const string&) override { return "value"; }
    void remove(const string&) override { cout << "SQL remove\n"; }

    void beginTx() override { cout << "SQL beginTx\n"; }
    void commitTx() override { cout << "SQL commitTx\n"; }
};

void doWork(KeyValueStore& store) {
    store.save("k", "v"); // works for all stores
}

void doTransactionalWork(Transactional& tx) {
    tx.beginTx();
    tx.commitTx();
}

int main() {
    InMemoryStore mem;
    SqlStore sql;

    doWork(mem); // ✅
    doWork(sql); // ✅

    doTransactionalWork(sql); // ✅
    // doTransactionalWork(mem); // ❌ won't compile (perfect!)
}
```
**Why this fixes brittleness**
- A store that doesn’t support transactions **does not pretend it does**
- Transaction-required code depends on `Transactional`
- Errors become **compile-time**, not runtime surprises
- No need for hacky “base class with default throw/no-op”

Fat interfaces force inheritance trees that “fill blanks” with throw/no-op defaults, making behavior depend on which derived class you got—ISP fixes this by splitting capabilities into separate small interfaces.

###### Avoid untestable designs
Big interfaces force tests to mock a lot of unrelated stuff.

Example: You want to test `OrderService` but it depends on `MegaSystem` with 20 methods.  
Now your test doubles need to implement all 20.

✅ With ISP, `OrderService` depends only on what it needs:

- `PaymentGateway`
- `OrderRepository`
	So mocking becomes easy and focused.

##### Mental model:
If implementing an interface makes you write `throw`, `return nullptr`, or “not supported” in multiple methods → ISP is already broken.

Because those implementations are telling you:
- “I don’t actually support this contract”
- meaning the interface has too many responsibilities.

##### Core Concepts (Rules you actually use)

###### Prefer role-based interfaces
Split by “client role”, not by class hierarchy.
Example roles:
- `Readable`
- `Writable`
- `Closable`  
    instead of one `File` with 15 methods.

###### ISP prevents LSP violations
If a class can’t support a method meaningfully, it’ll violate LSP when forced.

###### Interfaces should be minimal and stable
Small interfaces:
- are easier to mock/test
- change less often
- reduce coupling

###### Composition beats inheritance
Make a class implement multiple small interfaces if needed.


##### Architecture View

❌ Fat interface (forces everyone to implement everything)
```
IWorker
  - work()
  - eat()
  - sleep()
```

✅ Segregated (roles)
```
IWorkable -> work()
IEatable  -> eat()
ISleepable-> sleep()

Robot: IWorkable
Human: IWorkable + IEatable + ISleepable
```

##### Practical Example (C++)

❌ Non-ISP design
```cpp
class IPrinter {
public:
    virtual ~IPrinter() = default;
    virtual void print(const std::string& doc) = 0;
    virtual void scan(const std::string& doc) = 0;
    virtual void fax(const std::string& doc) = 0;
};

class BasicPrinter : public IPrinter {
public:
    void print(const std::string& doc) override {
        // ok
    }
    void scan(const std::string&) override {
        throw std::runtime_error("Not supported"); // ISP violation signal
    }
    void fax(const std::string&) override {
        throw std::runtime_error("Not supported"); // ISP violation signal
    }
};
```
**Why this is bad:**
- forces fake behavior
- breaks substitutability (client calls scan, gets runtime explosion)
- tests become messy

****
✅ ISP refactor (role interfaces)
```cpp
class IPrintable {
public:
    virtual ~IPrintable() = default;
    virtual void print(const std::string& doc) = 0;
};

class IScannable {
public:
    virtual ~IScannable() = default;
    virtual void scan(const std::string& doc) = 0;
};

class IFaxable {
public:
    virtual ~IFaxable() = default;
    virtual void fax(const std::string& doc) = 0;
};

class BasicPrinter : public IPrintable {
public:
    void print(const std::string& doc) override {
        // print
    }
};

class MultiFunctionPrinter : public IPrintable, public IScannable, public IFaxable {
public:
    void print(const std::string& doc) override { /*...*/ }
    void scan(const std::string& doc) override  { /*...*/ }
    void fax(const std::string& doc) override   { /*...*/ }
};
```
Now:
- no unsupported methods
- client depends only on what it needs

##### Notification System
```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <memory>

using namespace std;

// ---------------- ISP: small, role-based interfaces ----------------

class ITemplateRenderer {
public:
    virtual string render(const string& templateName,
                          const unordered_map<string, string>& data) const = 0;
    virtual ~ITemplateRenderer() = default;
};

class INotificationSender {
public:
    virtual void send(const string& to, const string& message) = 0;
    virtual ~INotificationSender() = default;
};

// ---------------- Concrete Renderer ----------------

class SimpleTemplateRenderer : public ITemplateRenderer {
public:
    // Very simple template engine: replaces {{key}} with value
    string render(const string& templateName,
                  const unordered_map<string, string>& data) const override {
        string tpl;

        if (templateName == "WELCOME") {
            tpl = "Hi {{name}}, welcome to ColourVision! Your OTP is {{otp}}.";
        } else if (templateName == "ORDER") {
            tpl = "Hi {{name}}, your order {{orderId}} is confirmed. Total: {{total}}.";
        } else {
            tpl = "Unknown template";
        }

        for (const auto& [k, v] : data) {
            string needle = "{{" + k + "}}";
            // replace all occurrences
            size_t pos = 0;
            while ((pos = tpl.find(needle, pos)) != string::npos) {
                tpl.replace(pos, needle.size(), v);
                pos += v.size();
            }
        }
        return tpl;
    }
};

// ---------------- Concrete Senders ----------------

class EmailSender : public INotificationSender {
public:
    void send(const string& to, const string& message) override {
        cout << "[EMAIL] To: " << to << "\n"
             << "Message: " << message << "\n\n";
    }
};

class SmsSender : public INotificationSender {
public:
    void send(const string& to, const string& message) override {
        cout << "[SMS] To: " << to << "\n"
             << "Message: " << message << "\n\n";
    }
};

// ---------------- Orchestrator (composes roles) ----------------

class NotificationService {
    const ITemplateRenderer& renderer;
    INotificationSender& sender;

public:
    NotificationService(const ITemplateRenderer& r, INotificationSender& s)
        : renderer(r), sender(s) {}

    void notify(const string& to,
                const string& templateName,
                const unordered_map<string, string>& data) {
        string msg = renderer.render(templateName, data);
        sender.send(to, msg);
    }
};

int main() {
    SimpleTemplateRenderer renderer;
    EmailSender email;
    SmsSender sms;

    unordered_map<string, string> data{
        {"name", "Ashok"},
        {"otp", "123456"},
        {"orderId", "ORD-98"},
        {"total", "₹1499"}
    };

    // Same renderer + different senders (mix & match)
    NotificationService emailNotifications(renderer, email);
    NotificationService smsNotifications(renderer, sms);

    emailNotifications.notify("ashok@example.com", "WELCOME", data);
    smsNotifications.notify("+91-99999-00000", "ORDER", data);

    return 0;
}
```

##### Real-World Use Cases
- **Storage systems**: `ReadableStore`, `WritableStore`, `DeletableStore`
- **Cache**: `Gettable`, `Puttable`, `Evictable`
- **DB layer**: `TransactionManager` separate from `Repository`
- **Device drivers**: print-only vs print+scan vs fax
- **APIs**: split admin APIs vs user APIs vs public APIs

##### Interview Questions

**Q. Define ISP in one line. What’s the smell that tells you ISP is violated?**
Clients shouldn’t be forced to depend on methods they don’t use; split fat interfaces into small role-based interfaces. If implementing an interface makes you write throw not supported, no-op, or return dummy/null for multiple methods, the interface is too big → ISP is broken.

**Q. How does ISP prevent LSP violations? Give an example.**
When interfaces are small, an implementation can fully honor the contract. You avoid “fake implementations” (throw/no-op), which would break substitutability.

**Example:** God interface `Machine { print(), scan(), fax() }`
- `SimplePrinter` can’t scan/fax → it throws → violates LSP when used as `Machine`.
ISP fix:
- `IPrinter`, `IScanner`, `IFax`
- `SimplePrinter : IPrinter` only  
    Now no one can pass `SimplePrinter` where `IScanner` is required → **compile-time safety**, no LSP surprises.


**Example:** God interface `Machine { print(), scan(), fax() }`

- `SimplePrinter` can’t scan/fax → it throws → violates LSP when used as `Machine`.
    

ISP fix:

- `IPrinter`, `IScanner`, `IFax`
- `SimplePrinter : IPrinter` only  
    Now no one can pass `SimplePrinter` where `IScanner` is required → **compile-time safety**, no LSP surprises.


**Q. If too many interfaces are created, what’s the downside? How do you balance?**
**Downside of too many interfaces**
- “abstraction soup”: hard to navigate/read
- more wiring/DI complexity
- too many small files/classes
- harder onboarding
How to balance
- Split only on **real axes of change**:
    - multiple implementations exist or are likely
    - different clients use different subsets
    - you see `not supported` / no-op methods
- Keep cohesive groups together (don’t split for the sake of it)

**Rule of thumb:**

> Split when different clients depend on different subsets OR implementations cannot support all methods honestly.


Q. **In a “FileSystem” LLD, what interfaces would you segregate and why?**
Separate roles by **capabilities**:
1. **Readable**
`read(path) -> bytes`
2. **Writable**
`write(path, bytes) append(path, bytes)`
3. **DirectoryLister**
`list(dirPath) -> vector<Path>`
4. **MetadataProvider**
`stat(path) -> size, timestamps, type`
5. **PermissionManager (optional)**
`chmod/chown/acl checks`
6. **PathOps / FileOps**
`move, copy, rename, delete`

**Why:** A read-only filesystem (like ISO image / S3 readonly) shouldn’t implement write APIs. A “Metadata-only indexer” shouldn’t implement read/write.

##### Quick ISP Checklist
- Does any implementer have to write **“not supported”**?
- Do clients import headers for methods they never call?
- Are you mocking 10 methods just to test 1?  
    If yes → split interfaces.


#### DIP (Dependency Inversion Principle) — “High-level policy must not depend on low-level details

**Intuition**

In real systems, **business rules (policy)** should stay stable, while **details** (DB, network, files, vendor SDKs) change often.

DIP says:
> **High-level modules** (core logic) should not depend on **low-level modules** (details).  
> Both should depend on **abstractions**.

This is what makes LLD:
- extensible (swap implementations)
- testable (mock/fake details)
- maintainable (change DB/provider without rewriting business logic

Here "extensible" means easy to add new variants or replace details by extending through interfaces/strategies, without touching stable core logic.

Example
```cpp
class INotificationSender {
public:
    virtual void send(std::string to, std::string msg) = 0;
    virtual ~INotificationSender() = default;
};
```
Concrete implementations:
- `EmailSender`
- `SmsSender`
- later: `WhatsAppSender`

Your `NotificationService` depends only on `INotificationSender`, so adding WhatsApp is:
- create `WhatsAppSender : INotificationSender`
- register/inject it  
    ✅ **No change to NotificationService**.

That’s “extensible”.

“Swap implementations” meaning
Extensible also includes replacing one detail with another:
- swap `MySqlUserRepo` → `PostgresUserRepo`
- swap `StripePayment` → `RazorpayPayment`  
    without rewriting business logic.


**Common confusion:**
- **DIP** = design rule (dependency direction)
- **DI (Dependency Injection)** = one way to implement DIP (passing dependencies in)

##### Core Concepts (Rules you’ll actually use)

**Rule A — Policy depends on interfaces, not concrete classes**
- High-level code includes only interface headers.
  
**Rule B — Abstractions must not depend on details**
- Interface should not import “mysql.h”, “aws sdk”, “curl”, etc.

**Rule C — Details depend on abstractions**
- Concrete classes implement interfaces and can include vendor headers.

**Rule D — Creation is separate from usage**
- “new EmailSender()” inside business code is a smell.
- Object construction belongs to a **composition root** (main / factory / builder).

“Creation is separate from usage” — meaning
- **Usage (business code):** do the work (send notification, charge payment, save order)
- **Creation (wiring):** decide _which concrete classes_ to use and _construct them_
**Composition root** = the place where you **build and connect** the object graph.
Typically:
- `main()`
- a `Factory`
- a `Builder`
    
- your app startup / bootstrap code

Business code should **use dependencies**, not **decide and build** them.

##### Architecture View:
Bad(wrong direction):
```
OrderService  --->  MySqlOrderRepo  ---> MySQL SDK
(high-level)       (detail)
```

Good(right direction):
```
OrderService  --->  IOrderRepo  <---  MySqlOrderRepo ---> MySQL SDK
(high-level)       (abstraction)      (detail)
```

##### Practical Example (C++): Order checkout with repo abstraction

❌ Non-DIP (hard dependency, untestable)
```cpp
class MySqlOrderRepo {
public:
    void save(int orderId) { /* talks to MySQL */ }
};

class CheckoutService {
    MySqlOrderRepo repo; // hard dependency
public:
    void checkout(int orderId) {
        // business logic...
        repo.save(orderId);
    }
};
```
Problems:
- Can’t unit test without MySQL
- Switching to Postgres touches business code
- “Policy” is contaminated by “detail”

✅ DIP + Constructor Injection (clean + testable)

Interface (stable, no DB headers):
```cpp
class IOrderRepo {
public:
    virtual ~IOrderRepo() = default;
    virtual void save(int orderId) = 0;
};
```

High-Level Policy:
```cpp
class CheckoutService {
    IOrderRepo& repo; // depends on abstraction
public:
    explicit CheckoutService(IOrderRepo& r) : repo(r) {}

    void checkout(int orderId) {
        // business rules...
        repo.save(orderId);
    }
};
```

Low-Level Detail:
```cpp
class MySqlOrderRepo : public IOrderRepo {
public:
    void save(int orderId) override {
        // MySQL specific implementation
    }
};
```

Composition root(where you wire things):
```cpp
int main() {
    MySqlOrderRepo repo;
    CheckoutService service(repo);
    service.checkout(101);
}
```

Unit Test Becomes Trivial(the real payoff):
```cpp
class FakeOrderRepo : public IOrderRepo {
public:
    int lastSaved = -1;
    void save(int orderId) override { lastSaved = orderId; }
};

// test
FakeOrderRepo fake;
CheckoutService svc(fake);
svc.checkout(7);
// assert(fake.lastSaved == 7);
```

##### Real-World Use Cases

- **Payment**: `PaymentProcessor` depends on `IPaymentGateway` (Razorpay/Stripe/Mock)
- **Notifications**: depends on `INotificationSender` (Email/SMS/WhatsApp)
- **Storage**: depends on `IKeyValueStore` (Redis/InMemory)
- **Observability**: depends on `IMetrics`, `ILogger` (prod vs test)
- **Feature flags/config**: depends on `IConfigProvider` (local/env/remote)

##### Interview-Level Practice Questions
1. Define DIP in one line. How is it different from DI?
   High-level business code should depend on **interfaces/contracts**, not concrete details like DB/HTTP/SDK implementations. **DIP** is the design rule (dependency direction). **DI** is a wiring technique (constructor/setter/parameter injection) commonly used to implement DIP

2. In a “Parking Lot” LLD, identify 2 places where DIP improves testability.
   **`PricingPolicy`**
   `ParkingLotService` depends on `IPricingPolicy`

   In tests, inject a fake policy that returns fixed fee.

   `SpotAllocationStrategy`
   `ParkingLotService` depends on `ISpotAllocator`

   In tests, inject deterministic allocator (always returns spot A1) → predictable tests.
(Other acceptable ones: `IPaymentProcessor`, `ITicketRepository`, `IClock`)

3. Show how you’d switch DB implementation without touching business logic.
   Use **Dependency Inversion + Repository (port/adapter)**: business depends on an **interface**, DB implementations live behind it. Then you swap the concrete repo via **DI/factory/config**, without touching business logic.
   
4. When do you use `unique_ptr` vs reference for injected dependencies?
 * Use **reference (`T&`)** when:
    - the dependency is **owned elsewhere**
    - it must exist for the service’s lifetime
    - you want non-null guarantee
        
- Use **`unique_ptr<T>`** when:
    - the service **owns** the dependency (controls lifetime)
    - you want polymorphic ownership without raw `new/delete`
    - you may inject different implementations and store ownership
        
**Rule:** reference = _borrowed_, `unique_ptr` = _owned_.


5. Why is “new ConcreteX() inside service method” a design smell?


##### Quick DIP Checklist (use while designing)

###### “Does business code include vendor/DB/network headers? ❌”

Meaning
Your **core/business layer** should not do:
```cpp
#include <mysql_driver.h>
#include <curl/curl.h>
#include <aws/s3/S3Client.h>
```
Because that means your business logic depends directly on low-level details.

**DIP-friendly version**

Business code includes only **your own interfaces**:
```cpp
#include "IUserRepository.h"
#include "IPaymentGateway.h"
```

###### Is the high-level module constructing low-level details? ❌

Meaning
Business code shouldn’t do:
```cpp
void checkout() {
    StripeGateway gw;  // or new StripeGateway()
    gw.charge(...);
}
```
That hard-codes the implementation and kills testability and swappability.

**DIP-friendly version**
Construct dependencies in **composition root** (main/factory), and inject:
```cpp
CheckoutService svc(gateway);
```

###### Can I unit test the policy layer with fakes? ✅

**Meaning**
You should be able to test business logic without DB/network.

**Example**
- test `CheckoutService` using `FakePaymentGateway`
- test `UserService` using `InMemoryUserRepo`

If you can do that easily, DIP is working.

###### Are dependencies visible in the constructor? ✅

**Meaning**
A class should clearly declare what it needs:
```cpp
class CheckoutService {
public:
    CheckoutService(IPaymentGateway& gw, IOrderRepo& repo);
};
```

That’s good because:
- dependencies are explicit
- easy to wire correctly
- easy to mock in tests

**Bad sign:** hidden dependencies like singletons/service locators:
```cpp
auto& gw = ServiceLocator::paymentGateway(); // hidden
```

Dependencies are explicit” means:

A class clearly states what other components it needs (logger, repo, payment gateway, etc.) in its constructor or function parameters, instead of hiding them inside the class.

✅ Explicit dependency (good)

You can tell just by looking at the constructor what this class needs:
```cpp
class OrderService {
    IPaymentGateway& payment;
    IOrderRepository& repo;

public:
    OrderService(IPaymentGateway& p, IOrderRepository& r)
        : payment(p), repo(r) {}
};
```
Here, dependencies are **visible and required**.

❌ Hidden/implicit dependency (bad)

The class uses a dependency but doesn’t declare it:
```cpp
class OrderService {
public:
    void placeOrder() {
        ServiceLocator::paymentGateway().charge(100); // hidden dependency
    }
};
```
Or:
```cpp
class OrderService {
public:
    void placeOrder() {
        ServiceLocator::paymentGateway().charge(100); // hidden dependency
    }
};
```
In both cases, someone reading the class can’t easily see what it relies on.

****

Why explicit dependencies are better
- Easier to **test** (inject fake/mock)
- Easier to **swap implementations**
- Easier to **reason about** (less surprise coupling)
- Easier to **construct correctly** (no missing hidden setup)
Explicit dependencies = “you can see what the class needs from its constructor signature.”

A **constructor signature** is the constructor’s **name + parameter types (and order)** — basically the “shape” of the constructor that the compiler uses to choose which constructor to call.

```cpp
class OrderService {
public:
    OrderService(IPaymentGateway& p, IOrderRepository& r) { }
};
```
The constructor signature here is:

`OrderService(IPaymentGateway&, IOrderRepository&)`
##### One-line takeaway
These checks ensure your **policy/business code depends on abstractions**, not concrete DB/network vendors, and is easy to test by injecting fakes.

#### 2.6 Cohesion, Coupling, Law of Demeter, YAGNI vs Extensibility

##### 2.6.1 Cohesion — “Do things in one place that belong together

**Intuition**
A module is cohesive if its parts work toward one focused purpose.
High cohesion = easy to understand, easy to change, fewer bugs.


**Practical Rules**

- **Functional cohesion (best):** everything contributes to one clear goal (e.g., `PricingEngine`)
- **Temporal cohesion:** stuff done “at the same time” (often a smell: `initEverything()`)
- **Logical cohesion:** “utility” dumping ground (big smell: `Helper`, `CommonUtil`)


**Architecture smell test**
If you can rename a class from `OrderService` to `OrderServiceAndSomethingElse`, it’s low cohesion.


**Example**

❌ Low cohesion:
```cpp
class UserService {
public:
    void createUser(...);       // business
    void sendWelcomeEmail(...); // notification
    void saveToDb(...);         // persistence
};
```

✅ Higher cohesion:
```cpp
UserService (business)
UserRepo (persistence)
WelcomeNotifier (notification)
```

###### Interview Questions

**Q. What’s the difference between SRP and cohesion?**
SRP is a change-driven rule: split when different stakeholders or change reasons exist.
Cohesion is a design quality: keep related data + behavior together; high cohesion usually helps SRP, but you can have high cohesion and still violate SRP if two stakeholders change it.


**Q. Is a “Utils” class cohesive? When can it be okay?**
Where I’d allow a `Utils` (good example)
I’d allow a `Utils`-style class **only when it’s:**
- **pure + stateless** (no DB, no network, no global state)
- **cross-cutting**, used across many modules
- has **one tight theme**
- doesn’t encode business/domain rules

Example: `StringUtils` for normalization
- **Utility** = reusable helper functions (the “small tools”)
- **Interface/Abstraction** = a _boundary_ that hides external systems and enables swapping (the “plug socket”)

```cpp
#include <iostream>
#include <string>
#include <algorithm>
#include <cctype>

using namespace std;

// ✅ Allowed "Utils": pure, stateless, one theme (string helpers)
class StringUtils {
public:
    static string trim(string s) {
        auto notSpace = [](unsigned char c){ return !std::isspace(c); };

        s.erase(s.begin(), std::find_if(s.begin(), s.end(), notSpace));
        s.erase(std::find_if(s.rbegin(), s.rend(), notSpace).base(), s.end());
        return s;
    }

    static string toLower(string s) {
        std::transform(s.begin(), s.end(), s.begin(),
                       [](unsigned char c){ return std::tolower(c); });
        return s;
    }
};

// ✅ Abstraction (contract): NOT a util
class IPaymentGateway {
public:
    virtual bool charge(int amountPaise) = 0;
    virtual ~IPaymentGateway() = default;
};

// Concrete implementations (details)
class StripeGateway : public IPaymentGateway {
public:
    bool charge(int amountPaise) override {
        cout << "[Stripe] Charging " << amountPaise << " paise\n";
        return true;
    }
};

class RazorpayGateway : public IPaymentGateway {
public:
    bool charge(int amountPaise) override {
        cout << "[Razorpay] Charging " << amountPaise << " paise\n";
        return true;
    }
};

// High-level business logic: depends on abstraction, not on Stripe/Razorpay
class CheckoutService {
    IPaymentGateway& gateway;
public:
    explicit CheckoutService(IPaymentGateway& g) : gateway(g) {}

    void checkout(string userInputAmount) {
        // use a util (pure helper)
        userInputAmount = StringUtils::trim(userInputAmount);

        int amount = stoi(userInputAmount);
        if (!gateway.charge(amount)) {
            throw runtime_error("Payment failed");
        }
        cout << "Checkout success!\n";
    }
};

int main() {
    cout << StringUtils::toLower("  HeLLo WoRld  ") << "\n\n";

    StripeGateway stripe;
    RazorpayGateway razor;

    CheckoutService s1(stripe);  // "swap" by wiring: service uses Stripe
    s1.checkout(" 100 ");

    cout << "\n";

    CheckoutService s2(razor);   // "swap" by wiring: service uses Razorpay
    s2.checkout("200");

    return 0;
}
```

**Utility (Utils) = a tool**
**`StringUtils` / `HashUtils`** are like:
- a **screwdriver**
- a **measuring tape**
- a **calculator**

They’re:
- simple helpers
- don’t “connect you to a company”
- don’t have multiple vendors behind them
- just do a small operation and return

✅ Example: trimming spaces is like using a screwdriver—same tool everywhere.


##### 2.6.2 Coupling — “How much a module knows about others"

###### Intuition

Coupling is the **amount of dependency knowledge** one module has about another.

Low coupling = you can swap/modify one part without rippling changes everywhere

###### Practical Rules
- Prefer depending on **interfaces** (DIP).
- Prefer **data contracts** (simple DTOs) over passing rich objects everywhere.
- Avoid deep knowledge of another module’s internals.
- **DTO = Data Transfer Object.**

It’s a **simple data container** used to move data between layers/services, usually:
- fields only (or very light getters/setters)
- **no business logic**
- no invariants beyond basic validation
- often serializable (JSON, protobuf)

###### Coupling types (interview-relevant)

**Content coupling (worst)**
**Meaning:** One module reaches into another module’s **internal state / private layout**.

Typical signs:
- `friend` used to access private fields
- “peek” into private members via pointer tricks / offsets
- relying on memory layout, “I know `balance` is the first int”
- calling internal helper methods that weren’t meant as part of the contract

**Bad Example(friend touching internal state)**
```cpp
class Account {
    int balance = 0;  // invariant: balance >= 0
    friend class AuditTool; // ❌ content coupling
public:
    void deposit(int amt) { balance += amt; }
};

class AuditTool {
public:
    void forceSet(Account& a, int b) { a.balance = b; } // breaks invariants
};
```

**Why it’s worst:**
- Any internal change (rename `balance`, change type, add invariant) breaks other modules.
- Invariants become impossible to enforce.

✅ Fix ideas:
- Expose a **safe public API** (`getBalance()`, `withdraw()`, `applyAdjustment()` with validation)
- Move behavior into the class that owns the state (tell, don’t ask)
- Remove `friend` unless it’s truly controlled (rare: serialization, tightly paired classes)

The “content coupling” problem in your code is this:
- `AuditTool` can directly write `Account::balance`
- That bypasses **invariants** (like `balance >= 0`)
- So `Account` loses control of its own state
    

Below are **two clean fixes** (pick based on what you want AuditTool to do).

**Fix 1 (most common): Remove friend, expose a safe API**
```cpp
#include <stdexcept>

class Account {
    int balance = 0; // invariant: balance >= 0

public:
    int getBalance() const { return balance; }

    void deposit(int amt) {
        if (amt <= 0) throw std::runtime_error("invalid deposit");
        balance += amt;
    }

    void withdraw(int amt) {
        if (amt <= 0 || amt > balance) throw std::runtime_error("invalid withdraw");
        balance -= amt;
    }

    // If you need “correction”, expose it safely:
    void applyCorrection(int newBalance) {
        if (newBalance < 0) throw std::runtime_error("balance cannot be negative");
        balance = newBalance;
    }
};

class AuditTool {
public:
    void correctBalance(Account& a, int newBalance) {
        a.applyCorrection(newBalance); // ✅ goes through Account's rules
    }
};
```
✅ Why this fixes it:
- `Account` owns its state and enforces invariants.
- `AuditTool` cannot “poke internals” anymore

**Fix 2 (better design): Audit should _observe_, not modify**
If “audit” is about logging/verification, it should not change balance at all.
```cpp
#include <iostream>

class Account {
    int balance = 0;
public:
    int getBalance() const { return balance; }

    void deposit(int amt) { balance += amt; /* log event */ }
    void withdraw(int amt) { balance -= amt; /* log event */ }
};

class AuditTool {
public:
    void report(const Account& a) {
        std::cout << "Balance=" << a.getBalance() << "\n"; // ✅ read-only
    }
};
```

✅ Why this is even cleaner:
- No one except `Account` can mutate the state.
- Auditing stays read-only (common in real systems).

**Quick rule to remember**
- If another class needs to **modify** your private state → redesign the API.
- “friend to break encapsulation” is almost always a smell in interviews.

****

**Control Coupling**

**Meaning:** You pass **flags** to control _what another module does internally_.

Smell:
- `doX=true`, `mode=2`, `type="A"` passed to change behavior inside callee
- lots of `if (flag)` branching inside one function

Bad example (flag-driven behavior)
```cpp
void processOrder(const Order& o, bool sendEmail, bool persist) {
    if (persist) { /* save to DB */ }
    if (sendEmail) { /* send email */ }
}
```

**Why it’s bad:**

- Caller must know the callee’s internal options.
- Adds branching complexity; every new behavior adds another flag.
- Test explode: combination of flags.


✅ Fix options:

**Split methods** (simple + common in LLD)
```cpp
void processOrder(const Order& o);
void processOrderAndPersist(const Order& o);
void processOrderAndNotify(const Order& o);
```

**Use Strategy / composition** (scales better):
```cpp
class INotifier { public: virtual void notify(const Order&) = 0; };
class IOrderRepo { public: virtual void save(const Order&) = 0; };

class OrderService {
    INotifier& notifier;
    IOrderRepo& repo;
public:
    void process(const Order& o) { /* core */ repo.save(o); notifier.notify(o); }
};
```


****


**Data coupling** (best)

**Meaning:** Modules talk by passing **only the minimal required data**, not internal objects or “kitchen sink” context.

Good signs:
- small parameters / simple DTOs
- clear inputs/outputs
- callee doesn’t need to know caller’s internals

**Good example**
```cpp
// needs only amount + region; doesn’t need UserService, DB, etc.
double computeTax(double amount, std::string stateCode);
```

**Common pitfall: “Stamp coupling” (close to data coupling but worse)**
Passing a big struct when only one field is used:
```cpp
struct CheckoutContext { int amount; string userId; string coupon; /* 20 more */ };

void charge(const CheckoutContext& ctx) { use(ctx.amount); } // ❌ stamp coupling
```

✅ Better:
```cpp
void charge(int amount);
```

###### Interview one-liners you can say
- **Content coupling:** “Touching private state/layout = worst; breaks invariants and makes changes dangerous.”
- **Control coupling:** “Flags that steer internal behavior = caller knows too much; prefer split APIs or strategies.”
- **Data coupling:** “Pass only what’s needed; stable contracts, fewer ripple effects.”

##### Law of Demeter (LoD) — “Don’t talk to strangers”

**Intuition**
A method should only talk to:
- itself (`this`)
- its direct members
- its direct parameters
- objects it creates locally
It should NOT navigate deep object chains (“train wrecks”).


The smell
❌:
```cpp
order.getCustomer().getAddress().getCity().toUpper();
```
Why bad:
- `Order` now depends on `Customer`, `Address`, `City` structure
- changes in any intermediate class break your code
- coupling explodes


**Fix patterns (pick one)**

**Fix A: Tell, don’t ask (push behavior down)**
✅:
```cpp
order.customerCityUpper();
```
Implementation:
```cpp
class Order {
    Customer customer;
public:
    std::string customerCityUpper() const {
        return customer.cityUpper();
    }
};
```

**Fix B: Provide stable query method at right level**
✅:
```cpp
std::string city = customer.getCity();
```
 Keep the navigation inside `Customer`.


 **Fix C: Use a mapper/assembler in application layer**
If you must build a response:
- do it in a “DTO assembler” / “mapper” class, not randomly everywhere.


**Architecture View**

Bad:
```
A -> B -> C -> D (A knows everything)
```
Good:
```
A -> B (B hides C/D internally)
```


**Interview Questions**

**Q. Explain LoD and why it reduces coupling.**
Law of Demeter says that a method should only communicate with its local variables, its direct parameters, its direct members and itself. It reduces coupling by avoiding long chaining methods which might not have been taken from the direct member or might not be a util.

**Q. Is LoD always good? What’s the trade-off?**
In most of the cases LoD is always, unless we are taking about using utils. Though we should use less of the utils, until its optimal and preserve invariants and enforce encapsulation.


##### YAGNI vs Extensibility — “Don’t overbuild, but don’t paint yourself into a corner”

**Intuition**
- **YAGNI**: “You Aren’t Gonna Need It” → don’t build hypothetical features.
- **Extensibility**: design so real future change is cheap.

The real skill is: **predict likely axes of change** and only abstract those.

**Practical Rules (very interview-useful)**

**Rule 1: Abstract only the _volatile_ parts**
**Meaning:** Only create interfaces for things that are likely to change or have multiple variants.

How to spot “volatile”
- Many variants today or likely tomorrow
- Depends on vendor/external system (Stripe, Razorpay, S3)
- Policy changes frequently (pricing rules, discounts)
Example: Payment methods are volatile → interface makes sense  
But `Money` addition is not volatile → don’t abstract.

Interview Line: I abstract the axis of change, not stable primitives.

****

**Rule 2: Start simple, refactor when axis of change becomes real**
Interviewers love: “Start with simple if/else for 2 types, but isolate it so it can evolve.”
**Meaning:** With only 1–2 types, an `if/else` is fine—**as long as it’s isolated** in one place.

**Why interviewers love it**
Because it shows you’re practical:
- not overengineering early
- but you’re keeping a seam so it can evolve

#### Example architecture (simple now, extensible later)

✅ “YAGNI-friendly OCP”:
```
CheckoutService
   |
   +--> PaymentDispatcher   // only place with switch/if
             |
             +--> StripePayment
             +--> RazorpayPayment
```
Today: `PaymentDispatcher` uses a small switch.  
Later: when types grow, you refactor dispatcher into a **registry/factory** without touching `CheckoutService`.

**Interview line:**

>“I keep the if/else contained in a dispatcher so it’s a single refactor point when types grow.”

****

**Rule 3: Don’t introduce a pattern unless it removes a pain**
**Meaning:** Patterns are not goals. They’re tools to remove a specific problem.

**Acceptable reasons (what interviewers want to hear)**
- reduces duplication
- makes a change safer
- improves testability

**Bad reason**
- “Because OCP says so”
- “Because Strategy is cool”

**Interview line:**

> “I introduce Strategy/Factory only when I see duplication or frequent changes.”

****

**Rule 4: Keep seams ready, not frameworks**

A **seam** is a **small, intentional “plug point”** in your design where you can swap/extend behavior later with minimal change.

**Characteristics**
- Minimal code/abstractions
- Localized boundary (one interface, one constructor param)
- Easy to understand and test
- Doesn’t force an architecture on the whole system

**Meaning:** Prepare for extension using **simple seams**:
- small interfaces
- constructor injection
- small modules

A **framework** is a **general infrastructure layer** that defines how you build large parts of the system (lifecycle, registration, plugin loading, configuration, wiring).

**Characteristics**
- Many abstractions + conventions
- Registration/bootstrapping system
- Often “inversion of control” over big parts of the codebase
- Harder to debug, more upfront complexity
- You pay cost even when you only have 2–3 cases


Don’t build huge frameworks upfront.
**“Seam” in simple terms**
A seam is a place where you can “plug in” something later without rewriting the system.

✅ Seam example:
- `CheckoutService(IPaymentMethod&)`
- `PaymentDispatcher` separated

❌ Framework example:
- 15 interfaces + abstract factory + plugin loader on day 1

**Interview line:**

> “I keep seams via interfaces + constructor injection, but avoid building a framework until complexity demands it.”


###### Interview Questions

1. How do you balance OCP with YAGNI?
	I keep the _core policy_ clean and stable, but I don’t introduce heavy abstractions until the **axis of change is real**. With 1–2 variants, I’ll use a simple `if/else` **contained in one place** (a dispatcher/factory). When new variants start arriving or changes become frequent, I refactor that single switch into Strategy/registry.  
	So: **YAGNI controls when I abstract; OCP guides where I place the extension point.**

2. Give an example where patterns made things worse.
	**Case: “Plugin framework” for 2 payment providers**  
	A team expected many payment types and built:
	- registry + plugin loader
	- abstract factory
	- config-driven discovery
	- multiple base interfaces per “payment”

	But reality: only Stripe + Razorpay existed for months.

**One-liner you can say:**

> “Start simple, isolate the switch, refactor to polymorphism when the change axis proves itself.”

3. What’s a “seam” and why is it better than over-abstraction?
	A **seam** is the _smallest deliberate boundary_ where you can swap an implementation (DB/mock/provider) safely. **Seams beat over-abstraction** because they localize change/testing to one spot without adding unnecessary layers and indirection everywhere.

**What got worse:**
- adding a new provider required touching 6–8 files + registration rules
- debugging became painful (indirection: who created what?)
- tests became harder (mocks everywhere, too many layers)
- onboarding slowed (“learn the framework before writing features”)
- small changes took longer and caused regressions

**The simpler design would have been:**
- `IPaymentGateway`
- `StripeGateway`, `RazorpayGateway`
- one `PaymentDispatcher` with a small switch  
    …and later evolve to a registry only if providers actually grow.

**Key takeaway:**

> Patterns are cost. If they don’t remove present pain (duplication/change risk/testability), they’re negative ROI.

#### The “2.6 Cheat Sheet” (use in interviews)
- **Cohesion:** one purpose per module (internally focused)
- **Coupling:** minimize knowledge about others (externally focused)
- **LoD:** avoid deep chains; talk to direct collaborators
- **YAGNI:** build what’s needed now
- **Extensibility:** abstract only likely change points