# 🧠 Memory Management Strategy

---

## 📚 목차

1. [Continuous Memory Allocation (연속 메모리 할당)](#1-continuous-memory-allocation)
2. [Segmentation (세그멘테이션)](#2-segmentation)
3. [Paging (페이징)](#3-paging)

---

## 1. Continuous Memory Allocation

### 1.1 기본 개념 — Relocation Registers

**재배치 레지스터(Relocation Registers)** 는 사용자 프로세스들을 서로, 그리고 OS 코드·데이터로부터 보호하기 위해 사용된다.

| 레지스터 | 역할 |
|---|---|
| **Base Register** | 가장 작은 물리 주소 값 보관 |
| **Bound Register** | 논리 주소의 범위 보관 (각 논리 주소는 limit 이하여야 함) |

- MMU가 논리 주소를 **동적으로** 물리 주소로 변환
- 커널 코드의 transient 처리 및 커널 크기 변경 등을 허용

```
CPU → [논리 주소] → < Bound 비교 > --YES--> (+Base) --> 물리 주소 --> Memory
                            |
                           NO
                            ↓
                    trap: addressing error
```

---

### 1.2 Multiple-Partition Allocation (다중 파티션 할당)

- 멀티프로그래밍의 degree는 **파티션 수**에 의해 제한
- **Variable-partition**: 각 프로세스의 필요에 맞게 가변 크기 파티션 사용
- **Hole**: 사용 가능한 메모리 블록. 다양한 크기의 홀이 메모리 전반에 산재

> 프로세스가 도착하면 → 충분히 큰 hole에서 메모리 할당

OS는 아래 정보를 유지 관리한다:
- 할당된 파티션 (Allocated partitions)
- 비어 있는 파티션 (Free partitions)

**할당 흐름 예시:**

```
[OS][Process 8][Process 5][Process 2]
        ↓ (Process 5 종료)
[OS][Process 8][  hole  ][Process 2]
        ↓ (Process 9 도착)
[OS][Process 8][Process 9][hole][Process 2]
        ↓
[OS][Process 8][Process 9][Process 2][Process 2]
```

---

### 1.3 Free Hole 할당 알고리즘

| 알고리즘 | 설명 | 특징 |
|---|---|---|
| **First-fit** | 충분히 큰 첫 번째 홀에 할당 | 빠름 |
| **Best-fit** | 충분히 큰 홀 중 가장 작은 것에 할당 | 가장 작은 잔여 홀 생성, 전체 탐색 필요 |
| **Worst-fit** | 가장 큰 홀에 할당 | 가장 큰 잔여 홀 생성, 전체 탐색 필요 |

---

### 1.4 Fragmentation (단편화)

#### External Fragmentation (외부 단편화)
- 요청을 만족할 **총 메모리는 존재**하지만, **연속적이지 않아** 할당 불가

#### Internal Fragmentation (내부 단편화)
- 할당된 메모리가 요청보다 **약간 더 큰** 경우 발생
- 파티션 내부의 미사용 메모리

#### 해결 방안

| 방법 | 설명 |
|---|---|
| **Compaction** | 메모리 내용을 재배치해 모든 빈 공간을 하나의 큰 블록으로 합침 (동적 재배치 필요) |
| **Paging / Segmentation** | 프로세스의 논리 주소를 비연속적으로 허용 → 물리 메모리 어디든 할당 가능 |

---

### 1.5 Base and Bound 방식의 비효율성

```
0KB  ┌──────────────┐
     │ Program Code │
2KB  ├──────────────┤
     │    Heap      │
4KB  ├──────────────┤
     │              │
     │   (free)     │  ← "free" 공간도 물리 메모리를 점유
     │              │
14KB ├──────────────┤
     │    Stack     │
16KB └──────────────┘
```

**문제점:**
- "free" 공간이 물리 메모리를 점유
- 주소 공간이 물리 메모리보다 크면 실행 불가

---

## 2. Segmentation

### 2.1 개념

**주소 공간을 논리적 세그먼트로 분할**

- 세그먼트 = 특정 길이의 주소 공간의 연속적인 부분
- 각 세그먼트는 주소 공간 내 **논리적 개체(logical entity)** 에 대응
- 대표적 세그먼트: **Code, Stack, Heap**

**각 세그먼트는 독립적으로:**
- 물리 메모리의 다른 위치에 배치 가능
- 성장(grow) 또는 축소(shrink) 가능
- **세그먼트마다 별도의 Base & Bounds** 존재

---

### 2.2 물리 메모리에 세그먼트 배치

**세그먼트 테이블 예시:**

| Segment | Base | Size |
|---------|------|------|
| Code    | 32K  | 2K   |
| Heap    | 34K  | 2K   |
| Stack   | 28K  | 2K   |

**주소 변환 공식:**

```
physical address = offset + base
offset = virtual address − start address of segment
```

---

### 2.3 주소 변환 예시

#### 예시 1 — Code 세그먼트 (virtual address: 100)

- Code 세그먼트 시작 = 가상 주소 0
- offset = 100 − 0 = **100**
- physical address = 100 + 32K = **32868**

#### 예시 2 — Heap 세그먼트 (virtual address: 4200)

- Heap 세그먼트 시작 = 가상 주소 4096
- offset = 4200 − 4096 = **104**
- physical address = 104 + 34K = **34920**

> ⚠️ `Virtual address + base`는 올바른 물리 주소가 아님! 반드시 **offset**을 사용해야 함.

---

### 2.4 Segmentation Fault

- 7KB처럼 Heap 끝을 초과하는 **불법 주소** 참조 시 발생
- 하드웨어가 **out of bounds** 감지 → OS가 segmentation fault 발생

---

### 2.5 어느 세그먼트를 참조하는가? (Explicit Approach)

가상 주소의 **상위 몇 비트**로 세그먼트를 구분:

```
 13  12  11  10   9   8   7   6   5   4   3   2   1   0
[  Segment  |              Offset                      ]
```

| Segment | bits |
|---------|------|
| Code    | 00   |
| Heap    | 01   |
| -       | 10   |
| Stack   | 11   |

**예: virtual address 4200 = `01 000001101000`**
- 상위 2비트 = `01` → **Heap 세그먼트**
- offset = `000001101000` = 104

**의사 코드:**

```c
// get top 2 bits of 14-bit VA
Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT   // SEG_MASK=0x3000, SEG_SHIFT=12

// get offset
Offset = VirtualAddress & OFFSET_MASK               // OFFSET_MASK=0xFFF

if (Offset >= Bounds[Segment])
    RaiseException(PROTECTION_FAULT)
else
    PhysAddr = Base[Segment] + Offset
    Register = AccessMemory(PhysAddr)
```

---

### 2.6 Stack 세그먼트 — 역방향 성장

스택은 **반대 방향(역방향)** 으로 성장하므로 추가 하드웨어 지원 필요

| Segment | Base | Size | Grows Positive? |
|---------|------|------|-----------------|
| Code    | 32K  | 2K   | 1 (양방향)       |
| Heap    | 34K  | 2K   | 1               |
| Stack   | 28K  | 2K   | 0 (역방향)       |

**예: virtual address 15KB = `11 1100 0000 0000`**

```
Segment bits = 11 → Stack
Offset = 3KB
Max segment size = 4KB (12비트 offset)
Correct negative offset = 3KB − 4KB = −1KB
Physical address = 28KB − 1KB = 27KB
```

---

### 2.7 Protection (보호 비트)

세그먼트 간 **공유** 및 **보호**를 위해 추가 하드웨어 지원 필요

| Segment | Base | Size | Grows Positive? | Protection   |
|---------|------|------|-----------------|--------------|
| Code    | 32K  | 2K   | 1               | Read-Execute |
| Heap    | 34K  | 2K   | 1               | Read-Write   |
| Stack   | 28K  | 2K   | 0               | Read-Write   |

- 코드 공유(Code sharing)는 현재 시스템에서도 사용됨
- 세그먼트당 read / write / execute 권한 비트 존재

---

### 2.8 Fine-Grained vs Coarse-Grained

| 구분 | 설명 |
|---|---|
| **Coarse-Grained** | 적은 수의 세그먼트 (예: code, heap, stack) |
| **Fine-Grained** | 많은 수의 세그먼트 → 유연성 높음, 하드웨어 **segment table** 필요 |

---

### 2.9 OS 지원 — External Fragmentation & Compaction

**문제:** 24KB가 비어 있지만 연속되지 않아 20KB 요청 불가

**해결책 — Compaction:**

```
[Not compacted]                    [Compacted]
0KB  ┌──────────────┐        0KB  ┌──────────────┐
     │      OS      │             │      OS      │
8KB  ├──────────────┤        8KB  ├──────────────┤
     │  (not in use)│             │              │
16KB ├──────────────┤             │  Allocated   │
     │  Allocated   │             │              │
24KB ├──────────────┤        24KB ├──────────────┤
     │  (not in use)│             │  Allocated   │
32KB ├──────────────┤        32KB ├──────────────┤
     │  Allocated   │             │              │
40KB ├──────────────┤             │  (not in use)│
     │  (not in use)│             │              │
48KB ├──────────────┤        48KB ├──────────────┤
     │  Allocated   │             │  Allocated   │
64KB └──────────────┘        64KB └──────────────┘
```

**Compaction 비용:**
1. 실행 중인 프로세스 정지 (Stop)
2. 데이터 복사 (Copy)
3. 세그먼트 레지스터 값 변경 (Change)

---

### 2.10 Segmentation 장단점

#### ✅ 장점 (Pros)
- 주소 공간의 sparse 할당 가능 (Stack & Heap 독립적 성장)
- 내부 단편화 없음
- 빠르고 하드웨어 친화적
- 세그먼트 공유 용이 (shared libraries 등)
- 각 세그먼트의 동적 재배치 지원

#### ❌ 단점 (Cons)
- 각 세그먼트는 **연속 할당** 필요 → **외부 단편화** 발생
- 큰 세그먼트를 위한 충분한 물리 메모리가 없을 수 있음
- 유연성 부족: 크지만 sparse하게 사용되는 heap도 전체가 메모리에 상주해야 함

---

## 3. Paging

### 3.1 개념

**주소 공간을 고정 크기 단위(Page)로 분할**

| 구분 | 설명 |
|---|---|
| **Page** | 가상 메모리를 동일한 크기의 블록으로 분할 |
| **Frame** | 물리 메모리를 동일한 크기의 블록으로 분할 |
| **Page Table** | VPN → PFN 매핑. 프로세스별로 존재 |

- Page/Frame 크기는 **2의 거듭제곱** (일반적으로 512B ~ 8KB)
- **외부 단편화 없음**

---

### 3.2 Paging 동작 개요

```
Process A                              Physical Memory
[page 3] ─┐                          ┌─ frame 11
[page 2] ──┼─ Page Table A ──────────┼─ frame 9
[page 1] ──┤  [4, 11, 7, 9]          ├─ frame 7
[page 0] ─┘                          └─ frame 4

Process B                             
[page 3] ─┐                          ┌─ frame 6
[page 2] ──┼─ Page Table B ──────────┼─ frame 5
[page 1] ──┤  [2, 10, 5, 6]          ├─ frame 10
[page 0] ─┘                          └─ frame 2
```

> 모든 페이지는 물리 메모리에 존재한다고 가정

---

### 3.3 간단한 페이징 예시

- **물리 메모리:** 128 bytes, 16 bytes 단위 frame
- **가상 주소 공간:** 64 bytes, 16 bytes 단위 page

```
가상 주소 공간          물리 메모리
0  ┌──────────┐    0  ┌──────────────┐ ← frame 0 (OS 전용)
   │  page 0  │   16  ├──────────────┤ ← frame 1 (unused)
16 ├──────────┤   32  ├──────────────┤ ← frame 2 (page 3 of AS)
   │  page 1  │   48  ├──────────────┤ ← frame 3 (page 0 of AS)
32 ├──────────┤   64  ├──────────────┤ ← frame 4 (unused)
   │  page 2  │   80  ├──────────────┤ ← frame 5 (page 2 of AS)
48 ├──────────┤   96  ├──────────────┤ ← frame 6 (unused)
   │  page 3  │  112  ├──────────────┤ ← frame 7 (page 1 of AS)
64 └──────────┘  128  └──────────────┘
```

---

### 3.4 주소 변환 (Address Translation)

**가상 주소의 두 구성요소:**

```
[ Va5 | Va4 | Va3 | Va2 | Va1 | Va0 ]
[   VPN    |        offset          ]
```

| 요소 | 설명 |
|---|---|
| **VPN** (Virtual Page Number) | 페이지 테이블의 인덱스 |
| **Offset** | 페이지 내 위치 |
| **PFN** (Page Frame Number) | 페이지 테이블이 결정 |

**물리 주소 = `<PFN, Offset>`**

**페이지 테이블:**
- OS가 관리
- VPN → PFN 매핑
- 가상 주소 공간의 페이지마다 하나의 PTE(Page Table Entry) 존재

---

### 3.5 주소 변환 예시

**virtual address 21 (64-byte 주소 공간, 16-byte 페이지)**

```
가상 주소:   [ 0 | 1 | 0 | 1 | 0 | 1 ]
              [VPN=01] [offset=0101]
                  ↓ (페이지 테이블 참조 → PFN=111)
물리 주소:   [ 1 | 1 | 1 | 0 | 1 | 0 | 1 ]
              [PFN=111] [offset=0101]
```

**32-bit 가상 주소 공간 예시 (4KB 페이지, 20-bit 물리):**

```
Virtual Address (32bits): 0x00004AFE
  └─ VPN (상위 20비트) → Page Table 참조 → PFN
  └─ Offset (하위 12비트) → 그대로 사용

Physical Address (20bits): 0x46AFE
```

| 항목 | 값 |
|---|---|
| Virtual address | 32 bits |
| Physical address | 20 bits |
| Page size | 4KB |
| Offset | 12 bits |
| VPN | 20 bits |
| Page table entries | 2²⁰ |

---

### 3.6 Internal Fragmentation 계산

| 항목 | 값 |
|---|---|
| Page size | 2,048 bytes |
| Process size | 72,766 bytes |
| 필요한 페이지 | 35 pages + 1,086 bytes |
| **내부 단편화** | 2,048 − 1,086 = **962 bytes** |
| Worst case | 1 frame − 1 byte |
| 평균 | 1/2 frame size |

> ⚠️ 작은 페이지 크기 → 단편화 감소, 하지만 **페이지 테이블 크기 증가**

---

### 3.7 페이지 테이블은 어디에 저장?

페이지 테이블은 메모리 내에 저장 (너무 크기 때문에 레지스터에 불가)

**32-bit 주소 공간, 4KB 페이지의 경우:**

```
4MB = 2²⁰ entries × 4 Bytes/entry
```

- 각 프로세스마다 별도 페이지 테이블 존재
- 물리 메모리에 저장

---

### 3.8 페이지 테이블의 내용 (PTE Flags)

| 플래그 | 설명 |
|---|---|
| **Valid Bit** | 해당 변환이 유효한지 여부 (가상 주소 사용 시마다 확인) |
| **Protection Bit** | 읽기/쓰기/실행 가능 여부 |
| **Present Bit** | 페이지가 물리 메모리에 있는지, 디스크에 있는지 |
| **Dirty Bit** | 페이지가 메모리에 올라온 후 수정되었는지 |
| **Reference Bit** | 페이지가 접근된 적 있는지 |

**x86 PTE 구조:**

```
 31                    12  11  10   9   8   7   6   5   4   3   2   1   0
[        PFN            ][ G][PAT][ D][ A][PCD][PWT][U/S][R/W][ P]
```

| 비트 | 설명 |
|---|---|
| P | Present (존재 여부) |
| R/W | Read/Write |
| U/S | User/Supervisor |
| A | Accessed |
| D | Dirty |
| PFN | Page Frame Number |

---

### 3.9 메모리 접근 의사 코드

```c
// 1. 가상 주소에서 VPN 추출
VPN = (VirtualAddress & VPN_MASK) >> SHIFT

// 2. PTE 주소 계산
PTEAddr = PTBR + (VPN * sizeof(PTE))

// 3. PTE 가져오기
PTE = AccessMemory(PTEAddr)

// 4. 접근 유효성 확인
if (PTE.Valid == False)
    RaiseException(SEGMENTATION_FAULT)
else if (CanAccess(PTE.ProtectBits) == False)
    RaiseException(PROTECTION_FAULT)
else
    // 5. 물리 주소 계산 및 메모리 접근
    offset = VirtualAddress & OFFSET_MASK
    PhysAddr = (PTE.PFN << PFN_SHIFT) | offset
    Register = AccessMemory(PhysAddr)
```

---

### 3.10 페이지 테이블 구현

| 레지스터 | 역할 |
|---|---|
| **PTBR** (Page Table Base Register) | 페이지 테이블의 시작 주소 가리킴 |
| **PTLR** (Page Table Length Register) | 페이지 테이블의 크기 나타냄 |

> ⚠️ **2번의 메모리 접근 문제:** 매 접근마다 ① 페이지 테이블 접근, ② 실제 데이터 접근

**해결책: TLB (Translation Look-aside Buffers)**
- 특수한 고속 하드웨어 캐시
- 자주 사용하는 VPN→PFN 변환을 캐시에 보관

---

### 3.11 Demand Paging

OS는 메인 메모리를 **페이지 캐시**로 활용

- 페이지를 필요할 때만 메모리에 로드
- 페이지는 물리 메모리 프레임에서 **evict** 가능
- Evict된 페이지는 디스크로 (dirty page만 기록)
- 페이지 이동은 프로세스에게 **투명(transparent)**

**장점:**
- I/O 감소
- 메모리 사용량 감소
- 빠른 응답
- 더 많은 프로세스 수용 가능

---

### 3.12 Page Fault

CPU가 유효하지 않은 PTE 접근 시 발생하는 예외

| 종류 | 설명 |
|---|---|
| **Major Page Fault** | 페이지가 유효하지만 메모리에 없음 → 디스크 I/O 필요 |
| **Minor Page Fault** | 디스크 I/O 없이 해결 가능 (lazy allocation, prefetch 등) |
| **Invalid Page Fault** | 세그먼테이션 위반 — 페이지가 사용 중이 아님 |

**Page Fault 처리 흐름:**

```
① 프로세스가 메모리 참조 (load M)
② PTE 확인 → invalid → trap 발생
③ OS: backing store(disk)에서 페이지 위치 확인
④ 빈 프레임(free frame)에 누락 페이지 로드
⑤ 페이지 테이블 업데이트 (reset page table)
⑥ 명령 재시작 (restart instruction)
```

---

### 3.13 Paging 장단점

#### ✅ 장점 (Pros)
- **외부 단편화 없음**
- 빠른 할당/해제 (연속 공간 탐색 불필요, 인접 공간 병합 불필요)
- 메모리 일부를 디스크로 page out 용이 (valid bit 활용)
- 일부 페이지가 디스크에 있어도 프로세스 실행 가능
- 페이지 보호 및 공유 용이

#### ❌ 단점 (Cons)
- **내부 단편화** 발생 (페이지 크기가 클수록 낭비 증가)
- **메모리 참조 오버헤드** (명령당 메모리 참조 2배) → TLB로 해결
- **페이지 테이블 저장 공간** 문제
  - 32-bit 주소 공간, 4KB page: 2²⁰ PTE
  - 4 bytes/PTE → 프로세스당 4MB
  - 100개 프로세스 → 총 400MB의 페이지 테이블
  - 해결: valid PTE만 저장 또는 페이지 테이블을 페이징

---

## 📊 요약 비교

| 구분 | Continuous | Segmentation | Paging |
|---|---|---|---|
| 단위 | 가변 | 가변 (논리 세그먼트) | 고정 (page/frame) |
| 외부 단편화 | ✅ 발생 | ✅ 발생 | ❌ 없음 |
| 내부 단편화 | ✅ 발생 | ❌ 없음 | ✅ 발생 |
| 주소 변환 | Base + Offset | Base + Offset (세그먼트별) | 페이지 테이블 |
| 공유 지원 | 어려움 | 가능 | 가능 |
| 디스크 Swap | 어려움 | 가능 | ✅ 용이 |

---

> 📌 **핵심 키워드:** `Base & Bound`, `Hole`, `First/Best/Worst-fit`, `External/Internal Fragmentation`, `Compaction`, `Segmentation`, `Paging`, `VPN`, `PFN`, `PTE`, `PTBR`, `TLB`, `Demand Paging`, `Page Fault`
