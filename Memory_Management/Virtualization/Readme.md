# 🧠 Memory Virtualization

> 부산대학교 정보컴퓨터공학부 - Operating System (CP26044)

---

## 📌 목차

- [Memory Virtualization 개요](#memory-virtualization-개요)
- [초기 시스템의 OS](#초기-시스템의-os)
- [멀티프로그래밍과 시분할](#멀티프로그래밍과-시분할)
- [Address Space](#address-space)
- [Virtual Address](#virtual-address)
- [Address Translation](#address-translation)
- [Static Relocation](#static-relocation)
- [Dynamic Relocation](#dynamic-relocation)
- [Base and Bounds Register](#base-and-bounds-register)
- [MMU: Memory Management Unit](#mmu-memory-management-unit)
- [OS Issues for Memory Virtualizing](#os-issues-for-memory-virtualizing)

---

## Memory Virtualization 개요

### Memory Virtualization이란?

- OS가 물리 메모리를 가상화한다.
- OS는 각 프로세스마다 **illusion 메모리 공간**을 제공한다.
- 각 프로세스가 마치 **전체 메모리를 혼자 사용하는 것처럼** 보이게 한다.

### Memory Virtualization의 목표

**Transparency (투명성)**
- 프로세스는 메모리가 공유되고 있다는 사실을 인식하지 못해야 한다.
- 프로그래밍을 위한 편리한 추상화를 제공한다.
- 즉, 크고 연속된 메모리 공간처럼 보이게 한다.

**Efficiency (효율성)**
- 가변 크기 요청으로 인한 단편화(fragmentation)를 최소화한다. (공간 측면)
- 하드웨어 지원을 활용한다. (시간 측면)

**Protection (보호)**
- 프로세스와 OS를 다른 프로세스로부터 보호한다.
- Isolation: 하나의 프로세스가 실패해도 다른 프로세스에 영향을 주지 않는다.
- 협력하는 프로세스들은 메모리 일부를 공유할 수 있다.

---

## 초기 시스템의 OS

- 메모리에 **하나의 프로세스**만 로드한다.
- **낮은 활용도와 효율성** 문제가 있다.

```
Physical Memory 구조:
┌──────────────────────┐ 0KB
│  Operating System    │
│  (code, data, etc.)  │
├──────────────────────┤ 64KB
│                      │
│   Current Program    │
│  (code, data, etc.)  │
│                      │
└──────────────────────┘ max
```

---

## 멀티프로그래밍과 시분할

### 멀티프로그래밍

- 메모리에 **여러 프로세스**를 로드한다.
- 각 프로세스를 짧은 시간씩 번갈아 실행한다.
- 메모리 내에서 프로세스 간 전환(Switch)이 발생한다.
- **활용도와 효율성이 향상**된다.

### 문제점

- 중요한 **Protection 이슈** 발생
  - 다른 프로세스의 메모리 영역에 잘못 접근(errant memory access)하는 문제

```
Physical Memory 구조:
┌──────────────────────┐ 0KB
│  Operating System    │
│  (code, data, etc.)  │
├──────────────────────┤ 64KB
│         Free         │
├──────────────────────┤ 128KB
│  Process C           │
│  (code, data, etc.)  │
├──────────────────────┤ 192KB
│  Process B           │
│  (code, data, etc.)  │
├──────────────────────┤ 256KB
│         Free         │
├──────────────────────┤ 320KB
│  Process A           │
│  (code, data, etc.)  │
├──────────────────────┤ 384KB
│         Free         │
├──────────────────────┤ 448KB
│         Free         │
└──────────────────────┘ 512KB
```

---

## Address Space

- OS는 물리 메모리의 **추상화**를 생성한다.
- Address Space는 실행 중인 프로세스에 관한 모든 것을 포함한다.
- 구성: **프로그램 코드, 힙, 스택** 등

### 구성 요소

**Code**
- 명령어(instructions)가 존재하는 공간

**Heap**
- 동적으로 메모리를 할당하는 공간
  - C 언어의 `malloc`
  - 객체지향 언어의 `new`

**Stack**
- 반환 주소(return address)나 값을 저장
- 지역 변수(local variables), 루틴에 대한 인자(arguments)를 포함

```
Address Space 구조:
┌──────────────────────┐ 0KB
│     Program Code     │
├──────────────────────┤ 1KB
│         Heap         │
├──────────────────────┤ 2KB
│          ↓           │
│        (free)        │
│          ↑           │
├──────────────────────┤ 15KB
│        Stack         │
└──────────────────────┘ 16KB
```

> Address Translation Mechanism을 통해 Address Space의 가상 주소가 Physical Memory의 물리 주소로 매핑된다.

---

## Virtual Address

- 실행 중인 프로그램의 **모든 주소는 가상(virtual)** 이다.
- OS가 가상 주소를 물리 주소로 변환(translate)한다.

### 예제: 64-bit Linux에서 주소 출력

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    printf("location of code  : %p\n", (void *) main);
    printf("location of heap  : %p\n", (void *) malloc(1));
    int x = 3;
    printf("location of stack : %p\n", (void *) &x);
    return x;
}
```

**출력 결과:**

```
location of code  : 0x40057d
location of heap  : 0xcf2010
location of stack : 0x7fff9ca45fcc
```

### Address Space 구조 (64-bit Linux 기준)

| 영역 | 가상 주소 범위 |
|------|--------------|
| Code (Text) | `0x400000` ~ `0x401000` |
| Data | `0x401000` ~ `0xcf2000` |
| Heap | `0xcf2000` ~ `0xd13000` |
| (free) | - |
| Stack | `0x7fff9ca28000` ~ `0x7fff9ca49000` |

> 프로그램에서 보이는 주소는 모두 **가상 주소**이며, 실제 물리 메모리 주소와 다르다.

---

## Address Translation

- **하드웨어**가 가상 주소를 물리 주소로 변환한다.
- 실제 정보는 물리 주소에 저장되어 있다.

### OS의 역할

- 하드웨어를 설정하기 위해 핵심 시점마다 개입해야 한다.
- 어느 위치가 사용 중이고 어느 위치가 비어있는지 추적(관리)해야 한다.

### 가정 (Assumption)

- 사용자의 address space는 물리 메모리에 **연속적으로(contiguously)** 배치되어야 한다.
- Address space의 크기는 물리 메모리 크기보다 **작아야** 한다.
- 각 address space는 **동일한 크기**를 가진다.

### 예제: Address Translation

**C 코드:**
```c
void func() {
    int x;
    ...
    x = x + 3; // 관심 있는 코드 라인
}
```

**어셈블리 코드:**
```asm
128 : movl 0x0(%ebx), %eax   ; load 0+ebx into eax
132 : addl $0x03,    %eax   ; add 3 to eax register
135 : movl %eax,  0x0(%ebx) ; store eax back to mem
```

**실행 순서:**
1. 주소 128에서 명령어 Fetch
2. 해당 명령어 실행 → 주소 15KB에서 load
3. 주소 132에서 명령어 Fetch
4. 해당 명령어 실행 (메모리 참조 없음)
5. 주소 135에서 명령어 Fetch
6. 해당 명령어 실행 → 주소 15KB에 store

> **Relocation Address Space**: OS는 프로세스를 주소 0이 아닌 물리 메모리의 다른 위치에 배치하려 한다.

---

## Static Relocation

### 소프트웨어 기반 재배치

- OS가 프로그램을 메모리에 로드하기 **전에** 직접 재작성(rewrite)한다.
- 정적 데이터와 함수들의 주소를 변경한다.

```
원본 (00000):            재배치 후 (10000):        재배치 후 (50000):
00128: movl 0x0(%ebx)   10128: movl 0x0(%ebx)   50128: movl 0x0(%ebx)
00132: addl 0x03,%eax   10132: addl 0x03,%eax   50132: addl 0x03,%eax
00135: movl %eax,0x0    10135: movl %eax,0x0    50135: movl %eax,0x0
Program Code            Program Code             Program Code
Stack (15360)           Stack (25360)            Stack (65360)
```

### 장점 (Pros)

- 하드웨어 지원이 필요 없다.

### 단점 (Cons)

- **보호(protection)가 적용되지 않는다.**
  - 프로세스가 OS나 다른 프로세스의 메모리 영역을 파괴할 수 있다.
  - 개인정보 보호 없음: 어떤 메모리 주소든 읽을 수 있다.
- **배치 후 address space를 이동할 수 없다.**
  - 외부 단편화(external fragmentation)로 인해 새 프로세스를 할당하지 못할 수 있다.

---

## Dynamic Relocation

### 하드웨어 기반 재배치

- **MMU (Memory Management Unit)** 가 모든 메모리 참조 명령어에서 주소 변환을 수행한다.
- 보호(protection)가 하드웨어에 의해 적용된다.
  - 가상 주소가 유효하지 않으면 MMU가 예외(exception)를 발생시킨다.
- OS가 현재 프로세스의 **유효한 address space 정보**를 MMU에 전달한다.

---

## Base and Bounds Register

### 개요

```
Address Space            Physical Memory
┌──────────────┐ 0KB    ┌──────────────┐ 0KB
│ Program Code │        │  Operating   │
│              │        │   System     │
├──────────────┤ 1KB    ├──────────────┤ 16KB
│     Heap     │        │  (not in use)│
│              │        ├──────────────┤ 32KB  ← base register: 32KB
│     heap↓    │        │     Code     │
│    (free)    │        │     Heap     │
│     stack↑   │        │  (allocated  │
│              │        │  not in use) │
├──────────────┤ 15KB   │     Stack    │
│    Stack     │        ├──────────────┤ 48KB
└──────────────┘ 16KB   │  (not in use)│
                        └──────────────┘ 64KB
bounds register: 16KB
```

### OS의 중요한 역할

OS와 사용자 프로세스 모두를 **보호**하는 것.

| 레지스터 | 역할 |
|---------|------|
| **Base register** | 가장 작은 합법적(legal) 물리 메모리 주소 |
| **Bound register** | 범위의 크기 (size of the range) |

### Hardware Address Protection

```
CPU → address
         ↓
    [address >= base?]
         ↓ yes
    [address < base+bound?]
         ↓ yes
         → memory (접근 허용)
         
         ↓ no (어느 쪽이든)
         → trap to operating system monitor (addressing error)
```

> **각 프로세스는 별도의 메모리 공간을 가지는 것이 보장된다.**

### Bounds Register의 두 가지 표현 방식

| 방식 | 값 | 의미 |
|------|-----|------|
| **address space의 크기** | 16KB | 가상 주소 공간의 총 크기 |
| **address space 끝의 물리 주소** | 48KB | 물리 메모리에서 끝나는 지점 |

---

## MMU: Memory Management Unit

- 가상 주소를 물리 주소로 매핑하는 **하드웨어 장치**
- 사용자 프로세스가 생성하는 모든 주소에 **base register 값을 더해** 물리 메모리로 전송
- 사용자 프로그램은 **논리 주소(logical address)** 만 다루며, 실제 물리 주소는 절대 보지 못함

```
                Base register
                  [ 14000 ]
                      |
CPU → logical         ↓        physical
      address  →    [  +  ]  →  address  → memory
        346                       14346
                    MMU
```

### Dynamic(Hardware base) Relocation 공식

```
physical address = virtual address + base

0 ≤ virtual address < bounds
```

### 실제 예시 (base = 32KB)

```asm
128 : movl 0x0(%ebx), %eax
```

| 동작 | 계산 |
|------|------|
| 주소 128에서 명령어 Fetch | `32896 = 128 + 32KB(base)` |
| 주소 15KB에서 load 실행 | `47KB = 15KB + 32KB(base)` |

---

## OS Issues for Memory Virtualizing

OS는 **base-and-bounds** 방식을 구현하기 위해 반드시 조치를 취해야 한다.

### 세 가지 핵심 시점 (Three Critical Junctures)

| 시점 | OS의 역할 |
|------|----------|
| **프로세스 시작 시** | 물리 메모리에서 address space를 위한 공간 찾기 |
| **프로세스 종료 시** | 사용한 메모리를 반환(회수)하기 |
| **컨텍스트 스위치 시** | base-and-bounds 쌍을 저장 및 복원하기 |

---

### 1. 프로세스 시작 시 (When a Process Starts Running)

- OS는 새 address space를 위한 공간을 찾아야 한다.
- **free list**: 현재 사용 중이지 않은 물리 메모리 범위 목록

```
Physical Memory:                    Free list:
┌──────────────┐ 0KB               ┌──────┐
│  Operating   │                   │ 16KB │
│   System     │                   └──┬───┘
├──────────────┤ 16KB                 ↓
│  (not in use)│  ← Free list에서   ┌──────┐
├──────────────┤ 32KB  탐색 후 할당  │ 48KB │
│   Code       │                   └──────┘
│   Heap       │
│ (alloc/n.u.) │
│   Stack      │
├──────────────┤ 48KB
│  (not in use)│
└──────────────┘ 64KB
```

---

### 2. 프로세스 종료 시 (When a Process Is Terminated)

- OS는 사용한 메모리를 **free list에 반환**해야 한다.

```
[종료 전]                           [종료 후]
Free list: 16KB → 48KB            Free list: 16KB → 32KB → 48KB

┌──────────────┐ 0KB              ┌──────────────┐ 0KB
│  OS          │                  │  OS          │
├──────────────┤ 16KB             ├──────────────┤ 16KB
│  (not in use)│                  │  (not in use)│
├──────────────┤ 32KB             ├──────────────┤ 32KB
│  Process A   │                  │  (not in use)│  ← 반환됨
├──────────────┤ 48KB             ├──────────────┤ 48KB
│  (not in use)│                  │  (not in use)│
└──────────────┘ 64KB             └──────────────┘ 64KB
```

---

### 3. 컨텍스트 스위치 시 (When Context Switch Occurs)

- OS는 base-and-bounds 쌍을 **저장하고 복원**해야 한다.
- **process structure** 또는 **PCB(Process Control Block)** 에 저장

```
[Context Switching 전]             [Context Switching 후]
base: 32KB, bounds: 48KB          base: 48KB, bounds: 64KB

┌──────────────┐ 0KB              ┌──────────────┐ 0KB
│  OS          │                  │  OS          │
├──────────────┤ 16KB             ├──────────────┤ 16KB
│  (not in use)│                  │  (not in use)│
├──────────────┤ 32KB             ├──────────────┤ 32KB
│  Process A   │ ← 실행 중        │  Process A   │
├──────────────┤ 48KB             ├──────────────┤ 48KB
│  Process B   │                  │  Process B   │ ← 실행 중
└──────────────┘ 64KB             └──────────────┘ 64KB

                    Process A PCB:
                    ┌──────────────────┐
                    │ ...              │
                    │ base   : 32KB    │
                    │ bounds : 48KB    │
                    │ ...              │
                    └──────────────────┘
```

> 컨텍스트 스위치 발생 시 현재 프로세스(A)의 base/bounds를 PCB에 저장하고, 다음 프로세스(B)의 base/bounds를 레지스터에 복원한다.
