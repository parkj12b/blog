---
title: "C++ 표준 (1) (C++11)"
date: 2026-05-06T21:10:00+09:00
lastmod: 2026-05-12T00:00:00+09:00
draft: false
categories: ["Programming"]
tags: ["C++", "C++11", "Programming"]
featureimage: "cpp-logo.png"
---

C 표준 98과 11을 쓰면서 C표준에 대해서는 어느정도 지식을 가지고 있던게 사실이다. 하지만 C++ 은 평소에 많이 쓰지 않기 때문에 항상 코테 풀 때 쓰는 문법만 쓰게 된다. 

이번 시리즈에서는 모던 C++, C++11 을 모던하다고 하기는 좀 그렇지만…, 훑어 보려고 한다.

내용은 [cppreference.com](http://cppreference.com) 을 위주로 설명하겠다.

# C++11

## `auto` and `decltype` / `decltype(auto)` (c++14)

### auto

```cpp
#include <iostream>

int main() {
    int x = 10;
    const int& cx = x; // cx is a constant reference to x

    // Type deduction with auto:
    auto a = cx;       // 'a' is deduced as 'int' (const and & are stripped). It's a copy.
    a = 20;            // Allowed! 'a' is a separate variable. 'x' is still 10.

    // If you WANT a reference or const, you have to ask for it explicitly:
    auto& b = cx;       // 'b' is deduced as 'const int&'
    const auto& c = x;  // 'c' is deduced as 'const int&'

    return 0;
}
```

- type 대신에 auto 를 사용할 수 있다.
    - 한눈에 봐도 타입을 유추할 수 있을때 쓰는게 좋다고 생각한다.
    - `auto a = cx` 는 명시적이지 않기 때문에 이런 경우는 사용하지 않는게 좋은거 같다.
- 컴파일 타임 추론이다
- `const` 와 `&` 참조형을 없애기 때문에 조심해야 한다

```cpp
#include <iostream>
#include <vector>

std::vector<int> numbers = {10, 20, 30};

// Returns a reference to the element
int& getElement(int index) {
    return numbers[index];
}

// Wrapper function using auto (BUGGY if we want to modify the array)
auto badWrapper(int index) {
    return getElement(index); // 'auto' strips the &, so this returns a COPY of the int
}

// Wrapper function using decltype(auto) (PERFECT)
decltype(auto) goodWrapper(int index) {
    return getElement(index); // Deduces 'int&', perfectly forwarding the reference
}

int main() {
    goodWrapper(1) = 99; // Modifies numbers[1] to 99
    // badWrapper(1) = 99; // ERROR: Can't assign to a temporary copy
    return 0;
}
```

- `decltype` 은 “declared type”, 선언 타입의 약자이다. expression 을 평가하지 않고 type 을 리턴한다.
- `decltype` 을 사용하면 auto 와 달리 const 나 & 를 없애지 않기 때문에 위와 같은 코드에 적합하다.

### decltype

```cpp
#include <iostream>

int main() {
    int x = 10;
    const int& cx = x;

    // Type deduction with decltype:
    decltype(cx) d = x; // 'd' is deduced EXACTLY as 'const int&'
    
    // d = 20; // ERROR! d is const, you cannot modify it.

    // Another example: inspecting an expression
    int y = 5;
    decltype(x + y) z = 15; // 'z' is deduced as 'int' because (x + y) results in an int
    
    return 0;
}
```

```cpp
The placeholder auto may be accompanied by modifiers, such as const or &, which will participate in the type deduction. The placeholder decltype(auto) must be the sole constituent of the declared type.(since C++14)
```

```cpp
int x = 42;
int* ptr = &x;

const auto&  ref = x;   // Deduces auto = int, result is const int&
auto*        p   = ptr; // Deduces auto = int, result is int*
const auto*  cp  = ptr; // Deduces auto = int, result is const int*
auto&&       forwarding_ref = x; // Deduces auto = int& (via reference collapsing)
```

- 위와 같이 auto 같은 경우 const 나 &랑 같이 쓸 수 있다.

```cpp
int x = 10;
const int& get_ref() { return x; }

// --- VALID USAGE ---
decltype(auto) a = get_ref(); // 'a' is EXACTLY const int&

// --- INVALID USAGE (Compiler Errors) ---
// const decltype(auto) b = get_ref(); 
// decltype(auto)& c = get_ref();

int main() {
    int x = 5;

    decltype(auto) y = x;   // y is 'int'
    decltype(auto) z = (x); // z is 'int&' (a reference to x!)

    z = 10; // This changes x to 10
}
```

- `decltype(auto)` 는 auto 와 다르게 const 나 & 와 사용이 불가능하다.

## `defaulted` and `deleted` 함수

### defaulted

```cpp
class NetworkPacket {
public:
    int payloadSize;

    // Custom constructor
    NetworkPacket(int size) : payloadSize(size) {} 

    // Because we wrote the one above, we lost the default constructor.
    // NetworkPacket() {} // We COULD write this, but there's a better way:
    
    NetworkPacket() = default; 
};
```

- c++ 에서는 class 의 생성자를 명시적으로 선언하면 default 생성자는 생성되지 않는다.
- default 생성자가 필요할 경우 명시적으로 생성자 오버로딩을 할 수 있지만 C++11 은 default 키워드를 추가함으로써 명시적으로 default 함수를 사용할 수 있게 한다.

### delete

```cpp
#include <iostream>

class TcpSocket {
private:
    int socket_fd;

public:
    TcpSocket() {
        // Assume this opens a real socket
        socket_fd = 10; 
    }

    ~TcpSocket() {
        // Close the socket
    }

    // --- C++11 Resource Protection ---
    // Ban copying! You cannot clone a hardware socket.
    TcpSocket(const TcpSocket& other) = delete;            // Copy constructor deleted
    TcpSocket& operator=(const TcpSocket& other) = delete; // Copy assignment deleted

    // (In a real system, you would provide move constructors here instead)
};

int main() {
    TcpSocket server;
    
    // TcpSocket client = server; // COMPILE ERROR: Call to deleted constructor!
    
    return 0;
}
```

- `delete` 키워드를 사용하여 함수 정의를 명시적으로 삭제할 수 있다.

```cpp
= delete ("에러 에러 에러");
```

- 위와 같이 string-literal 을 넘겨서 에러 메세지를 같이 줄 수 있다. (C++26)

## `final` and `override`

### final

상속이나 오버라이딩을 방지한다.

### final (class)

```cpp
class Base final {
    // ...
};

// ERROR: Cannot inherit from a 'final' class.
class Derived : public Base { 
};
```

### final (virtual function)

```cpp
class Base {
public:
    virtual void process() { /* ... */ }
};

class Derived : public Base {
public:
    // Overrides Base::process, but prevents further overriding
    void process() override final { /* ... */ } 
};

class DeepDerived : public Derived {
public:
    // ERROR: 'process' was marked final in 'Derived'.
    // void process() override { /* ... */ } 
};
```

### override

```cpp
class Base {
public:
    virtual void doSomething(int x) const { /* ... */ }
};

class Derived : public Base {
public:
    // ERROR: Missing 'const'. The compiler catches this because of 'override'.
    // void doSomething(int x) override { /* ... */ } 

    // ERROR: Wrong parameter type (float instead of int).
    // void doSomething(float x) override { /* ... */ }

    // CORRECT: Matches the base class signature exactly.
    void doSomething(int x) const override { /* ... */ } 
};
```

- 오버라이딩 하는 함수의 오타나 virtual 함수의 function signature mismatch 와 같은 부분들을 컴파일러 에러로 잡아준다.

## Trailing return type 후위 반환 타입

코드를 보면 이해가 빠르다.

### 기존방식

```cpp
int add(int a, int b) {
    return a + b;
}
```

### Trailing return type

```cpp
auto add(int a, int b) -> int {
    return a + b;
}
```

- python 의 type hint 와 비슷한 문법이다.

### 기존 방식의 문제점

후위 반환 타입이 도입된데에는 template 과 decltype 에 있다.

```cpp
// ERROR: 'a' and 'b' have not been declared yet! 
// The compiler reads left-to-right.
template <typename T, typename U>
decltype(a + b) add(T a, U b) {
    return a + b;
}
```

- 위 템플릿 함수를 보면 decltype(a+b) 를 사용하여 a+b 의 타입을 컴파일러가 자동으로 알아낼 수 있게 한다.
- 문제는 `decltype(a + b)` 부분에서 `T a, U b` 가 아직 선언되지 않았다는 점이다.
- 컴파일러는 왼쪽에서 오른쪽으로 파싱하기 때문에 오른쪽에 타입을 놓아야한다.

```cpp
// ✅ Works in C++14 and later (Recommended)
template <typename T, typename U>
auto add(T a, U b) {
    return a + b;
}
```

- C++14 같은 경우 return 구문에서 type 을 유추할 수 있고 이 방식이 추천된다고 한다.

이와 비슷하게 LR 파싱을 인한 기이한 문법 구조는 java 에서도 찾아볼 수 있는데 다음과 같은 예 가 있다.

```java
obj.<T>method()
```

- `LT` 나 `GT` 와 헷갈리지 않기 위해 C++ 과 다르게 함수 호출 타입을 왼쪽에 적는다.

## rvalue references (rvalue 참조)

lvalue 와 rvalue 는 프로그래밍 언어론에서 사용되는 용어이다. 프로그래밍을 한다고 해서 꼭 알아야 하는 용어는 아니지만 c++ 같은 언어를 다루다보면 종종 등장하는 언어이다. 필자가 진행했었던 B → x86 컴파일러 프로젝트에서도 많이 다루었고 기본적으로 프로그래밍 언어론에서 등장하기도 한다.

관심이 있다면 아래 B reference manual 을 찾아보기 바란다.

[](https://www.nokia.com/bell-labs/about/dennis-m-ritchie/kbman.html)

### Lvalue

lvalue 는 메모리 상에 특정한 주소를 가지는 지속적인 값이다.

```java
int x = 5; // lvalue = x, rvalue = 5
```

위 코드를 보면 좀 더 와닿는데 = 의 왼쪽에 올 수 있는 값이 lvalue 라고 보면된다.

```java
int y = x;
```

하지면 위와 같이 x 가 값으로 사용되는 경우도 있다. 이 경우 C++ 에서 Lvalue-to-Rvalue 변호나을 진행하는 것 이라고 보면 된다.

```java
&x;
&(x + 5);
&"hello"
```

c나 c++ 에서 lvalue 는 &를 통해 어느 메모리 위치에 저장되어 있는지 알 수 있다.

- 이걸 모르는 사람은 없을거라 생각되나 “hello” 와 같은 string literal 은 프로그램의 rodata (read-only data) 섹션에 저장된다. 즉, 메모리 주소를 갖는다.

```cpp
#include<iostream>

int main() {
    std::cout << "The memory address of string 'hello world': " << &"hello world" << std::endl;
}

//output
//The memory address of string 'hello world': 0x62c6e3348035

```

![image.png](image.png)

- 위와 같이 .rodata 섹션에 있는것 까지 확인해 봤다.
    - C-style string 이기 떄문에  Null (00) 값으로 분리되어 있다.

### rvalue

rvalue 는 주소를 특정할 수 없는 임시로 존재하는 값이다.

```java
int x = 10; // 10은 임시 값이다
int y = x + 5; // x + 5 = 15 이고 연산으로 생긴 임시 값이다.
"hello" // hello
int result = calculate();
```

- 위 함수 calculate() 의 리턴값도 rvalue 이다.

### C++11 이전 문제

C++11 이전에 참조형을 쓰려면 lvalue 만 bind 할 수 있었다.

```cpp
int& ref = x;       // ✅ OK: ref binds to lvalue 'x'
int& ref2 = 10;     // ❌ ERROR: Cannot bind lvalue reference to rvalue '10'
const int& ref = 10;
```

- 위 처럼 const 에 rvalue 를 bind 할 수는 있지만 const 이기 때문에 값을 바꿀수 없다.

```cpp
Vector v2 = createHugeVector();
```

위에 코드는 c++ 에서 다음과 같이 동작한다.

1. rvalue vector 가 생성
2. v2 백터로 copy
3. rvalue vector 소멸

### C++11 해결책: rvalue Reference

```cpp
int&& r_ref = 10;        // ✅ OK: rvalue reference binds to temporary rvalue
int&& r_ref2 = x + 5;    // ✅ OK: binds to the temporary result of x + 5
int&& r_ref3 = x;        // ❌ ERROR: Cannot bind rvalue reference to an lvalue
```

- temp value 를 소멸하는 대신 할당한다

```cpp
class MyVector {
    int* data;
public:
    // 1. Traditional COPY Constructor (Takes a const lvalue reference)
    MyVector(const MyVector& other) {
        // Must allocate new memory and copy everything slowly
        data = new int[1000000];
        // ... copy items one by one ...
    }

    // 2. C++11 MOVE Constructor (Takes an rvalue reference &&)
    MyVector(MyVector&& other) noexcept {
        // FAST: "Steal" the pointer from the temporary object
        data = other.data; 
        
        // Remove the data from the temporary so its destructor 
        // doesn't delete the memory we just stole!
        other.data = nullptr; 
    }
};
```

```cpp
#include <iostream>

class Buffer {
public:
    int* data;

    // 1. Normal Constructor
    Buffer(int val) {
        data = new int(val);
        std::cout << "   [Created] Buffer at: " << data << "\n";
    }

    // 2. COPY Assignment Operator (Takes a normal lvalue reference)
    Buffer& operator=(const Buffer& other) {
        delete data;                 // Clean up our current memory
        data = new int(*other.data); // 🔴 ALLOCATE NEW MEMORY
        std::cout << "   [COPY Assignment] 🔴 Allocated new memory at: " << data << "\n";
        return *this;
    }

    // 3. MOVE Assignment Operator (Takes an rvalue reference &&)
    Buffer& operator=(Buffer&& other) noexcept {
        delete data;          // Clean up our current memory
        data = other.data;    // 🟢 STEAL THE EXACT MEMORY ADDRESS
        other.data = nullptr; // Hollow out the temporary
        std::cout << "   [MOVE Assignment] 🟢 Stole exact memory at:   " << data << "\n";
        return *this;
    }

    ~Buffer() { 
        delete data; 
    }
};

// A function that returns a temporary Buffer
Buffer createTemporary() {
    return Buffer(999);
}

int main() {
    Buffer targetBox(10);
    Buffer namedBox(20);

    std::cout << "\n--- SCENARIO 1: Assigning a named variable (Lvalue) ---\n";
    // namedBox is an lvalue. The compiler MUST copy it.
    targetBox = namedBox; 

    std::cout << "\n--- SCENARIO 2: Assigning a raw temporary (Rvalue) ---\n";
    // Buffer(30) is a temporary. The compiler AUTOMATICALLY calls the && Move Assignment!
    // No std::move() is used here!
    targetBox = Buffer(30); 

    std::cout << "\n--- SCENARIO 3: Assigning from a function return (Rvalue) ---\n";
    // createTemporary() returns a temporary. It AUTOMATICALLY triggers the && move!
    targetBox = createTemporary();

    return 0;
}
```

```cpp
   [Created] Buffer at: 0x60532c487020
   [Created] Buffer at: 0x60532c487450

--- SCENARIO 1: Assigning a named variable (Lvalue) ---
   [COPY Assignment] 🔴 Allocated new memory at: 0x60532c487020

--- SCENARIO 2: Assigning a raw temporary (Rvalue) ---
   [Created] Buffer at: 0x60532c487770
   [MOVE Assignment] 🟢 Stole exact memory at:   0x60532c487770

--- SCENARIO 3: Assigning from a function return (Rvalue) ---
   [Created] Buffer at: 0x60532c487020
   [MOVE Assignment] 🟢 Stole exact memory at:   0x60532c487020
```

- Scenario 1: memory reusage
    - 메모리 할당기가 메모리를 재사용 하여 같은 메모리가 나옴
- Scenario 2: targetBox = Buffer(30) 으로 …7020 은 소멸
    - …7770 이 할당된 temp 메모리를 그대로 targetBox 로 가져옴
- Scenario 3: 소멸된 …7020 이 createTemporary() 에서 할당
    - 자동으로 move 사용 ⇒ targetBox  = …7020
    - return 된 temp value 는 rvalue 이기 때문에 자동으로 move operator 호출.
