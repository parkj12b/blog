---
title: "C++ 표준 (3) (C++11)"
date: 2026-05-10T14:49:00+09:00
lastmod: 2026-05-12T00:00:00+09:00
draft: false
categories: ["Programming"]
tags: ["C++", "C++11", "Programming"]
featureimage: "cpp-logo.png"
---

[C++ 표준 (2) (C++11) ]({{< ref "cpp-standard-2-cpp11" >}})

## 타입 별칭

타입 별칭은 코테문제를 풀떄도 자주 애용해오던 기능이다. 새로운 타입을 만드는게 아닌 기존의 타입에 대한 별칭을 설정할 수 있게 해준다.

```cpp
using NewName = ExistingType;
```

```cpp
using tdii = tuple<double, int, int>;

// 굉장히 길다...
priority_queue<tuple<double, int, int>, vector<tuple<double, int, int>>, greater<tuple<double, int, int>>> pq1;

// 간결해 진다 :)
priority_queue<tdii, vector<tdii>, greater<tdii>> pq2;

// C style
typedef void (*Callback)(int, const std::string&);

// with type alias
using Callback = void (*)(int, const std::string&);

```

### 템플릿 타입 별칭

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

- 템플릿 타입 별칭도 가능하다.

## 가변 길이 템플릿 (parameter pack)

C++11 이전에 템플릿은 고정길이 변수만 받을 수 있었다. C++11 에서 parmeter pack 이 도입 됨으로써 가변길이 템플릿이 가능해졌다.

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

- `typename...` 오는 부분을 parameter pack 이라고 한다.
    - 0 개 이상의 인자를 받을 수 있다.
- compile 타임에 이 함수를 부르는 type 을 체크하고 그에 상응하는 함수를 컴파일러가 만들게 된다.
- 파라미터 팩이 있는 함수와 겹칠수 있는 함수가 있는 경우 호출시 파라미터 팩이 없는 함수가 우선순위를 갖게된다.

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

- 위와 같이 함수 시그니처가 겹칠 수 있는 템플릿을 선언 할 경우 파라미터 팩을 나중에 두어야 한다. 그렇지 않을 경우 compiler 에러가 발생한다.

## Generalized Union

union 같은 경우 같은 메모리공간에 서로 다른 타입이 올 수 있게 함으로써 데이터 공간을 효율적이게 쓸 수 있도록 도와준다.

C++11 이전에 Union 같은경우 Plain Old Data (POD) type (int, float, raw pointer 등등) 밖에 넣을 수 없었다. Generalized Union 이 추가되면서 string 과 같은 complex type 들도 union 에 넣을 수 있게 되었다.

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

여기서 complex type 을 사용하기 위해 필요한 두가지가 있다.

1. placement `new` 의 사용
2. 명시적 소멸자 호출

### placement `new`

기본적인 `new` 같은 경우 `heap` 에서 memory 할당후 객체를 생성하지만 placement `new` 같은 경우 메모리 주소를 넘겨줌으로써 memory 할당을 skip 하고 객체를 만든다.

```cpp
#include <new>

// new (memory_address) Type(arguments...);
```

placement `new` 를 사용하기 위해서는 `<new>` 헤더를 include 해야한다.

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

Plain Old Data POD 는 위에서도 언급했듯이 C-style struct 이다. C++11 위원회는 POD 의 조건이 너무 제한되어 있고 두개의 다른 개념을 강제하고 있다고 생각했다.

1. 이 객체가 `memcpy()` 로 안전하게 복사될 수 있는지
2. C struct 와 정확히 동일하게 메모리에 놓여져 있는지

POD 를 일반화하기 위해서 이 개념을 두가지 컨셉으로 만들었다.

### Trivial Types

Trivial Types 은 `memcpy()` 로 안전하게 복사할 수 있는 type 을 말한다.

클래스나 구조체가 Trivial 하기 위해서는 아래의 것들을 금지한다

- 사용자 정의 / ~ 의 삭제
    - 생성자
    - 복사, 이동 생성자
    - 소멸자
- 가상 함수 또는 가상 기초 클래스

### Standard Layout Types

타입이 차지하는 메모리 공간의 크기를 예측할 수 있으면 그것을 Standard Layout Type 으로 분류한다.

Standard Layout Types 는 다음과 같은 것들을 가질 수 있다.

- 사용자 정의 생성자 소멸자
- 멤버 함수
- `private` / `protected` 멤버 (모든 non-static member 들이 같은 access type 을 가지고 있어야 한다. private 과 public 을 섞을 수 없다.)
- 기초 클래스 (모든 데이터 멤버는 하나의 클래스에 선언되어 있어야 한다. [derived data, base empty] or [derived empty, base data] )

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

### 분리의 의미

Trivial Type 과 Standard Layout Type 을 나눔으로써 각각 상황에 맞게 필요한 타입을 알 수 있게 되었다.

- C API 나 OS 구조체와의 통신 - `std::is_standard_layout`
- 네트워크 통신 혹은 큰 데이터를 `memcpy()` 로 빠르게 옮기고 싶을떄 - `std::is_trivial`

## 사용자 정의 literal

보통 literal 이라고 하면 우리는 `100`, `100.1`, `100.1f`, `100UL` 과 같은 값을 값을 떠올린다.

```cpp
ReturnType operator"" _suffix(Parameters);
```

- 사용자 정의 literal 은 `_` underscore 로 시작해야 한다.
- 사용할 수 있는 Parameter 의 타입은 다음과 같다
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

## Attributes 속성

컴파일러에게 추가적인 정보를 넘겨줄 때 사용된다. 기본적으로는 컴파일러 확장이었고 GCC/Clang 에서는 `__attributes__((...))` MSVC 에서는 `__delspec(...)` 을 사용한다.

```cpp
[[attribute_name]] void myFunction();
```

```cpp
// A compiler-specific attribute using the C++11 standard syntax
[[gnu::always_inline]] void fastFunction();
```

컴파일러는 컴파일러만의 네임스페이스 스코프를 가질 수 있다.

자주 사용하는 속성은 다음과 같다.

- `[[noreturn]]`
- `[[deprecated(”message”)]]` C++14
- `[[fallthrough]]` , `[[nodiscard]]` ,`[[maybe_unused]]` C++17
- `[[likely]]`, `[[unlikely]]` C++20

## 람다식

다른 많은 언어들의 람다식, 익명함수와 같다 보면 된다.

```cpp
[captures] (parameters) -> return_type { 
    // function body 
}
```

- [captures]: 주변에 있는 어떤 변수들이 람다식 내에서 이용가능 해지는지를 정한다.
- (parameters): 일반 함수와 같은 매개변수
- return_type: trailing return type. 보통 컴파일러는 return 값의 타입을 추론할 수 있기 떄문에 생략할 수 있다.

### 기본 람다식

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

### captures 예제

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

C++11 에서는 noexcept 를 지정자와 연산자로 사용할 수 있다.

### noexcept 지정자

```cpp
void doSomethingFast() noexcept; // Promises not to throw
void doSomethingElse() noexcept(true); // Same as above
void riskyFunction() noexcept(false); // Can throw (same as omitting noexcept)
```

noexcept 로 지정된 함수가 예외를 던지게 되면 프로그램은 terminate() 함수를 호출하고 바로 종료된다.

### noexcept 연산자

```cpp
constexpr bool is_safe = noexcept(doSomethingFast()); // is_safe == true
```

사용법이 다르지만 noexcept 지정자와 다르지 않다.

noexcept 를 사용해야 하는 곳:

1. 이동 생성자, 이동 할당자
2. 소멸자 (묵시적으로 noexcept 가 기본)
3. Swap 함수
4. exception 을 던지지 않는 함수들

noexcept 를 사용하면 안되는 곳:

1. 동적 메모리 할당
2. 레거시 C++ / 외부 라이브러리 코드
3. exception 을 던지는 함수

## Alignas, Alignof

C11 에서도 다뤘던 함수들이다. 메모리 정렬 컴파일러 속성에서 C++11 로 추가되었다. 여기서 다시 다루지는 않고 추가적인 내용만 기록하겠다.

Alignas, Alignof 는 아시다싶이 low-level, data oriented design, performance critical application 에서 자주 사용된다.

- Cache Optimization
- SIMD instruction
- DMA and Embedded Hardware

C++11 에서 alignas 를 new 로 할당할 경우 기본 메모리 할당자는 무시할 수 있다. 그럴 경우 `posix_memalign()` 또는 `_aligned_malloc()` 과 같은 함수들을 대신 사용해야 한다.

<aside>
💡

C++17 에서 alignedment-aware allocator for new 연산자가 나오면서 수정되었다.

</aside>

### C11 과의 차이점

C++ 에서 alignas 와 alignof 는 키워드이지만 C11 에서는 앞에 underscore 가 붙은 `_Alignas` 와 `_Alignof` 로 쓴다.

C11 에서 C++ 스타일로 사용하려면 `<stdalign.h>` 를 추가할 수 있다.

## 멀티쓰레드 메모리 모델

C++11 이전에는 모든 application이 하나의 쓰레드에서 동작한다고 가정했다. C++11 에서는 `std::atomic` 이 

[](https://en.cppreference.com/cpp/atomic/memory_order)

자세한건 위에 링크를 찾아보기 바란다. gcc 컴파일러 확장에도 __sync 와 modern 버전인 __atomic 이 있다. __atomic 같은 경우 C++11 의 메모리 모델과 1대1 대응되게 설계되었다. 컴파일러 확장보다는 C++ 의 기능을 사용하는게 지향된다고 한다.

## Thread-local-storage

Thread Local Storage (TLS) 는 `thread_local` 키워드로 변수를 선언 함으로써 각각의 쓰레드가 서로 다른 메모리 변수를 갖게 된다.

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

지역변수를 사용하는건 mutex 를 잡거나 atomic 을 이용한 cache-line bouncing 보다는 훨씬 빠르지만 공짜는 아니다. 기볹거인 local 과 global 변수를 참조하는 것 보다 느리다.

### x86-64 구현

구현은 다음과 같다.

1. 컴파일러는 프로그램의 .tdata 섹션과 .tbss 섹션에 `thread_local` 변수를 선언한다.
2. 운영체제는 특정한 CPU 레지스터를 TLS block 을 가르키게 한다. `FS` 혹은 `GS`
3. 변수에 접근할 때 컴파일러는 `FS` 레지스터에서 오프셋을 통해 접근한다.

.tdata 와 .tbss 섹션은 read-only 이다. 새로운 thread 가 spawn 하면 새로운 block 을 할당한다.

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

다른 언어의 range in 이나 for in 과 비슷하다.

주의해야할 점은 iterator 를 사용할 때와 마찬가지로 컨테이너의 형태를 수정하면 안된다.

## static_assert

컴파일 타임에 할 수 있는 assert 다.

```cpp
static_assert(sizeof(int) >= 4, "Integers must be at least 32 bits on this platform.");
```

### 예제

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
