+++
title = "C11 표준 기능"
date = "2026-04-22T15:52:00Z"
draft = false
tags = ["C11", "C"]
categories = ["Programming"]
featureimage = "featured.svg"
+++

{{< linkcard 
    title="C11 (C standard revision) - Wikipedia" 
    description="C11 표준은 정렬 사양, 멀티스레딩 등과 같은 기능들을 C 언어에 도입했습니다." 
    url="https://en.wikipedia.org/wiki/C11_(C_standard_revision)" 
>}}

지난 시리즈인 [Standard features of C99]({{< ref "Standard_features_of_c99" >}})에 이어, C11의 새로운 기능들을 계속해서 살펴보겠습니다.

# C99에서의 변경 사항

## 정렬 사양 (Alignment specification)

### 구성 요소

- **`_Alignas`**: 변수나 구조체 멤버를 특정 바이트 경계에 강제로 맞추는 데 사용되는 지정자입니다.
- **`_Alignof`**: 특정 타입의 정렬 요구 사항을 반환하는 연산자입니다 (`sizeof`와 유사).
- **`aligned_alloc`**: 특정 정렬 경계에 메모리를 동적으로 할당하기 위한 표준 라이브러리 함수입니다.

### 기억해야 할 중요한 규칙들

1. **2의 거듭제곱만 가능:** 유효한 정렬 요구 사항은 항상 2의 거듭제곱(**1, 2, 4, 8, 16, 32, 64** 등)입니다. 3바이트나 5바이트 정렬은 요청할 수 없습니다.
2. **크기 배수:** `aligned_alloc(alignment, size)`를 사용할 때, `size` 파라미터는 반드시 `alignment` 파라미터의 수학적 배수여야 합니다. 64바이트 정렬을 요청한다면 크기는 64, 128, 192 등이 되어야 합니다. 그렇지 않으면 함수의 동작은 정의되지 않습니다 (종종 `NULL`을 반환합니다).
3. **표준 해제:** C11에서는 특별한 "정렬된 해제" 함수가 필요하지 않습니다. `aligned_alloc`에 의해 생성된 포인터는 표준 `free()` 함수에 전달해도 안전합니다.

### 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <stdalign.h> // 'alignas' 및 'alignof' 매크로 제공

// 구조체 멤버에 alignas를 사용하여 전체 구조체의 정렬을 강제함
// 64바이트는 현대 CPU 캐시 라인의 일반적인 크기입니다.
struct CacheLineData {
    alignas(64) int data[4]; 
};

int main(void) {
    // --- 1. alignof 연산자 ---
    // 타입의 기본 또는 강제된 정렬을 확인합니다.
    printf("Standard char alignment: %zu byte\n", alignof(char));
    printf("CacheLineData struct alignment: %zu bytes\n", alignof(struct CacheLineData));

    // --- 2. alignas 지정자 ---
    // 표준 char 배열(보통 1바이트 정렬)을 메모리의 32바이트 경계에 
    // 위치하도록 강제합니다.
    alignas(32) char buffer[128];
    printf("Custom local buffer alignment: %zu bytes\n", alignof(buffer));

    // --- 3. aligned_alloc 함수 ---
    // 사용법: aligned_alloc(alignment, size)
    // 중요한 규칙: 'size'는 반드시 'alignment'의 정수 배수여야 함
    size_t align = 64;
    size_t size = 256; // 256은 정확히 64의 4배입니다.

    void *ptr = aligned_alloc(align, size);

    if (ptr != NULL) {
        printf("\nDynamically allocated memory address: %p\n", ptr);
        
        // 정수 타입으로 캐스팅하여 수학적 정렬을 확인합니다.
        if (((uintptr_t)ptr % align) == 0) {
            printf("Success: The pointer is perfectly aligned to a %zu-byte boundary!\n", align);
        }
        
        // aligned_alloc으로 할당된 메모리는 free()로 정상적으로 해제됩니다.
        free(ptr);
    } else {
        printf("Memory allocation failed.\n");
    }

    return 0;
}
```

- aligned_alloc은 표준 메모리 할당자(ptmalloc, jemalloc 또는 tcmalloc)에 의해 구현됩니다.

### 커널 내부 정렬 (Kernel Internal Alignment)

```c
#define __aligned(x) __attribute__((__aligned__(x)))

// 커널에서의 사용 예:
struct network_data {
    int packet_count;
    char buffer[256] __aligned(64); 
};
```

- `__attribute__((__aligned__(x)))`는 컴파일러 지시어입니다.
    - 표준 C가 아님
    - GCC, Clang과 같이 GNU 구문을 지원하는 컴파일러에서만 작동함

### 슬픈 매크로들 (Sad Sad Macros)

```c
#if defined(__GNUC__) || defined(__clang__)
    #define MY_ALIGN(x) __attribute__((aligned(x)))
#elif defined(_MSC_VER) || __STDC_VERSION__ >= 201112L
    #define MY_ALIGN(x) _Alignas(x)
#else
    #define MY_ALIGN(x) // 오래되거나 알 수 없는 컴파일러를 위한 폴백
#endif
```

## _Noreturn / <stdnoreturn.h>

`void`를 위한 것이 아닙니다. 중간에 종료되거나 `exec` 또는 그와 동등한 동작을 수행하는 함수를 위한 것입니다.

- **데드 코드 제거 (Dead Code Elimination):** 컴파일러는 함수 호출 바로 다음에 작성된 코드가 도달 불가능하다는 것을 알고 바이너리에서 제거할 수 있습니다.
- **경고 억제:** 함수가 `int`를 반환해야 하지만 `_Noreturn` 함수를 호출하며 끝나는 경우, 컴파일러는 "반환 문 누락"에 대해 불평하지 않습니다.
- **스택 최적화:** 컴파일러는 "뒤로 가기 버튼"이 절대 눌리지 않을 것임을 알기 때문에 반환 주소를 저장하거나 특정 레지스터를 보존할 필요가 없습니다.

### 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdnoreturn.h> // 'noreturn'을 '_Noreturn'으로 정의

// 이 함수는 프로그램을 종료하거나 영원히 루프를 돕니다.
// main()으로 제어권을 다시 넘기지 않습니다.
noreturn void stop_everything(const char *msg) {
    fprintf(stderr, "Fatal: %s\n", msg);
    exit(1); 
}

int main(void) {
    printf("Initializing system...\n");

    int critical_error = 1;

    if (critical_error) {
        stop_everything("Hardware failure detected.");
        
        // 컴파일러는 이 부분의 코드가 "죽었다"는 것을 압니다.
        // 대부분의 컴파일러는 "코드가 실행되지 않음" 경고를 줄 것입니다.
        printf("This line will never, ever run.\n");
    }

    return 0;
}
```

### 일반적인 사용 사례

- **종료 래퍼 (Exit Wrappers):** `exit()`, `abort()` 또는 `quick_exit()`를 호출하는 함수들.
- **무한 루프:** 임베디드 시스템에서 `main` 함수나 "idle 태스크"는 종종 반환되지 않습니다.
- **롱 점프 (Long Jumps):** 정상적인 스택을 통하지 않고 프로그램 실행의 다른 지점으로 점프하기 위해 `longjmp()`를 사용하는 함수들.

### C23 - 대괄호 구문 (bracket syntax)

```c
#include <stdio.h>
#include <stdlib.h>

// C23 스타일: 속성이 함수 선언 앞에 배치됩니다.
// 컴파일러에게 "이 실행 경로는 여기서 끝남"이라고 명확하게 알립니다.
[[noreturn]] void emergency_shutdown(const char *reason) {
    printf("SHUTDOWN: %s\n", reason);
    
    // 커널이나 임베디드 시스템에서는 이것이 무한 루프이거나
    // 하드웨어 리셋 호출일 수 있습니다.
    exit(EXIT_FAILURE); 
}

int main(void) {
    printf("System heartbeat: OK\n");

    int critical_failure = 1;

    if (critical_failure) {
        emergency_shutdown("Power surge detected.");
        
        // 이것은 형식적으로 "도달 불가능한 코드"입니다.
        // 현대적인 컴파일러는 종종 컴파일된 바이너리에서 이를 완전히 생략합니다.
        printf("Trying to recover..."); 
    }

    return 0;
}
```

### 디스어셈블리 (Disassembly)

```c
11fc:	48 8d 05 45 0e 00 00 	lea    0xe45(%rip),%rax        # 2048 <_IO_stdin_used+0x48>
1203:	48 89 c7             	mov    %rax,%rdi
1206:	e8 65 fe ff ff       	call   1070 <puts@plt>
```

- 두 실행 파일을 디스어셈블해보면, 이 세 가지 명령어가 `noreturn` 버전에서 최적화되었음을 확인할 수 있습니다.

## _Generic

복잡한 `tgmath`와 같은 제네릭을 대체합니다.

```c
//_Generic ( 제어-표현식 , 연관-리스트 )

_Generic((variable),
    type1: outcome1,
    type2: outcome2,
    default: default_outcome
)
```

- 훨씬 더 깔끔합니다.

### 예제

```c
#include <stdio.h>
#include <math.h>
#include <stdlib.h>

// _Generic은 함수 이름(포인터)을 선택하고, 그 다음에 (X)로 호출합니다.
#define absolute_value(X) _Generic((X), \
    int: abs,                           \
    float: fabsf,                       \
    double: fabs,                       \
    long double: fabsl,                 \
    default: fabs                       \
)(X)

int main(void) {
    int i = -10;
    float f = -3.14f;
    double d = -9.81;

    // 컴파일러는 absolute_value(d)를 다음과 같이 확장합니다:
    // (fabs)(d)
    // 이는 완벽하게 유효하며 선택되지 않은 브랜치에서 타입 충돌을 일으키지 않습니다!

    printf("Integer: %d\n", absolute_value(i));
    printf("Float: %f\n", absolute_value(f));
    printf("Double: %f\n", absolute_value(d));

    return 0;
}
```

## 멀티스레딩 지원 (Multi-threading support)

### 핵심 구성 요소

- **`<threads.h>`:** 메인 헤더. `thread_local` 매크로(`_Thread_local`의 깔끔한 래퍼), 스레드 생성(`thrd_create`), 뮤텍스(`mtx_t`)를 제공합니다.
- **`thread_local`:** 저장 클래스 지정자. 전역 또는 정적 변수를 `thread_local`로 선언하면, **각 스레드는 해당 변수의 독립적인 복사본**을 갖게 됩니다.
- **`<stdatomic.h>`:** 락-프리(lock-free) 원자적 타입(예: `atomic_int`)을 제공합니다. 이를 통해 뮤텍스 잠금의 무거운 성능 비용 없이 여러 스레드가 동일한 변수를 안전하게 읽고 쓸 수 있습니다.

`Threads.h`는 자주 사용되지 않습니다. Linux의 Pthread와 WIN32 API가 여전히 C에서의 멀티스레딩을 지배하고 있습니다.

- `_Atomic`은 뮤텍스를 잡지 않고 원자적 연산을 수행하기 위해 TAS(Test-and-Set) 및 CAS(Compare-and-Swap)를 사용합니다.
- 큰 데이터에 `_Atomic`이 설정된 경우 내부적으로 뮤텍스를 폴백으로 사용할 수도 있습니다.

### 왜 `_Atomic`이 수동 뮤텍스보다 우수한가

- **최적화:** 컴파일러는 커널 수준의 뮤텍스보다 훨씬 빠른 CAS 루프를 자주 사용할 수 있습니다.
- **메모리 순서 (Memory Ordering):** C11 원자성(atomics)을 사용하면 **메모리 배리어 (Memory Barriers)**(예: `memory_order_acquire` 또는 `memory_order_release`)를 지정할 수 있습니다. 이는 시스템을 멈추지 않고 코어 전체에서 캐시를 동기화하는 방법을 CPU에 정확히 알려줍니다.
- **정확성:** `_Atomic` 변수의 잠금 해제를 "잊어버리는" 것이 불가능합니다. 동기화는 변수 자체에 묶여 있으며 그 주변의 코드 로직에 묶여 있지 않습니다.

### 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <threads.h>   // C11 Threads, Mutexes, and thread_local
#include <stdatomic.h> // C11 Atomics

#define NUM_THREADS 5
#define ITERATIONS 10000

// --- 전역 변수 ---

// 1. 뮤텍스로 보호되는 공유 변수
mtx_t shared_mutex;
int shared_counter = 0;

// 2. 원자적 락-프리 공유 변수
atomic_int atomic_counter = 0;

// 3. 스레드 로컬 변수 (각 스레드는 0으로 초기화된 자신만의 변수를 가짐)
thread_local int local_counter = 0; 

// --- 스레드 함수 ---
// 반드시 int를 반환하고 void* 인자를 받아야 함
int worker_thread(void *arg) {
    int thread_id = *(int*)arg;

    for (int i = 0; i < ITERATIONS; i++) {
        // --- 뮤텍스 보호 ---
        mtx_lock(&shared_mutex);
        shared_counter++; 
        mtx_unlock(&shared_mutex);

        // --- 원자적 보호 ---
        // atomic_counter++는 뮤텍스 잠금 없이 안전하게 증가함
        atomic_counter++; 

        // --- 스레드 로컬 ---
        // 잠금 없이 수정해도 완벽하게 안전함; 다른 스레드는 이를 볼 수 없음.
        local_counter++; 
    }

    printf("Thread %d finished. Its local_counter is: %d\n", thread_id, local_counter);
    return thrd_success;
}

int main(void) {
    thrd_t threads[NUM_THREADS];
    int thread_ids[NUM_THREADS];

    // 뮤텍스 초기화 (mtx_plain은 표준 비재귀 뮤텍스를 의미함)
    if (mtx_init(&shared_mutex, mtx_plain) != thrd_success) {
        fprintf(stderr, "Failed to initialize mutex.\n");
        return 1;
    }

    // 스레드 생성
    printf("Starting %d threads...\n\n", NUM_THREADS);
    for (int i = 0; i < NUM_THREADS; i++) {
        thread_ids[i] = i + 1;
        if (thrd_create(&threads[i], worker_thread, &thread_ids[i]) != thrd_success) {
            fprintf(stderr, "Failed to create thread %d.\n", i);
            return 1;
        }
    }

    // 모든 스레드가 끝날 때까지 대기 (Join)
    for (int i = 0; i < NUM_THREADS; i++) {
        thrd_join(threads[i], NULL);
    }

    // 정리
    mtx_destroy(&shared_mutex);

    // 최종 결과 출력
    printf("\n--- Final Results ---\n");
    printf("Expected total for shared counters: %d\n", NUM_THREADS * ITERATIONS);
    printf("Mutex shared_counter:  %d\n", shared_counter);
    printf("Atomic atomic_counter: %d\n", atomic_counter);

    return 0;
}
```

## 비정규 부동 소수점 수 (Subnormal floating-point numbers)

`double`의 최소값보다 작은 수를 표현하기 위해 필요합니다.

- 컴퓨터가 `1.17e-38`보다 작은 수를 표현해야 하는 경우, 그냥 `0`으로 언더플로우되어 물리 및 공학 시뮬레이션에서 수학적 오류를 일으킬 수 있습니다.
- 이를 해결하기 위해 비정규 수(Subnormal Numbers)를 사용합니다.
    - `1.xxxx * 2^exponent`를 유지하는 대신, `0.xxxx * 2^minimum exponent`를 사용합니다.
    - 이를 비정규 수 또는 비정규화 수(denormalized numbers)라고 부릅니다.
- 항상 존재해왔지만 C11 이전에는 C 프로그래머가 코드를 실행하는 컴퓨터가 계산을 수행할지 아니면 반올림하여 버릴지 알 수 있는 표준적인 방법이 없었습니다.
- C11은 단순히 `FLT_HAS_SUBNORM` 매크로를 추가하여 코드가 컴파일러에게 물어볼 수 있게 했습니다.

```c
#include <stdio.h>
#include <float.h>

int main(void) {
    printf("--- Subnormal Support ---\n");
    printf("Does float support subnormals?       %d\n", FLT_HAS_SUBNORM);
    printf("Does double support subnormals?      %d\n", DBL_HAS_SUBNORM);
    
    // 가장 작은 정규 float vs 가장 작은 비정규 float
    printf("\nSmallest normal float:   %e\n", FLT_MIN);
    printf("Smallest subnormal float:  %e\n", FLT_TRUE_MIN); // C11에서 추가됨!

    printf("\n--- Serialization Digits (Float) ---\n");
    printf("Trustworthy digits (FLT_DIG):        %d\n", FLT_DIG);
    printf("Round-trip digits (FLT_DECIMAL_DIG): %d\n", FLT_DECIMAL_DIG);

    // 라운드-트립(Round-Trip) 문제 시연
    float original = 0.1f;
    
    // FLT_DIG(6자리)까지만 출력하는 경우...
    printf("\nPrinted with %%..%df: %.*f\n", FLT_DIG, FLT_DIG, original);
    
    // 다른 파서가 0.1f의 정확한 바이너리 표현을 재구축하도록 보장하려면,
    // FLT_DECIMAL_DIG(9자리)를 사용하여 출력해야 합니다.
    printf("Printed with %%..%df: %.*f\n", FLT_DECIMAL_DIG, FLT_DECIMAL_DIG, original);

    return 0;
}
```

| **매크로 유형** | **예시** | **알려주는 정보** |
| --- | --- | --- |
| **신뢰할 수 있는 자릿수** | `FLT_DIG` (6) | *10진수 -> 부동소수점 -> 10진수*. 수학이 흐릿해지지 않고 파싱할 수 있는 자릿수입니다. |
| **라운드-트립 자릿수** | `FLT_DECIMAL_DIG` (9) | *부동소수점 -> 10진수 -> 부동소수점*. 부동소수점을 완벽하게 직렬화하기 위해 출력해야 하는 자릿수입니다. |
| **비정규 최소값** | `FLT_TRUE_MIN` | 비정규화된 상태를 포함하여 가능한 절대적인 최소 0이 아닌 수입니다 (C11). |

## 구조체 내부의 익명 공용체 및 구조체 (Anonymous Unions and Structs Inside Struct)

### C11 이전

```c
#include <stdio.h>

// C11 이전 스타일
struct LegacyVector {
    union {
        struct { float x, y, z; } coords; // 이름이 있는 구조체
        float array[3];
    } data; // 이름이 있는 공용체
};

int main(void) {
    struct LegacyVector v;
    
    // 이 끔찍하고 장황한 구문을 보세요:
    v.data.coords.x = 1.0f;
    v.data.coords.y = 2.0f;
    v.data.coords.z = 3.0f;

    printf("Legacy X: %f\n", v.data.coords.x);
    printf("Legacy Array[0]: %f\n", v.data.array[0]);

    return 0;
}
```

### C11

```c
#include <stdio.h>

// C11 스타일
struct ModernVector {
    union { // <-- 익명 공용체
        struct { float x, y, z; }; // <-- 익명 구조체
        float array[3];
    }; 
};

int main(void) {
    struct ModernVector v;
    
    // 아름답고 깔끔한 접근.
    // 우리는 정확히 동일한 메모리 위치에 쓰고 있습니다...
    v.x = 1.0f;
    v.y = 2.0f;
    v.z = 3.0f;

    // ...하지만 깊은 중첩 없이 배열로 직접 읽을 수 있습니다!
    for (int i = 0; i < 3; i++) {
        printf("Index %d: %f\n", i, v.array[i]);
    }

    return 0;
}
```

- 스레드 매크로와 마찬가지로, MSVC와 GCC는 C11 이전부터 비표준 컴파일러 확장으로 익명 구조체와 공용체를 지원해 왔습니다.

## Static_assert

```c
#include <stdio.h>
#include <assert.h> // static_assert 매크로 제공

// 포인터가 8바이트인 64비트 시스템에서 컴파일되고 있는지 보장합니다.
static_assert(sizeof(void *) == 8, "This code requires a 64-bit architecture!");

// 정수가 정확히 32비트(4바이트)임을 보장합니다.
static_assert(sizeof(int) == 4, "Integers must be exactly 32 bits.");

int main(void) {
    printf("Architecture checks passed! Compiling...\n");
    return 0;
}
```

## fopen의 독점적 생성 및 열기 모드 (Exclusive create-and-open mode for fopen)

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    const char *lock_file = "app.lock";

    // 독점 모드로 파일을 열려고 시도합니다.
    // "wx"는 "app.lock"이 이미 존재하면 안전하게 실패하도록 보장합니다.
    FILE *fp = fopen(lock_file, "wx");

    if (fp == NULL) {
        fprintf(stderr, "ERROR: Another instance is already running "
                        "(or lock file '%s' exists).\n", lock_file);
        return EXIT_FAILURE;
    }

    printf("Lock file acquired. Running application...\n");

    // 최선의 관행으로 잠금 파일에 프로세스 ID(PID)를 기록합니다.
    fprintf(fp, "LOCKED\n");
    
    // 실제 애플리케이션 작업을 여기서 수행합니다...
    
    // 정리
    fclose(fp);
    
    // 프로그램이 정상적으로 종료되면 잠금 파일을 제거합니다.
    if (remove(lock_file) == 0) {
        printf("Lock file removed safely.\n");
    } else {
        perror("Failed to remove lock file");
    }

    return EXIT_SUCCESS;
}
```

- POSIX의 `O_CREAT|O_EXCL`처럼 동작합니다.
- TOCTOU(Time-of-check to Time-of-use) 경쟁 조건을 해결합니다.
- 보안이 중요한 시스템에서 많이 사용됩니다.
- 여전히 권한 제어가 부족합니다.
    - Linux / macOS 사용자는 권한 비트와 함께 `O_EXCL`을 사용하는 `open()`을 사용합니다.

```c
// POSIX 방식: 원자적 생성 및 보안 권한 설정
int fd = open("lock.txt", O_CREAT | O_EXCL | O_WRONLY, S_IRUSR | S_IWUSR);
```

# 기타

- 개선된 유니코드 지원
    - 데이터 내 유니코드
- gets 함수 제거
- 경계 검사 인터페이스 (Annex K)
    - 널리 구현되지 않음
    - Microsoft는 C11 이전부터 `_s` 함수들을 가지고 있었음
        - 표준화 부족
- 분석 가능성 기능 (Annex L)
    - 아무도 채택하지 않음
- quick_exit
- timespec_get
- 복소수 값을 위한 매크로
