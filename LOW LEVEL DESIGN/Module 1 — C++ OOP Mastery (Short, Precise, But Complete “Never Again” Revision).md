
#### Class Fundamentals & Object Model

Class = Data(State) + Behavior(methods) + Invariants(rules that must always hold)

Object = An object is a concrete runtime instance whose base memory layout is determined at compile time.
But its behavior and dynamic memory usage may be resolved at runtime. 

###### Access Control

- `private` → invariants live here
- `protected` → for inheritance only (use sparingly)
- `public` → stable API

##### Rule 
Public methods may change state, but must **preserve invariants**. 


```pgsql
+----------------------+
|        Class         |
|----------------------|
| private data         | <-- invariants protected
|----------------------|
| public methods       | <-- controlled access
+----------------------+

```

![[Pasted image 20260106204405.png]]

##### Can we achieve the same thing using `struct` instead of `class`?
Anything you can do with a `class`, you can do with a `struct` in C++.

<u>Correct usage</u>
**struct** - Data Only

```cpp
struct Point{
	int x;
	int y;
}
```


**class** - Behavior + rules

```cpp
class BankAccount{
	double balance;
public:
	void deposit(double amount);
	void withdraw(double amount);
}
```


In C++ and C#, struct supports private members, but not in C, Go, Rust.

* Struct fails semantically(design-wise) means the code violates the expected meaning or intent implied by its structure or keywords, even though it is technically correct.
* Good Code is not just about what is correct, it is also honest about what it represents.

**If struct can do everything class can, why not always use struct?**
This question is C++ - specific, because only in C++ structs and classes are nearly identical.

**Q. Why should data members almost always be private?**
Data members should be private to enforce encapsulation, controlling how state changes and to protect the invariants.

**Q. What is an invariant?**
Invariants are set of conditions which must be preserved throughout the object's lifecycle after construction. 

```cpp
class BankAccount {
    double balance; // invariant: balance >= 0

public:
    BankAccount(double initial) {
        if (initial < 0) throw std::invalid_argument("Invalid");
        balance = initial;
    }
	
	void withdraw(double amount){
		if(balance < amount) throw std::logic_error("Insufficient")
		balance -= amount
	}
};

```

**Q. Can a const method modify an object?**
Const method cannot modify an object's logical state but it can modify the members which are marked mutable.

```cpp
class Cache {
    mutable int hits = 0;

public:
    int getValue() const {
        hits++;    // ✅ Allowed
        return 42;
    }
};

```

- `hits` is marked `mutable`
- It is **not part of the logical state**
- Used for caching / statistics
#### Constructors and Destructors

**Q. What is a Constructor?**
A constructor is a special function that establishes the initial valid state of an object and enforce invariants.

* Same name as class.
* No return type.
* Called automatically when an object is created.

```cpp
class User {
    int age;

public:
    User(int a) {
        if (a < 0) throw std::invalid_argument("Invalid age");
        age = a;
    }
};
```

**Q. What is a Destructor?**
A destructor is a special function that cleans up resources owned by an object before it is destroyed.

* Name ~ClassName.
* No arguments, no return type.
* Called automatically.
* Used for resource release, not business logic.

```cpp
#include <iostream>

class FileLogger {
public:
    FileLogger() {
        std::cout << "File opened\n";
    }

    ~FileLogger() {
        std::cout << "File closed\n";
    }
};

int main() {
    {
        FileLogger logger;
        std::cout << "Writing logs...\n";
    } // logger goes out of scope here

    std::cout << "Program ends\n";
}
```

Output(conceptually)

```
File opened
Writing logs...
File closed
Program ends
```

**What this shows (important)**
* Constructor runs when an object is created.
* Destructor runs automatically when object goes out of scope.
* No manual call needed.
* Destructor is guaranteed to run.


**Intuition**

Constructors **establish invariants**.  
Destructors **cleanly release ownership**.

If either is wrong → memory leaks, UB(Undefined Behavior), or broken objects.


**Rules**
- Constructor **must fully initialize the object**
- If construction fails → object must not exist
- Destructors:
    - Should **never throw**
    - Must be `virtual` if class is used polymorphically

```
Construction ---> Valid Object ---> Destruction
   |                   |                |
   |-- establish --> invariants --> cleanup
```



<b><u>Types of Constructor</u></b>

<b>1. Default Constructor</b>

It takes no arguments.

```cpp
class User {
public:
    User() {
        // default initialization
    }
};
```



**2. Parameterized Constructor**

A constructor that takes parameters to initialize the object.

```cpp
class User {
    int age;
public:
    explicit User(int a) : age(a) {}
};
```

**Note -:  We use explicit keyword from preserving conversion of types implicitly.**


**3. Copy Constructor**

Creates a new object as a copy of an existing object.

```cpp
class User {
    int age;
    int id;

public:
    User(int a, int i) : age(a), id(i) {}

    // Copy constructor
    User(const User& other)
        : age(other.age), id(other.id) {}
};
```

**Usage**
```cpp
User u1(25, 101);
User u2 = u1;   // age=25, id=101 (copied)
```

📌 **Both objects now have the same state, but are independent.**

**4. Move Constructor**

```cpp
class Buffer {
    int* data;     // resource
    int size;      // metadata
    bool owns;     // ownership flag

public:
    Buffer(int s) : size(s), owns(true) {
        data = new int[s];
    }
        // Move constructor
    Buffer(Buffer&& other) noexcept
        : data(other.data),        // (1) STEAL resource
          size(other.size),        // (2) COPY metadata
          owns(other.owns) {       // (3) TRANSFER ownership flag

        other.data = nullptr;      // (4) RESET source
        other.size = 0;
        other.owns = false;
    }
```

```
- `*`(Pointer) → Where is it?
    
- `&`(Reference) → Another name for it
    
- `&&`(Temporary I can steal from) → Temporary I can steal from
```


**What is noexcept?**
It is a promise to the compiler that a function will never throw an exception and if it does the program will terminate.

**5. Delegating Constructor**

  One constructor calls the another constructor of the same class.

```cpp
class Account {
    int balance;
    bool active;

public:
    Account() : Account(0, false) {}   // delegates

    Account(int b, bool a)
        : balance(b), active(a) {}
};
```

**Usage**

```cpp
Account a1;            // balance=0, active=false
Account a2(1000, true);
```


**Real World Uses:**

1. File Handles 
	* Constructor: Establishes a database connection.
	* Destructor: Closes the file and releases the file handle.
2. Database Connections:
	* Constructor: Establishes a database connection.
	* Destructor: Closes the connection and returns it to the pool.
3. Locks(Mutex):
	* Constructor: Establishes a database connection.
	* Destructor: Closes the connection and returns it to the pool.
4. Network Sockets:
	* Constructor: Open the network socket and allocates OS resources.
	* Destructor: Closes the socket and frees OS resources.

**Q. Why shouldn't destructors throw?**
Destructors should not throw because throwing an exception during stack unwinding the C++ runtime will call **std::terminate**, leading to the termination of the program and possible resource leaks.

```cpp
#include <iostream>
#include <stdexcept>

struct Bad {
    ~Bad() {
        std::cout << "~Bad called\n";
        throw std::runtime_error("Destructor threw"); // ❌ dangerous
    }
};

int main() {
    try {
        Bad b;  // b will be destroyed when leaving scope
        throw std::runtime_error("Main exception");   // start unwinding
    } catch (...) {
        std::cout << "This will NOT execute\n";
    }
    return 0;
}
```

Output

```
/tmp/uRH9AMQSJ4/main.cpp: In destructor 'Bad::~Bad()':
/tmp/uRH9AMQSJ4/main.cpp:7:9: warning: 'throw' will always call 'terminate' [-Wterminate]
    7 |         throw std::runtime_error("Destructor threw"); // ❌ dangerous
      |         ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/tmp/uRH9AMQSJ4/main.cpp:7:9: note: in C++11 destructors default to 'noexcept'
~Bad called
terminate called after throwing an instance of 'std::runtime_error'
  what():  Destructor threw
Aborted


=== Code Exited With Errors ===
```


**Q. What happens when you add virtual to a destructor?**
Adding virtual to a destructor ensures proper cleanup by invoking the derived class destructor when an object is deleted through a base class pointer.

**Without Virtual**


```cpp
#include <iostream>
class Base {
public:
    ~Base() { std::cout << "~Base\n"; }  // ❌ not virtual
};

class Derived : public Base {
public:
    ~Derived() { std::cout << "~Derived\n"; }
};

int main() {
    Base* p = new Derived();
    delete p;   // ❌ Undefined behavior (typically only ~Base runs)
}
```

Output
```
~Base


=== Code Execution Successful ===
```

With Virtual

```cpp
#include <iostream>

class Base {
public:
    virtual ~Base() { std::cout << "~Base\n"; }  // ❌ not virtual
};

class Derived : public Base {
public:
    ~Derived() { std::cout << "~Derived\n"; }
};

int main() {
    Base* p = new Derived();
    delete p;   // ❌ Undefined behavior (typically only ~Base runs)
}

```

Output

```
~Derived
~Base


=== Code Execution Successful ===
```

What does “proper cleanup” mean?
“Proper cleanup” means:
- All destructors run
- In reverse order of construction
- All owned resources are released


**Q. What happens if a constructor throws?**
If a constructor throws, object creation fails and the object is never created. Any already-constructed sub-objects are destroyed automatically, and the exception propagates to the caller. The program terminates only if the exception is not handled.

Without Exception
```cpp
#include <iostream>
#include <stdexcept>

class User {
public:
    User(int age) {
        std::cout << "Constructor started\n";
        if (age < 0) {
            throw std::invalid_argument("Invalid age");
        }
        std::cout << "Constructor finished\n";
    }

    ~User() {
        std::cout << "Destructor called\n";
    }
};

int main() {
    User u(-5);   // ❌ constructor throws
    std::cout << "User created\n";
}
```

```
Constructor started
terminate called after throwing an instance of 'std::invalid_argument'
  what():  Invalid age
Aborted


=== Code Exited With Errors ===
```

With Exception

```cpp
#include <iostream>
#include <stdexcept>

class User {
    int age;

public:
    User(int a) {
        std::cout << "Constructor started\n";

        if (a < 0) {
            throw std::invalid_argument("Age cannot be negative");
        }

        age = a;
        std::cout << "Constructor finished\n";
    }

    ~User() {
        std::cout << "Destructor called\n";
    }
};

int main() {
    try {
        User u(-5);   // constructor throws
        std::cout << "User created\n";
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << "\n";
    }

    std::cout << "Program continues\n";
}
```

```
Constructor started
Caught exception: Age cannot be negative
Program continues


=== Code Execution Successful ===
```


#### Copy & Move Semantics (Rule of 0 / 3 / 5)

**Intuition**
Copy = duplicate ownership
Move = transfer ownership

If you manage resources → you must define behavior explicitly.

**Rule of 0**
* If no raw resources → let compiler handle everything.

**Rule of 3**
If you define one, define all
* Destructor
* Copy constructor
* Copy assignment

Copy Assignment operator:

```cpp
ClassName& operator=(const ClassName& other); // use when a = b
```

```cpp
#include <iostream>
#include <cstring>

class String {
    char* data;

public:
    // 1️⃣ Constructor
    explicit String(const char* s) {
        data = new char[strlen(s) + 1];
        strcpy(data, s);
    }

    // 2️⃣ Destructor
    ~String() {
        delete[] data;
    }

    // 3️⃣ Copy constructor
    String(const String& other) {
        data = new char[strlen(other.data) + 1];
        strcpy(data, other.data);
    }

    // 4️⃣ Copy assignment operator
    String& operator=(const String& other) {
        if (this != &other) {
            delete[] data;
            data = new char[strlen(other.data) + 1];
            strcpy(data, other.data);
        }
        return *this;
    }
};
```

**Rule of 5**
Add:
* Move Constructor
* Move Assignment

Move Assignment operator:

```cpp
ClassName& operator=(const ClassName& other);
```

```cpp
#include <iostream>
#include <cstring>

class Buffer {
    int* data;
    size_t size;

public:
    // 1️⃣ Constructor
    explicit Buffer(size_t s) : size(s) {
        data = new int[s];
    }

    // 2️⃣ Destructor
    ~Buffer() {
        delete[] data;
    }

    // 3️⃣ Copy constructor
    Buffer(const Buffer& other) : size(other.size) {
        data = new int[size];
        std::copy(other.data, other.data + size, data);
    }

    // 4️⃣ Copy assignment operator
    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new int[size];
            std::copy(other.data, other.data + size, data);
        }
        return *this;
    }

    // 5️⃣ Move constructor
    Buffer(Buffer&& other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }

    // 6️⃣ Move assignment operator
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
};
```

* **Why is move usually noexcept?**
  So that the standard library containers can safely use move operations instead of copies. If a move constructor is not noexcept, containers fall back to copying to preserve exception safety, which defeats the purpose of move semantics.

* **What breaks if move throws?**
  Containers lose exception safety guarantees and fall back to copying or fail to compile.

#### References, Pointers & Ownership
Pointers answer “where?”
References answer “who?”
Ownership answers “who deletes?”

1️⃣ `std::unique_ptr` → **Sole ownership**

**Meaning (simple)**
**Only one owner exists. When the owner dies, the resource is destroyed.**
- Cannot be copied
- Can be moved
- Models **exclusive ownership**

🏠 Real-world analogy: **House key**
- You own a house key
- Only **one person** has it
- If you give it away, **you no longer have it**
- When the key is destroyed, access is gone


2️⃣ `std::shared_ptr` → **Shared ownership**

**Meaning (simple)**
**Multiple owners exist. Resource lives until the last owner goes away.**
- Reference counted
- Copyable
- More overhead than `unique_ptr`

🏢 Real-world analogy: **Shared office printer**
- Many employees use the same printer
- Printer stays as long as **someone still uses it**
- When the last user leaves → printer is removed


3️⃣ `std::weak_ptr` → **Observe without owning**

Meaning (simple)
**Can see the object, but does not keep it alive.**
- Does NOT increase reference count
- Must be converted (`lock()`) before use
- Prevents cycles

![[Pasted image 20260107184839.png]]
![[Pasted image 20260107184950.png]]

**Q. When should you not use shared_ptr?**
Do not use shared_ptr unless multiple entities genuinely own the resource.

**Q. What problem does weak_ptr solve?**
weak_ptr breaks shared_ptr ownership cycles and allows safe observation without ownership.

**Q. Can references be reseated?**
No, references cannot be reseated in C++. 
Reseating a reference would mean:
* Making a reference that was bound to one object later refer to a different object.

#### Inheritance

Inheritance is about substitutability, not code reuse.

This means:
If a `Derived` object **cannot be used wherever a `Base` is expected**, then inheritance is **wrong**, even if code reuse looks convenient.

This idea comes from the **Liskov Substitution Principle (LSP)**.

1️.  Core Concept: Use inheritance only for **“is-a”**

Correct Example
```cpp
class Animal {
public:
    virtual void speak() = 0;
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void speak() override {
        std::cout << "Bark\n";
    }
};
```

✔ Dog **is an** Animal  
✔ Can substitute `Animal*` with `Dog*`


Wrong Example
```cpp
class Rectangle {
public:
    virtual void setWidth(int w) {}
    virtual void setHeight(int h) {}
};

class Square : public Rectangle {
public:
    void setWidth(int w) override {
        // breaks height invariant
    }
};
```

❌ Square **is NOT** a Rectangle (behavior differs)  
❌ Violates substitutability

📌 **Even though math says square is a rectangle, code behavior says NO.**

2️. Avoid Data Inheritance. 
Don’t let derived classes directly access or modify base class data.
Expose behavior, not raw data.

❌ Data inheritance (dangerous)
```cpp
class Account {
protected:
    int balance;   // ❌ exposed to derived classes
};

class SavingsAccount : public Account {
public:
    void withdraw(int amount) {
        balance -= amount;   // can break invariants
    }
};
```
**What’s wrong here**
- `SavingsAccount` can:
    - Make balance negative
    - Bypass validation
- Base class **loses control**
- Invariants can be violated silently

📌 This creates **fragile designs**.

✅ Behavior-based design (correct)

```cpp
class Account {
private:
    int balance;

public:
    virtual void withdraw(int amount) = 0;

protected:
    void debit(int amount) {
        if (amount > balance)
            throw std::runtime_error("Insufficient funds");
        balance -= amount;
    }
};
```

```cpp
class SavingsAccount : public Account {
public:
    void withdraw(int amount) override {
        debit(amount);   // safe, controlled
    }
};
```
**Why this is correct**
- Base class controls invariants
- Derived class extends **behavior**, not data
- Safer and easier to maintain

3️. Prefer Composition over Inheritance

Don’t inherit from a class just to reuse its code.
Instead, put that class inside your class and use it.

Let’s compare with a very concrete example
Scenario: You want logging

❌ Inheritance (WRONG reason)
```cpp
class Logger {
public:
    void log(const std::string& msg) {
        std::cout << msg << std::endl;
    }
};

class OrderService : public Logger {  // ❌
};
```

Ask yourself honestly:
	Is `OrderService` a `Logger` ?
❌ No  
It **uses** a logger.  
It is **not** one.

This breaks the meaning of inheritance.

✅ Composition (CORRECT)

```cpp
class OrderService {
    Logger logger;   // has-a Logger
};
```

![[Pasted image 20260107212802.png]]

**Real-World Use**
- Payment methods
- Notification channels
- Rendering engines

**Q. Why prefer composition?**
Composition is preferred because it models has-a relationships correctly, avoids incorrect is-a relationships, reduces coupling, and makes systems more flexible and easier to change. Inheritance should not just be for code reuse.

**Q. What is object slicing?**
When a derived object is copied by value into a base object, the derived part is sliced off.

**Q. When is multiple inheritance acceptable?**
Multiple inheritance is acceptable when inheriting from multiple pure abstract classes to model multiple “is-a” relationships without shared state.

Simple Example (Correct Usage)
Interfaces (no data, only behavior)

```cpp
#include <iostream>
using namespace std;

class Printable {
public:
    virtual void print() = 0;
    virtual ~Printable() = default;
};

class Savable {
public:
    virtual void save() = 0;
    virtual ~Savable() = default;
};
```

Class inheriting from multiple interfaces
```cpp
class Document : public Printable, public Savable {
public:
    void print() override {
        cout << "Printing document\n";
    }

    void save() override {
        cout << "Saving document\n";
    }
};
```

Usage
```cpp
int main() {
    Document doc;
    doc.print();
    doc.save();
}
```

**Why this is acceptable**
- `Document` **is-a Printable**
- `Document` **is-a Savable**
- No shared data
- No ambiguity
- Clear responsibilities

#### Polymorphism & Virtual Dispatch

**Polymorphism**
Polymorphism allows different objects to be treated uniformly while invoking behavior specific to their actual type.

That means:
- The caller talks to a **base type**
- The **actual object decides** what code runs
- You can add new behavior **without modifying existing code**

This is the core idea behind **open–closed principle**.

```cpp
#include <iostream>
#include <vector>

using namespace std;

// Base class
class Shape {
public:
    virtual double area() = 0;      // polymorphic behavior
    virtual ~Shape() = default;     // required for safe deletion
};

// Derived class: Circle
class Circle : public Shape {
    double radius;

public:
    Circle(double r) : radius(r) {}

    double area() override {
        return 3.14 * radius * radius;
    }
};

// Derived class: Rectangle
class Rectangle : public Shape {
    double length;
    double width;

public:
    Rectangle(double l, double w) : length(l), width(w) {}

    double area() override {
        return length * width;
    }
};

// Derived class: Triangle
class Triangle : public Shape {
    double base;
    double height;

public:
    Triangle(double b, double h) : base(b), height(h) {}

    double area() override {
        return 0.5 * base * height;
    }
};

int main() {
    vector<Shape*> shapes;

    shapes.push_back(new Circle(10));
    shapes.push_back(new Rectangle(4, 5));
    shapes.push_back(new Triangle(6, 8));

    for (Shape* s : shapes) {
        cout << "Area: " << s->area() << endl;  // virtual dispatch
    }

    for (Shape* s : shapes) {
        delete s;   // safe because destructor is virtual
    }

    return 0;
}
```

```cpp
Area: 314
Area: 20
Area: 24


=== Code Execution Successful ===
```

What you’ll observe when you run it
- Same call: `s->area()`
- Different behavior for each shape
- Correct cleanup via virtual destructor

What problem does `virtual` solve?
* Without `virtual`, C++ decides **at compile time** which function to call.
* With `virtual`, C++ decides **at runtime** based on the **actual object type**.

**How it works (mental model, no jargon overload)**
- Each class with virtual functions has a **vtable**
- Each object has a hidden pointer (**vptr**) to its vtable
- When you call a virtual function:    
    - C++ looks up the function in the vtable
    - Calls the correct override

**Visual (conceptual)**

```
Base* ptr ───▶ Derived object
                  │
                  ▼
               vtable
                  │
                  ▼
           Derived::area()
```

A `vtable` (virtual table) is a lookup table that maps virtual functions to the correct implementation at runtime.

`Virtual dispatch` ensures function calls are resolved at runtime based on the object’s dynamic type.

The `override` keyword ensures the function correctly overrides a virtual base function and prevents silent bugs.



**Q. Can constructors be virtual?**
No, constructors cannot be virtual because virtual dispatch requires a fully constructed object, and during construction the compiler must decide which constructor to call at compile time.

**What is `final`?**
`final` is used to prevent further inheritance or overriding.**

Why mark methods final?
Methods are marked final to prevent derived classes from overriding critical behavior, protect class invariants, and avoid accidental or unsafe overrides.

```cpp
class Account {
protected:
    int balance = 100;

public:
    virtual void withdraw(int amount) final {
        if (amount <= 0 || amount > balance) {
            cout << "Invalid withdrawal\n";
            return;
        }
        balance -= amount;
    }
};
```

```cpp
class HackerAccount : public Account {
    // ❌ compile-time error
    // void withdraw(int amount) override {}
};
```

Result
- Invariant is protected    
- Incorrect overrides are blocked at compile time
- Behavior is guaranteed

**Another simple example: Prevent accidental overrides**
```cpp
class Base {
public:
    virtual void process() final {
        cout << "Base processing\n";
    }
};

class Derived : public Base {
    // void process() override {}  // ❌ not allowed
};
```

Prevents:
- Bugs
- Unintended behavior changes

#### Const Correctness

Const is a design guarantee, not syntax sugar.

**Rules**
- Methods not changing logical state → `const`
- `const` propagates through APIs
- Use `mutable` **only for caches**

```cpp
#include <iostream>
#include <string>
#include <stdexcept>

using namespace std;

class Book {
    string title;
    int pages;

    // cache (performance-only, not logical state)
    mutable bool cacheValid = false;
    mutable int cachedWords = 0;

public:
    Book(string t, int p) : title(std::move(t)), pages(p) {}

    // 1) Doesn't change logical state -> const
    const string& getTitle() const { return title; }
    int getPages() const { return pages; }

    // Mutating method -> NOT const
    void setPages(int p) {
        pages = p;
        cacheValid = false; // invalidate cache because logical state changed
    }

    // 3) mutable only for caches (logical constness)
    int estimatedWords() const {
        if (!cacheValid) {
            cout << "[compute] estimating words...\n";
            cachedWords = pages * 300;  // pretend expensive calculation
            cacheValid = true;
        }
        return cachedWords;
    }
};

class Library {
    Book book;

public:
    Library(Book b) : book(std::move(b)) {}

    // 2) const propagates: const Library can only return const Book&
    const Book& getBook() const { return book; }

    // Non-const overload: non-const Library can return Book&
    Book& getBook() { return book; }
};

void printReport(const Library& lib) { // const propagates into here
    const Book& b = lib.getBook();     // gets const Book&
    cout << b.getTitle() << " pages=" << b.getPages()
         << " words=" << b.estimatedWords() << "\n";

    // b.setPages(999); // ❌ won't compile: b is const
}

int main() {
    Library lib(Book("C++ LLD Notes", 100));

    // const propagation through APIs
    printReport(lib);

    // cache demo: compute happens only once
    cout << "Again (cache): " << lib.getBook().estimatedWords() << "\n";

    // mutation invalidates cache (needs non-const access)
    lib.getBook().setPages(120);
    cout << "After change: " << lib.getBook().estimatedWords() << "\n";
}
```

Output
```cpp
C++ LLD Notes pages=100 words=[compute] estimating words...
30000
Again (cache): 30000
After change: [compute] estimating words...
36000


=== Code Execution Successful ===
```

#### Operator Overloading (Minimal & Safe)

Overload operators only when the meaning is obvious and consistent with built-in types, ensure every overload preserves invariants, and avoid clever side effects so the code stays readable and predictable.

```cpp
#include <stdexcept>
#include <iostream>

class Money {
    long long paise;
    std::string currency; // invariant: currency must match when adding

public:
    Money(long long p, std::string c) : paise(p), currency(std::move(c)) {}

    const std::string& curr() const { return currency; }
    long long value() const { return paise; }

    // natural: money + money
    Money operator+(const Money& other) const {
        if (currency != other.currency)
            throw std::invalid_argument("Currency mismatch");
        return Money(paise + other.paise, currency);
    }
};
```

```cpp
bool operator==(const User& a, const User& b) {
    return a.id == b.id;
}
```

**Rules**
- Overload only when natural
- Preserve invariants
- Avoid cleverness

#### Exceptions & Safety Guarantees

**RAII**(Resource Acquisition Is Initialization) is the C++ idiom where resources are acquired in constructors and released in destructors, ensuring automatic cleanup and exception safety.

```cpp
#include <iostream>

class FileHandle {
    FILE* f;

public:
    FileHandle(const char* path) : f(std::fopen(path, "r")) {
        if (!f) throw std::runtime_error("Failed to open file");
    }

    ~FileHandle() {
        if (f) std::fclose(f); // cleanup guaranteed
    }
};

int main() {
    FileHandle fh("data.txt");  // acquire in constructor
    // use file...
} // fh destroyed here -> file closed automatically
```

RAII prevents:
- memory leaks
- file handle leaks
- socket leaks
- lock not released
- exception-safety bugs

##### The 3 Guarantees

###### Basic Guarantee(minimum)
 
If an exception occurs, the program remains in a valid state and there are no resource leaks.

- Object invariants still hold
- Some state may have changed (partial progress allowed)
- But object is still usable / destructible safely

✅ Most code should at least give this.

**Example idea**
“Push to vector”: if it fails, vector is still valid, but element may or may not be inserted.

###### Strong Guarantee (transaction/rollback)
**If an exception occurs, the operation has no effect (state rolls back).**
- “All or nothing"    
- Either the operation succeeds completely, or the object stays exactly as before.

✅ Ideal for APIs where partial changes would be confusing or dangerous.

Common technique: **copy-and-swap**
Do work on a temporary; only commit at the end.

###### No-throw Guarantee (guaranteed success)

 **The operation will never throw.**
- Marked `noexcept`
- Used heavily by STL for moves, destructors, swaps

✅ Needed for building blocks, cleanup, and performance-critical operations.

##### One simple class showing Basic vs Strong

Example: `BankAccount` invariant: balance ≥ 0
Basic guarantee version (valid, but might change)

```cpp
class BankAccount {
    int balance; // invariant: balance >= 0
public:
    void deposit(int amt) {
        if (amt < 0) throw std::invalid_argument("neg");
        balance += amt;           // after this point, if something throws later, balance changed
        // ... imagine logging could throw here
    }
};
```
If logging throws after `balance += amt`, balance changed → still valid (basic), not strong.

**Strong guarantee version (commit at end)**

```cpp
class BankAccount {
    int balance; // invariant: balance >= 0
public:
    void deposit(int amt) {
        if (amt < 0) throw std::invalid_argument("neg");

        int newBalance = balance + amt;  // compute first
        // ... do risky work first (may throw), like logging preparation
        balance = newBalance;            // commit only at end
    }
};

```
If anything throws before `balance = newBalance`, object unchanged → strong guarantee.

Where these guarantees matter most (real-world)
- **Money transfer**: strong guarantee (no partial debit/credit)
- **In-memory cache update**: basic might be fine
- **Destructor / cleanup / unlock**: no-throw guarantee
- **Move constructor** (for performance in containers): ideally `noexcept`

Interview-ready summary (memorize)
**Basic guarantee**: no leaks, object remains valid.  
**Strong guarantee**: operation is atomic—rollback on failure.  
**No-throw guarantee**: operation never throws (often `noexcept`).  
Goal: exceptions should not break invariants or corrupt program state.

**Q. Why must destructors not throw?**
Destructors must not throw because if they throw during stack unwinding (while another exception is already active), the runtime calls `std::terminate`, causing immediate program termination and potentially skipping further cleanup.

**Q. Why RAII enables strong exception safety?**
RAII enables strong exception safety because you can perform work on temporary objects that own all resources. If an exception occurs, destructors of those temporaries automatically release resources and discard partial work, so nothing is committed. Only after everything succeeds do you “commit” (e.g., swap), achieving all-or-nothing behavior.


#### Modern C++ Rules for LLD

- **Prefer value semantics:** Design types that behave like `int`/`std::string`—copying creates an independent value and ownership is clear.
- **Prefer RAII:** Acquire resources in constructors and release them in destructors so cleanup is automatic, even on exceptions.
- **Prefer composition:** Build classes by _having_ other objects (has-a) instead of inheriting just for reuse.
- **Avoid raw `new/delete`:** Don’t manually manage heap memory—use vectors,  `std::unique_ptr`, `std::shared_ptr`, or containers.
- **Minimize inheritance depth:** Keep inheritance hierarchies shallow to reduce coupling, complexity, and fragile base-class problems.