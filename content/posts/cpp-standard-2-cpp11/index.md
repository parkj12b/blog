---
title: "C++ Standard (2) (C++11)"
date: 2026-05-08T12:45:00+09:00
lastmod: 2026-05-12T00:00:00+09:00
draft: false
categories: ["Programming"]
tags: ["C++", "C++11", "Programming"]
featureimage: "cpp-logo.png"
---

[C++ Standard (1) (C++11) ]({{< ref "cpp-standard-1-cpp11" >}})

## move constructor and move assignment operator

The move constructor and move assignment operator are related to `&&`, rvalue reference, which was covered in the previous post.

### move constructor

```cpp
ClassName(ClassName&& other) noexcept;
```

- `noexcept` is a promise that the function does not throw an exception.

**Importance of `noexcept` in move constructor**

Since the move constructor steals resources without copying, an exception occurring in the middle is an unrecoverable error. If a move constructor is not declared as `noexcept`, data structures like `vector` fallback to the copy constructor.

### move assignment operator

```cpp
ClassName& operator=(ClassName&& other) noexcept;
```

- It is similar to the move constructor, but since the object being assigned may already hold resources, it cleans up its own resources first before assigning.

### Example

```cpp
#include <iostream>
#include <utility> // For std::move

class IntArray {
private:
    int* data;
    size_t size;

public:
    // 1. Regular Constructor
    IntArray(size_t s) : size(s), data(new int[s]) {
        std::cout << "Constructor called. Allocated " << size << " ints.\n";
    }

    // 2. Destructor
    ~IntArray() {
        std::cout << "Destructor called. Cleaning up.\n";
        delete[] data;
    }

    // 3. Copy Constructor (Deep Copy)
    IntArray(const IntArray& other) : size(other.size), data(new int[other.size]) {
        std::cout << "Copy Constructor called. Deep copying.\n";
        for (size_t i = 0; i < size; ++i) {
            data[i] = other.data[i];
        }
    }

    // ==========================================
    // 4. MOVE CONSTRUCTOR
    // ==========================================
    IntArray(IntArray&& other) noexcept 
        : size(other.size), data(other.data) { // Steal the data!
        
        std::cout << "Move Constructor called. Stealing resources.\n";
        
        // Leave the 'other' object in a valid, empty state
        // so its destructor doesn't crash or delete our stolen data.
        other.size = 0;
        other.data = nullptr; 
    }

    // ==========================================
    // 5. MOVE ASSIGNMENT OPERATOR
    // ==========================================
    IntArray& operator=(IntArray&& other) noexcept {
        std::cout << "Move Assignment Operator called.\n";
        
        // Protect against self-assignment (e.g., arr = std::move(arr))
        if (this != &other) {
            // Free the existing resource we currently own
            delete[] data;

            // Steal the new resources
            size = other.size;
            data = other.data;

            // Reset the 'other' object
            other.size = 0;
            other.data = nullptr;
        }
        return *this;
    }
};

// A simple factory function returning an object by value (creates a temporary)
IntArray createArray(size_t size) {
    return IntArray(size);
}

int main() {
    std::cout << "--- 1. Testing Move Constructor ---\n";
    // createArray(100) returns a temporary rvalue. 
    // The compiler will use the Move Constructor to initialize arr1.
    IntArray arr1 = createArray(100); 

    std::cout << "\n--- 2. Testing Move Assignment ---\n";
    IntArray arr2(50);
    // arr2 already exists. We assign it a temporary rvalue.
    // The compiler will use the Move Assignment Operator.
    arr2 = createArray(200); 

    std::cout << "\n--- 3. Using std::move ---\n";
    // We can explicitly cast an lvalue (named variable) to an rvalue 
    // using std::move if we know we are done with it.
    IntArray arr3 = std::move(arr1); // Calls Move Constructor

    std::cout << "\n--- End of Scope ---\n";
    return 0;
}
```

- `std::move` shown above does not actually perform a move but is a function that casts an lvalue to an rvalue reference.

### Key Summary

- `noexcept` is important when declaring move constructors and assignment operators.
- Rule of Five: In modern C++, if a class directly manages resources, it must implement all five (destructor, copy constructor, copy assignment operator, move constructor, and move assignment operator). + The default constructor is the Orthodox Canonical Class Form (OCCF) of C++11.
- `std::move` converts an lvalue to an rvalue reference.

## Scoped enums

### enums before C++11

```cpp
enum Color { RED, GREEN, BLUE };
enum Alert { GREEN, YELLOW, RED }; // ERROR
```

- These are C-style enums.
- `RED` and `GREEN` in `Color` conflict with `Alert`, and since both enums share the same scope, an error occurs.

```cpp
enum Fruit { APPLE, BANANA };
enum Car { FORD, TOYOTA };

Fruit mySnack = APPLE;
int number = mySnack; // Implicitly converts to 0. (Wait, a fruit is a number?)

if (APPLE == FORD) {
    // This evaluates to true! Both are technically integer 0.
    // This is a recipe for terrible bugs.
}
```

Another problem can be seen in the code above: even though `APPLE` and `FORD` are different enums, implicit type conversion was possible, and comparison between different enums was allowed.

### C++11 scoped enum

```cpp
enum class Color { Red, Green, Blue };
enum class Alert { Green, Yellow, Red }; // Perfectly fine!

Color myColor = Color::Red;
Alert myAlert = Alert::Red; // No confusion here.
```

- With the introduction of `enum class`, namespaces must be explicitly specified, and scopes are separated.

```cpp
enum class Fruit { Apple, Banana };
enum class Car { Ford, Toyota };

Fruit mySnack = Fruit::Apple;
// int number = mySnack;           // ERROR! Cannot implicitly convert.
// if (Fruit::Apple == Car::Ford)  // ERROR! Cannot compare different enum classes.

// If you REALLY need the integer value, you must be explicit:
int number = static_cast<int>(mySnack);
```

The problems mentioned above, such as implicit type conversion and comparison between different enums, have been resolved.

```cpp
// We know this will only ever need 1 byte of memory.
enum class Status : unsigned char {
    Success = 0,
    Failure = 1,
    Pending = 2
};

// Because the type and size are known, you can forward declare it in a header!
enum class Status : unsigned char;
```

- In the case of C-style enums, forward declaration was difficult because the memory size occupied by the enum was unknown.
- In C++, you can specify the size.

## constexpr and literal types

`constexpr` is a function seen for the first time that did not exist in C. It uses compile-time optimization to pre-calculate values known at compile time and embed them into the code.

```cpp
// This function can run during compilation!
constexpr int square(int x) {
    return x * x;
}

int main() {
    // 1. Evaluated at COMPILE-TIME. 
    // The compiled binary just contains the number 25. The function isn't called at runtime.
    constexpr int compile_time_val = square(5); 

    // 2. Evaluated at RUNTIME.
    // Because 'runtime_input' isn't known until the program runs, 
    // 'square' acts like a normal runtime function here.
    int runtime_input;
    std::cin >> runtime_input;
    int runtime_val = square(runtime_input); 
}
```

- As shown above, even for a `constexpr` function, values that can only be known at runtime are calculated at runtime.

It cannot be used for complex objects or those requiring dynamic memory allocation like `std::string` or `std::vector`.

### Literal types

Literal types are simple types that can be created, manipulated, and destroyed at compile time.

1. **Scalar types:** `int`, `float`, `double`, `char`, `bool`, etc.
2. **References** (`int&`) and **Pointers** (`int*`).
3. **Arrays** of literal types.
4. **Classes / Structs**, but *only* if they meet strict rules.

All of these are literal types.

### Rules for a class to be a literal type

- The destructor must be a default destructor.
- There must be at least one `constexpr` constructor (the compiler must know how to create it at compile time).
- All member variables must also be literal types.
- It cannot include virtual functions or virtual base classes.

```cpp
constexpr int compile_time_val = square(5); 

1184:    c7 45 f0 19 00 00 00    movl   $0x19,-0x10(%rbp)
```

- You can see in the code above that 5 * 5 was replaced with `0x19` (decimal 25).

## list initialization

List initialization is a way to initialize variables or objects using curly braces `{}`.

```cpp
// 1. Primitive types
int a{5};          // Direct list initialization
int b = {10};      // Copy list initialization
int c{};           // Value initialization (c becomes 0)

// 2. Arrays
int arr[3]{1, 2, 3}; 

// 3. Pointers
int* ptr{nullptr};

// 4. Custom Classes / Structs
struct Point { int x; int y; };
Point p1{10, 20};

// 5. STL Containers
#include <vector>
#include <string>

std::vector<int> numbers{1, 2, 3, 4, 5};
std::string text{"Hello, World!"};
```

At first glance, you might wonder why this is good.

### Prevention of narrowing conversions

```cpp
// Truncation
int x = 4.9;
int y(4.9);

// Compile Error!
int z{4.9};
```

- A compile error occurs, preventing narrowing conversions.

### Most Vexing Parse

```cpp
class Timer {};

// You might think this creates a Timer object named 't'...
Timer t(); 

// ...but it actually declares a function named 't' that returns a Timer!
```

- Older compilers could not distinguish whether a statement like `Timer t()` calls a function or creates an object.

### Seamless Container Initialization (`std:initializer_list`)

```cpp
// Pre-C++11
std::vector<int> v;
v.push_back(1);
v.push_back(2);
v.push_back(3);

// C++11 using List Initialization
std::vector<int> v{1, 2, 3}; 

std::map<std::string, int> ages{
    {"Alice", 28},
    {"Bob", 34}
};
```

- You no longer need to use tedious initialization methods like `push_back`.

## delegating constructors and inheriting constructors

### delegating constructors

```cpp
class Player {
    int hp;
    int mp;
    int level;

public:
    // Target constructor (The main constructor that actually handles all initialization)
    Player(int h, int m, int l) : hp{h}, mp{m}, level{l} {
        // Assume complex initialization logic
    }

    // Delegating constructor (Delegates initialization by calling the target constructor)
    Player() : Player(100, 50, 1) {} 
    
    // Another delegating constructor
    Player(int h) : Player(h, 0, 1) {} 
};

// Usage:
Player p1{};      // hp:100, mp:50, level:1
Player p2{200};   // hp:200, mp:0, level:1
```

```cpp
// The above is interpreted as follows:
When calling Player() -> Call Player(100, 50, 1)
```

- If you understand the above, you'll see why it's called a delegating constructor.

### inheriting constructors

```cpp
class Weapon {
public:
    Weapon(int damage) { /* ... */ }
    Weapon(int damage, float speed) { /* ... */ }
    Weapon(std::string name, int damage, float speed) { /* ... */ }
};

class Sword : public Weapon {
public:
    // Before C++11 (Redundant code)
    // Sword(int damage) : Weapon(damage) {}
    // Sword(int damage, float speed) : Weapon(damage, speed) {}
    // Sword(std::string name, int damage, float speed) : Weapon(name, damage, speed) {}

    // C++11 Inheriting constructor (Solved with just one line!)
    using Weapon::Weapon; 
};

// Usage:
Sword basic_sword{10}; 
Sword fast_sword{15, 1.5f};
Sword excalibur{"Excalibur", 100, 2.0f};
```

- Before C++11, to inherit a class, you had to add constructors one by one, but now it can be handled with just a single line: `using Weapon::Weapon`.

## brace-or-equal initialization

```cpp
class Character {
    // Using Brace-or-equal initializers
    int hp = 100;          // Using equal sign (=)
    int mp{50};            // Using curly braces ({}) (Uniform Initialization)
    std::string name = "NPC"; 

public:
    // 1. Default constructor: The default values specified above (100, 50, "NPC") are automatically used.
    Character() {} 

    // 2. Overloaded constructor: Changes only hp and name, uses the default value (50) for mp.
    Character(int h, std::string n) : hp{h}, name{n} {} 
};
```

- Allows initialization directly inside the class.

Before C++11, since initialization had to be done only through constructors, there was the inconvenience of having to use an initializer list for every constructor where default values were used.

```cpp
// Before C++11 (Inconvenient way)
class Box {
    int width, height, color;
public:
    Box() : width(10), height(10), color(0) {} 
    Box(int w, int h) : width(w), height(h), color(0) {} // Redundant color(0)!
};

// After C++11 (Using Brace-or-equal)
class Box {
    int width = 10;
    int height = 10;
    int color = 0;   // Specify once here, and that's it!
public:
    Box() {}
    Box(int w, int h) : width{w}, height{h} {} // color automatically becomes 0
};
```

As initialization is centralized, missing initializations can be prevented, and it's better in terms of readability.

## nullptr

Many people probably know about `nullptr`. Before C++11, `NULL` or `0` was used, which caused the following problems.

```cpp
#include <iostream>

// Function 1: Takes an integer
void doSomething(int x) {
    std::cout << "Called with an integer!" << std::endl;
}

// Function 2: Takes an integer pointer
void doSomething(int* ptr) {
    std::cout << "Called with a pointer!" << std::endl;
}

int main() {
    doSomething(0);    // Calls the integer version (Expected)
    
    // THE BUG:
    // You might think this passes a null pointer and calls the pointer version...
    // But because NULL is just 0, it calls the INTEGER version!
    doSomething(NULL); 
}
```

When making a `doSomething(NULL)` call as shown above, the preprocessor replaces `NULL` with `0`, so to call the `int *ptr` version, there was the disadvantage of having to do an `(int *)` type cast from the outside.

```cpp
int *x = nullptr;
doSomething(nullptr);
```

- `nullptr` can put 0 into a pointer value without casting.
