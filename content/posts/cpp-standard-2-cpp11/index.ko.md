---
title: "C++ 표준 (2) (C++11)"
date: 2026-05-08T12:45:00+09:00
lastmod: 2026-05-12T00:00:00+09:00
draft: false
categories: ["Programming"]
tags: ["C++", "C++11", "Programming"]
featureimage: "cpp-logo.png"
---

[C++ 표준 (1) (C++11) ]({{< ref "cpp-standard-1-cpp11" >}})

## move 생성자와 move 할당자

move 생성자와 move 할당자는 전 포스팅에서 다루었던 `&&` , rvalue reference 와 관련이 있다.

### move 생성자

```cpp
ClassName(ClassName&& other) noexcept;
```

- noexcept 는 함수가 exception 을 던지지 않는다는 약속이다.

**move 생성자에서 noexcept 의 중요성**

move 생성자는 어떠한 자원을 복사없이 stealing 하는 것이기 때문에 중간에 exception 이 발생할 시 복구할 수 없는 에러이다. move 생성자가 noexcept 로 선언되지 않을 시 vector 같은 자료구조는 복사생성자로 fallback 한다.

### move 할당자

```cpp
ClassName& operator=(ClassName&& other) noexcept;
```

- move 생성자와 비슷하지만 할당되는 객체가 자원을 가지고 있을 수 있기 때문에 먼저 자원을 정리하고 할당한다.

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

- 위에 보면 std::move 가 있는데 이는 move를 하는것은 아니고 lvalue 를 rvalue reference 로 변환하는 함수다.

### 핵심 요약

- move 생성자와 할당자를 선언할 떄는 noexcept 가 중요하다.
- 5의 법칙: 모던 c++ 에서 클래스가 직접 관리하는 자원이 있다면 5가지 (소멸자, 복사 생성자, 복사 할당자, 이동 생성자, 이동 할당자) 를 모두 구현해야 한다. + 기본생성자가 C++11 의 Orthodox Canonical class form (OCCF) 이다
- std::move 는 lvalue 를 rvalue reference 로 변환한다.

## Scoped enums

### C++11 이전의 enum

```cpp
enum Color { RED, GREEN, BLUE };
enum Alert { GREEN, YELLOW, RED }; // ERROR
```

- C-style enum 이다.
- Color 의 RED, 와 GREEN 이 Alert 와 겹치고 두 enum 은 같은 scope 를 공유하기 때문에 에러가 발생한다

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

또 다른 문제점은 위와 같은 코드에서 살펴 볼 수 있는데 APPLE 과 FORD 는 다른 enum 임에도 불구하고 묵시적 타입 변환이 가능하고 서로다른 enum 의 비교가 가능했다.

### C++11 scoped enum

```cpp
enum class Color { Red, Green, Blue };
enum class Alert { Green, Yellow, Red }; // Perfectly fine!

Color myColor = Color::Red;
Alert myAlert = Alert::Red; // No confusion here.
```

- enum class 가 생기면서 namespace 를 명시하여 사용해야 하며 scope 도 분리되었다.

```cpp
enum class Fruit { Apple, Banana };
enum class Car { Ford, Toyota };

Fruit mySnack = Fruit::Apple;
// int number = mySnack;           // ERROR! Cannot implicitly convert.
// if (Fruit::Apple == Car::Ford)  // ERROR! Cannot compare different enum classes.

// If you REALLY need the integer value, you must be explicit:
int number = static_cast<int>(mySnack);
```

위에서 얘기했던 묵시적 타입 변환, 서로 다른 enum 비교와 같은 문제가 해결되었다.

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

- C-style enum 같은 경우 enum 이 차지하는 메모리 크기를 알 수 없기 때문에 전방선언이 힘들었다고한다.
- c++ 에서는 크기를 지정할 수 있다.

## constexpr 과 literal types

const_expr 은 C 에서는 존재하지 않았던 처음보는 함수다. compile time optimization 을 활용한, 컴파일 타임에 알 수 있는 값들을 먼저 계산하고 코드에 박아 넣는 형식이다. 

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

- 위와 같이 constexpr 함수이더라도 runtime 에만 알 수 있는 값은 runtime 에 계산된다.

이와 같이 복잡한 객체다 동적 메모리할당이 필요한 std::string, std::vector 에는 사용될 수 없다.

### Literal types

Literal type 은 컴파일 타임에 생성, 조작, 소멸될 수 있는 간단한 타입이다.

1. **Scalar types:** `int`, `float`, `double`, `char`, `bool`, etc.
2. **References** (`int&`) and **Pointers** (`int*`).
3. **Arrays** of literal types.
4. **Classes / Structs**, but *only* if they meet strict rules.

이들 모두 literal type 이다.

### 클래스가 literal type 이기 위한 룰

- 소멸자가 기본 소멸자이어야 한다.
- `constexpr` 생성자가 하나라도 있어야 한다. (compile time 에 어떻게 생성할지 컴파일러가 알아야 함)
- 멤버 변수들도 다 literal type 이어야 한다.
- 가상 함수나 가상 base class 를 포함할 수 없다.

```cpp
constexpr int compile_time_val = square(5); 

1184:    c7 45 f0 19 00 00 00    movl   $0x19,-0x10(%rbp)
```

- 위 코드는 5 * 5 를 `0x19` 10진수로 15로 바꿔서 넣었다는걸 볼 수 있다.

## list initialization 리스트 초기화

리스트 초기화는 중괄호 {} 를 사용하여 변수나 객체를 초기화하는 방식이다.

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

언뜻보면 이게 왜 좋지? 라는 생각이 들 수 있다.

### 축소 변환 방지

```cpp
// 절삭
int x = 4.9;
int y(4.9);

// 컴파일 에러!
int z{4.9};
```

- 컴파일 에러가 발생 → 축소 변환 방지가 된다.

### Most Vexing Parse 가장 짜증나는 해석?

```cpp
class Timer {};

// You might think this creates a Timer object named 't'...
Timer t(); 

// ...but it actually declares a function named 't' that returns a Timer!
```

- 오래된 컴파일러는 Timer t()와 같은 문구가 함수를 부르는지 객체를 만드는지 구분하지 못한다.

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

- push_back 과 같은 슬픈 초기화 방법을 사용할 필요가 없어진다.

## 위임 생성자와 상속 생성자

### 위임 생성자

```cpp
class Player {
    int hp;
    int mp;
    int level;

public:
    // 타겟 생성자 (모든 초기화를 실제로 담당하는 메인 생성자)
    Player(int h, int m, int l) : hp{h}, mp{m}, level{l} {
        // 복잡한 초기화 로직이 있다고 가정
    }

    // 위임 생성자 (타겟 생성자를 호출하여 초기화를 떠넘김)
    Player() : Player(100, 50, 1) {} 
    
    // 또 다른 위임 생성자
    Player(int h) : Player(h, 0, 1) {} 
};

// 사용:
Player p1{};      // hp:100, mp:50, level:1
Player p2{200};   // hp:200, mp:0, level:1
```

```cpp
//위는 다음과 같이 해석 하면된다.
Player() 호출 시 -> Player(100, 50, 1) 로 호출
```

- 위를 이해한다면 왜 위임 생성자인지 알 수 있을 것 이다.

### 상속 생성자

```cpp
class Weapon {
public:
    Weapon(int damage) { /* ... */ }
    Weapon(int damage, float speed) { /* ... */ }
    Weapon(std::string name, int damage, float speed) { /* ... */ }
};

class Sword : public Weapon {
public:
    // C++11 이전의 방식 (불필요한 반복 코드)
    // Sword(int damage) : Weapon(damage) {}
    // Sword(int damage, float speed) : Weapon(damage, speed) {}
    // Sword(std::string name, int damage, float speed) : Weapon(name, damage, speed) {}

    // C++11 상속 생성자 (단 한 줄로 해결!)
    using Weapon::Weapon; 
};

// 사용:
Sword basic_sword{10}; 
Sword fast_sword{15, 1.5f};
Sword excalibur{"Excalibur", 100, 2.0f};
```

- C++11 이전에 클래스를 상속하려면 생성자를 하나하나 붙여줘야 됬지만 이제 `using Weapon:Weapon` 단 하나의 줄로 처리 가능하다.

## 중괄호 / 괄호 초기화

```cpp
class Character {
    // Brace-or-equal initializers 사용
    int hp = 100;          // 등호(=) 사용
    int mp{50};            // 중괄호({}) 사용 (Uniform Initialization)
    std::string name = "NPC"; 

public:
    // 1. 기본 생성자: 위에서 지정한 기본값(100, 50, "NPC")이 자동으로 들어갑니다.
    Character() {} 

    // 2. 오버로딩된 생성자: hp와 name만 바꾸고, mp는 기본값(50)을 그대로 씁니다.
    Character(int h, std::string n) : hp{h}, name{n} {} 
};
```

- 클래스 내부에서 바로 초기화를 가능케 한다.

C++11 이전에는 생성자로만 초기화를 해야했기 때문에 default 값이 들어가는 모든 생성자에 initializer list 를 사용해야 하는 불편함이 있었다.

```cpp
// C++11 이전 (불편한 방식)
class Box {
    int width, height, color;
public:
    Box() : width(10), height(10), color(0) {} 
    Box(int w, int h) : width(w), height(h), color(0) {} // color(0) 중복!
};

// C++11 이후 (Brace-or-equal 사용)
class Box {
    int width = 10;
    int height = 10;
    int color = 0;   // 여기서 한 번만 지정하면 끝!
public:
    Box() {}
    Box(int w, int h) : width{w}, height{h} {} // color는 자동으로 0이 됨
};
```

초기화가 중앙화 되면서 초기화 누락도 방지할 수 있고 가독성 측면에서도 좋아졌다고 볼 수 있다.

## nullptr

nullptr 은 많이들 알고 있을것 같다. C++11 이전에는 `NULL` 이나 `0` 을 사용했었고 이는 다음과 같은 문제를 야기한다.

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

위와 같이 `doSomething(NULL)` 콜을 하게되면 preprocessor 가 `NULL` 을 `0` 으로 대체해버리기 때문에 `int *ptr` 버전을 부르려면 밖에서 `(int *)` 타입 캐스팅을 해야하는 단점이 있었다.

```cpp
int *x = nullptr;
doSomething(nullptr);
```

- nullptr 는 캐스팅 없이 ptr 값에 0을 넣을 수 있다.
