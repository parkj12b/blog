---
title: "C++ Standard (3) (C++11)"
date: 2026-05-10T14:49:00+09:00
lastmod: 2026-05-12T00:00:00+09:00
draft: false
categories: ["Programming"]
tags: ["C++", "C++11", "Programming"]
featureimage: "cpp-logo.png"
---

[C++ Standard (2) (C++11) ]({{< ref "cpp-standard-2-cpp11" >}})

## type alias

Type aliasing is a feature I've often used when solving coding test problems. It allows you to set an alias for an existing type rather than creating a new one.

```cpp
using NewName = ExistingType;
```

```cpp
using tdii = tuple<double, int, int>;

// Extremely long...
priority_queue<tuple<double, int, int>, vector<tuple<double, int, int>>, greater<tuple<double, int, int>>> pq1;

// Much more concise :)
priority_queue<tdii, vector<tdii>, greater<tdii>> pq2;

// C style
typedef void (*Callback)(int, const std::string&);

// with type alias
using Callback = void (*)(int, const std::string&);

```

### Alias templates

```cpp
#include <map>
#include <string>

// We want to create a dictionary type where the key is always a string,
// but the value can be any type 'T'.

// C++11 Alias Template:
template <typename T>
using StringMap = std::map<std::string, T>;

int main() {
    // StringMap<int> is identical to std::map<std::string, int>
    StringMap<int> ages;
    ages["Alice"] = 30;
    
    // StringMap<double> is identical to std::map<std::string, double>
    StringMap<double> bankBalances;
    bankBalances["Bob"] = 1500.50;
    
    return 0;
}
```

- Alias templates are also possible.

## Variadic templates (parameter pack)

Before C++11, templates could only accept a fixed number of arguments. With the introduction of the parameter pack in C++11, variadic templates became possible.

```cpp
#include <iostream>

template <typename T>
void print(T arg) {
  std::cout << arg << std::endl;
}

template <typename T, typename... Types>
void print(T arg, Types... args) {
  std::cout << arg << ", ";
  print(args...);
}

int main() {
  print(1, 3.1, "abc");
  print(1, 2, 3, 4, 5, 6, 7);
}
```

```cpp
template <typename T, typename... Types>
```

- The part with `typename...` is called a parameter pack.
    - It can accept zero or more arguments.
- At compile time, the compiler checks the types used to call this function and creates the corresponding functions.
- If there is a function that can overlap with the function containing the parameter pack, the function without the parameter pack takes priority when called.

```cpp
#include <iostream>

template <typename T, typename... Types>
void print(T arg, Types... args) {
  std::cout << arg << ", ";
  print(args...);
}

// ERROR!!! 
template <typename T>
void print(T arg) {
  std::cout << arg << std::endl;
}
```

- When declaring templates whose function signatures might overlap as shown above, the parameter pack must be placed later. Otherwise, a compiler error will occur.

## Generalized Union

Unions help use data space efficiently by allowing different types to occupy the same memory space.

Before C++11, unions could only contain Plain Old Data (POD) types (int, float, raw pointers, etc.). With the addition of Generalized Unions, complex types like `string` can also be placed in unions.

```cpp
#include <iostream>
#include <string>

// A generalized union containing a non-POD type (std::string)
union DataUnion {
    int intValue;
    std::string stringValue;

    // 1. We MUST define a constructor. 
    // Let's make 'intValue' the default active member.
    DataUnion() : intValue(0) {}

    // 2. We MUST define a destructor.
    // In a real scenario, you'd need a separate tag (enum) to track which 
    // member is currently active so you know whether to call the string's destructor.
    // For this simple example, we leave it empty and handle it manually in main().
    ~DataUnion() {} 
};

int main() {
    DataUnion data;

    // Using the trivial type is easy
    data.intValue = 42;
    std::cout << "Integer: " << data.intValue << std::endl;

    // --- SWITCHING TO THE STRING ---
    
    // ERROR: You cannot just assign to it! The string object doesn't exist yet.
    // data.stringValue = "Hello"; // Undefined behavior/Crash!

    // CORRECT: Use placement new to construct the string in place.
    new (&data.stringValue) std::string("Hello C++11 Unions!");
    std::cout << "String: " << data.stringValue << std::endl;

    // --- CLEANING UP ---
    
    // CORRECT: We must explicitly call the destructor for the string 
    // before the union goes out of scope, or before switching back to the int.
    data.stringValue.~basic_string();

    return 0;
}
```

Two things are required here to use complex types:

1. Use of placement `new`
2. Explicit destructor call

### placement `new`

While standard `new` allocates memory on the `heap` and then creates an object, placement `new` skips memory allocation by passing a memory address and creates the object there.

```cpp
#include <new>

// new (memory_address) Type(arguments...);
```

To use placement `new`, the `<new>` header must be included.

```cpp
#include <iostream>
#include <new>    // Required for placement new
#include <string>

class Player {
public:
    std::string name;
    int health;

    Player(std::string n, int h) : name(n), health(h) {
        std::cout << "Player " << name << " constructed.\n";
    }
    
    ~Player() {
        std::cout << "Player " << name << " destroyed.\n";
    }
};

int main() {
    // STEP 1: Pre-allocate raw memory. 
    // We create a buffer of raw bytes just large enough to hold a Player.
    // This memory is on the stack, but it could be anywhere (heap, global, etc.)
    alignas(Player) char memoryBuffer[sizeof(Player)];

    // STEP 2: Use placement new to construct the object IN our buffer.
    // We pass the address of our buffer to the new operator.
    Player* p1 = new (memoryBuffer) Player("Hero", 100);

    // STEP 3: Use the object normally
    std::cout << "Health: " << p1->health << std::endl;

    // STEP 4: The Golden Rule - Explicit Destructor Call
    // You MUST manually call the destructor when you are done.
    p1->~Player();

    // DO NOT CALL delete p1! 
    // We didn't use standard new, so standard delete will try to free 
    // stack memory, which will crash your program.

    return 0;
}
```

## Generalized PODs

Plain Old Data (POD) is a C-style struct as mentioned above. The C++11 committee felt that the conditions for POD were too restrictive and were forcing two different concepts together:

1. Whether this object can be safely copied with `memcpy()`
2. Whether it is laid out in memory exactly the same as a C struct

To generalize POD, these concepts were split into two:

### Trivial Types

Trivial types are types that can be safely copied with `memcpy()`.

For a class or struct to be trivial, the following are prohibited:

- User-defined or deleted:
    - Constructor
    - Copy/move constructor
    - Destructor
- Virtual functions or virtual base classes

### Standard Layout Types

If the size of the memory space occupied by a type can be predicted, it is classified as a Standard Layout Type.

Standard Layout Types can have the following:

- User-defined constructor/destructor
- Member functions
- `private` / `protected` members (all non-static members must have the same access type; you cannot mix private and public)
- Base classes (all data members must be declared in a single class: [derived data, base empty] or [derived empty, base data])

```cpp
#include <iostream>
#include <type_traits> // Required for checking type properties

// In C++98, this would NOT be a POD because of private members and methods.
// In C++11, it is a Standard-Layout type!
class Point3D {
private:
    // All data members have the SAME access level (private). 
    // This guarantees a predictable memory layout.
    float x;
    float y;
    float z;

public:
    // It has user-defined constructors (so it is NOT Trivial)
    Point3D(float x, float y, float z) : x(x), y(y), z(z) {}

    // It has member functions
    float getX() const { return x; }
};

int main() {
    // Check if it is Trivial (Result: False, because of the constructor)
    std::cout << "Is Trivial? " 
              << std::is_trivial<Point3D>::value << std::endl;

    // Check if it has Standard Layout (Result: True!)
    std::cout << "Is Standard Layout? " 
              << std::is_standard_layout<Point3D>::value << std::endl;

    // Check if it is a full POD (Result: False, because it's not Trivial)
    std::cout << "Is POD? " 
              << std::is_pod<Point3D>::value << std::endl;

    return 0;
}
```

### Significance of separation

By separating Trivial Type and Standard Layout Type, it's possible to know which type is needed for each situation:

- Communication with C APIs or OS structs - `std::is_standard_layout`
- When you want to move large data quickly with `memcpy()` for network communication, etc. - `std::is_trivial`

## User-defined literals

Usually, when we think of literals, we think of values like `100`, `100.1`, `100.1f`, `100UL`.

```cpp
ReturnType operator"" _suffix(Parameters);
```

- User-defined literals must start with an underscore `_`.
- The possible parameter types are as follows:
    - `unsigned long long`
    - `long double`
    - `(const char* text, std::size_t length)`
    - `char`, `wchar_t`, `char16_t`, or `char32_t`

```cpp
#include <iostream>

// Let's say our internal representation is always Long Double Meters
using Distance = long double;

// 1. Literal for Meters (Base unit, no conversion)
Distance operator"" _m(long double val) {
    return val;
}

// 2. Literal for Kilometers (Convert to meters)
Distance operator"" _km(long double val) {
    return val * 1000.0;
}

// 3. Literal for Centimeters (Convert to meters)
Distance operator"" _cm(long double val) {
    return val / 100.0;
}

int main() {
    // Look at how readable this is!
    Distance tripToStore = 2.5_km;
    Distance tableLength = 150.0_cm;
    Distance step = 1.0_m;

    Distance total = tripToStore + tableLength + step;

    std::cout << "Total distance in meters: " << total << "m\n";
    // Output: Total distance in meters: 2502.5m

    return 0;
}
```

```cpp
#include <iostream>
#include <string>
#include <algorithm>

// A custom literal that automatically converts a string to uppercase
std::string operator"" _upper(const char* str, std::size_t length) {
    std::string result(str, length);
    std::transform(result.begin(), result.end(), result.begin(), ::toupper);
    return result;
}

int main() {
    std::string loudGreeting = "hello world"_upper;
    
    std::cout << loudGreeting << std::endl; 
    // Output: HELLO WORLD

    return 0;
}
```

## Attributes

Attributes are used to pass additional information to the compiler. Originally they were compiler extensions: `__attributes__((...))` in GCC/Clang and `__delspec(...)` in MSVC.

```cpp
[[attribute_name]] void myFunction();
```

```cpp
// A compiler-specific attribute using the C++11 standard syntax
[[gnu::always_inline]] void fastFunction();
```

Compilers can have their own namespace scopes.

Commonly used attributes include:

- `[[noreturn]]`
- `[[deprecated(”message”)]]` (C++14)
- `[[fallthrough]]` , `[[nodiscard]]` ,`[[maybe_unused]]` (C++17)
- `[[likely]]`, `[[unlikely]]` (C++20)

## Lambda expressions

These are similar to lambda expressions or anonymous functions in many other languages.

```cpp
[captures] (parameters) -> return_type { 
    // function body 
}
```

- `[captures]`: Determines which surrounding variables become available within the lambda expression.
- `(parameters)`: Parameters similar to regular functions.
- `return_type`: trailing return type. Usually, the compiler can deduce the return type, so it can be omitted.

### Basic lambda expression

```cpp
#include <iostream>

int main() {
    // A lambda that takes two ints and returns their sum.
    // Notice we skipped the "-> int" because the compiler deduces it.
    auto add = [](int a, int b) { 
        return a + b; 
    };

    std::cout << "3 + 4 = " << add(3, 4) << std::endl;

    return 0;
}
```

### sort()

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cmath>

int main() {
    std::vector<int> numbers = {5, -2, 8, -1, 3};

    // Sort by absolute value using an inline lambda
    std::sort(numbers.begin(), numbers.end(), [](int a, int b) {
        return std::abs(a) < std::abs(b);
    });

    for (int n : numbers) {
        std::cout << n << " "; // Output: -1 -2 3 5 8
    }

    return 0;
}
```

### captures example

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    int multiplier = 10;
    int sum_of_results = 0;

    // We capture 'multiplier' by value (read-only copy)
    // We capture 'sum_of_results' by reference (we want to modify it)
    std::for_each(numbers.begin(), numbers.end(), [multiplier, &sum_of_results](int n) {
        int result = n * multiplier;
        sum_of_results += result; // Modifying the outside variable
    });

    std::cout << "Total Sum: " << sum_of_results << std::endl; 
    // Output: Total Sum: 150

    return 0;
}
```

wildcard capture: `[=]`

reference capture: `[&]`

## noexcept

In C++11, `noexcept` can be used as a specifier and an operator.

### noexcept specifier

```cpp
void doSomethingFast() noexcept; // Promises not to throw
void doSomethingElse() noexcept(true); // Same as above
void riskyFunction() noexcept(false); // Can throw (same as omitting noexcept)
```

If a function specified with `noexcept` throws an exception, the program calls the `terminate()` function and terminates immediately.

### noexcept operator

```cpp
constexpr bool is_safe = noexcept(doSomethingFast()); // is_safe == true
```

The usage is different, but it's not different from the `noexcept` specifier.

Where to use `noexcept`:

1. Move constructors, move assignment operators
2. Destructors (implicitly `noexcept` by default)
3. Swap functions
4. Functions that do not throw exceptions

Where not to use `noexcept`:

1. Dynamic memory allocation
2. Legacy C++ / external library code
3. Functions that throw exceptions

## alignas, alignof

These are features also covered in C11. They were added to C++11 as memory alignment compiler attributes. I won't cover them again here, but I'll record additional content.

As you know, `alignas` and `alignof` are often used in low-level, data-oriented design, and performance-critical applications.

- Cache Optimization
- SIMD instructions
- DMA and Embedded Hardware

In C++11, when allocating with `alignas` using `new`, the default memory allocator may be ignored. In that case, functions like `posix_memalign()` or `_aligned_malloc()` should be used instead.

<aside>
💡

This was corrected in C++17 with the introduction of the alignment-aware allocator for the `new` operator.

</aside>

### Differences from C11

In C++, `alignas` and `alignof` are keywords, but in C11, they are written as `_Alignas` and `_Alignof` with an underscore prefix.

To use C++ style in C11, you can include `<stdalign.h>`.

## Multi-threaded memory model

Before C++11, all applications were assumed to run in a single thread. C++11 introduced `std::atomic` and a memory model.

[](https://en.cppreference.com/cpp/atomic/memory_order)

Please check the link above for details. GCC compiler extensions also have `__sync` and the modern version `__atomic`. `__atomic` is designed to correspond 1:1 with the C++11 memory model. It's recommended to use C++ features rather than compiler extensions.

## Thread-local storage

Thread Local Storage (TLS) allows each thread to have its own instance of a variable by declaring it with the `thread_local` keyword.

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex cout_mutex;

// A standard global variable (Shared among all threads)
int shared_counter = 0;

// A thread-local variable (Each thread gets its own instance starting at 0)
thread_local int local_counter = 0;

void do_work(int thread_id) {
    shared_counter++;
    local_counter++; // No atomics or mutexes needed!

    std::lock_guard<std::mutex> lock(cout_mutex);
    std::cout << "Thread " << thread_id 
              << ": shared=" << shared_counter 
              << ", local=" << local_counter << "\n";
}

int main() {
    std::thread t1(do_work, 1);
    std::thread t2(do_work, 2);
    t1.join();
    t2.join();
    return 0;
}
```

Using local variables is much faster than using mutexes or cache-line bouncing via atomics, but it's not free. It's slower than referencing standard local and global variables.

### x86-64 implementation

The implementation is as follows:

1. The compiler declares `thread_local` variables in the `.tdata` and `.tbss` sections of the program.
2. The operating system makes a specific CPU register point to the TLS block, such as `FS` or `GS`.
3. When accessing a variable, the compiler accesses it via an offset from the `FS` register.

The `.tdata` and `.tbss` sections are read-only templates. When a new thread is spawned, a new block is allocated.

## range-for

```cpp
#include <vector>
#include <iostream>

struct LargePayload { uint64_t data[128]; };
std::vector<LargePayload> buffer(1000);

// 1. The "Read-Only" (Best Practice Default)
// Uses const reference. No memory is copied. Fast and safe.
for (const auto& payload : buffer) {
    // Read payload...
}

// 2. The "Mutator" 
// Uses standard reference. Modifies the actual elements in the container.
for (auto& payload : buffer) {
    payload.data[0] = 0xFF; // Directly writes to the vector's memory
}

// 3. The "Accidental Copy" (Avoid for large types)
// Deduced by value. Copies all 1024 bytes for EVERY element. 
// Destroys performance. Only use this for primitive types (int, float).
for (auto payload : buffer) {
    // ...
}

// 4. The "Forwarder" (Advanced)
// Uses a universal/forwarding reference. Rarely used in simple loops, 
// but useful in generic template programming where you don't know 
// if the container yields values, references, or proxy objects (like std::vector<bool>).
for (auto&& payload : buffer) {
    // ...
}
```

This is similar to `range in` or `for in` in other languages.

A point of caution is that, just like when using iterators, you should not modify the structure of the container.

## static_assert

This is an assert that can be performed at compile time.

```cpp
static_assert(sizeof(int) >= 4, "Integers must be at least 32 bits on this platform.");
```

### Example

```cpp
#include <cstdint>
#include <cstddef>

// A struct meant to be written exactly to an SPI or DMA buffer
struct HardwareHeader {
    uint16_t transaction_id;
    uint8_t  command_code;
    uint8_t  status_flags;
    uint32_t payload_address;
};

// Guarantee the total size is exactly 8 bytes (no hidden padding)
static_assert(sizeof(HardwareHeader) == 8, 
              "HardwareHeader size mismatch! Compiler padding detected.");

// Guarantee specific members sit at exact byte offsets
static_assert(offsetof(HardwareHeader, payload_address) == 4, 
              "payload_address must be at offset 4 for the hardware to read it.");
```

```cpp
// Ensure we are compiling for a 32-bit embedded target
static_assert(sizeof(void*) == 4, 
              "This firmware is optimized exclusively for 32-bit architectures.");
```

```cpp
#include <type_traits>

template <typename T>
class FastMemoryPool {
    // If someone tries to create a FastMemoryPool<std::string>, 
    // the compiler will immediately stop and print this error.
    static_assert(std::is_trivially_copyable<T>::value, 
                  "FastMemoryPool only supports trivially copyable types "
                  "to safely utilize memmove optimizations.");

    T* pool_data;
public:
    // ... pool logic ...
};
```
